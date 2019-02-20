---
vue-note-completed: true
---
# core/instance/proxy.js
## initProxy
### 作用
开发环境下对`render`函数要访问的数据进行代理，做错误提示。
### 过程
1. 首先判断环境是否实现了原生的`Proxy`。如果有，进行下一步。否则令`vm.renderProxy = vm`，和生产环境下一样。
2. 接着判断`vm`的选项中有`render`函数并且`render._withStripped`为真，先不管这个属性是干嘛的，来进行`get`或者`set`的代理。`getHandler`详见下文。
> 回想在[init](./init.md)中，进行这一步的时候，还没有进行`initRender`，也就是说选项中如果有`render`，要么是在实例化时传递进来的，要么是构造函数本身就有，总之，它的`render`选项已经提前定义好了。先考虑没有的情况，即只传递了`template`选项的情况。注：这不是引用，只是为了格式好看。
## hasHandler
### 作用
只对实例渲染时的`has`操作进行代理，以达到过滤不允许使用的变量的目的，具体使用在[这里]()，即在`template`中使用的表达式的一些变量，如：
```html
<div :title="window.title">
</div>
```
中的`window`是不被允许的。具体哪些不能使用，请看下文。
### 过程
1. 定义`has`，值为`key in vm`，这个变量在`vm`中有定义，即在`data`等属性中有这个属性。
2. 定义`isAllowed`，它的值是`allowedGlobals(key)`，这个函数是判断是否是允许的全局变量，上文的`window`就不是。或者这个变量名以`_`开头（**没太懂为什么要使用`key.charAt(0)`而不是直接访问**）且它不在`vm.$data`中有定义，即它是定义的私有属性，如：
```js
export default {
  _name: 'private_name'
}
```
> 但实际上以`_`开头的这个规则，是写给`babel_helper`使用的。因为有些方法会被编译成·`_*`。
3. 如果`has`和`isAllowed`都为`false`，即渲染函数内无法访问这个变量，根据`key in vm.$data`的值给出不同的提示。
   - 如果为`true`，提示这个值只能通过`$data.${key}`来使用，因为以`_`或者`$`开头的属性不会被代理，也就是不是响应式的。
   - 如果为`false`，提示这个属性没有定义。
4. 接着返回`has || !isAllowed`，我本以为会返回`has || isAllowed`，判断这个变量是否可以用嘛（这个时候还没有看在这个拦截操作是在哪里被调用的，我猜看了之后会好理解很多），思考之后才明白，如上文的`window`，`has`为`false`，`isAllowed`也为`false`，就是因为不允许使用，所以需要渲染成`this.window`，这个拦截应该返回`true`，也就是`!isAllowed`。
---
> 接`initProxy`，那么如果传递了`render`选项并且`render._withStriped`为真呢？`withStriped`，中文意思应该是分割？搜索之后发现只有测试代码中出现了这个属性。同时在git记录中看到对`getHandler`有注解说是为了对手写的`render`函数也能进行属性检查。那么非手写的`render`函数应该是经过了特殊处理得到的，会触发`has`操作（具体看下文）。手写的则不会，但是同时也需要属性检查，所以需要这个然后对`get`操作进行拦截。<br>
> 但同时我们知道（如果不知道，现在知道了）除了手写`render: h => h()`外，`render`函数是编译得到的，这个编译分为两种情况，一种是在浏览器里运行时编译，另外就是在使用构建工具时提前编译。显然运行时编译在这一步是没有`render`函数的，只有使用构建工具时才会出现这种情况，`render && render._withStriped`。<br>
> 这么一想，也就是说，运行时编译出的`render`函数里会触发`has`拦截，构建工具编译的`render`函数不会。为什么呢？

隐约记得之前看的[源码解读](http://hcysun.me/vue-design/art/6vue-init-start.html#%E6%B8%B2%E6%9F%93%E5%87%BD%E6%95%B0%E7%9A%84%E4%BD%9C%E7%94%A8%E5%9F%9F%E4%BB%A3%E7%90%86)好像提到过这个，就去看了一下。
> 1. 触发`has`操作是因为编译出的`render`函数有`with`语句。
> 2. 在用`webpack`和`vue-loader`时，会将模板编译成不带`with`的语句，并设置`_withStriped`为`true`。 —— [HcySunYang](https://github.com/HcySunYang)
## getHandler
### 作用
在测试时对属性的访问进行拦截，根据规则给出提示。
### 过程
和`hasHandler`差不多，只是没有了全局变量和私有变量的判断。
## 其它
除了上面的函数，这里还引入了[config](../config.md)，如果`hasProxy`，拦截`set`操作，设置`config.keyCodes`不能使用事件修饰符的保留字。
## 代码
```javascript
/* not type checking this file because flow doesn't play well with Proxy */

import config from 'core/config'
import { warn, makeMap, isNative } from '../util/index'

let initProxy

if (process.env.NODE_ENV !== 'production') {
  const allowedGlobals = makeMap(
    'Infinity,undefined,NaN,isFinite,isNaN,' +
    'parseFloat,parseInt,decodeURI,decodeURIComponent,encodeURI,encodeURIComponent,' +
    'Math,Number,Date,Array,Object,Boolean,String,RegExp,Map,Set,JSON,Intl,' +
    'require' // for Webpack/Browserify
  )

  const warnNonPresent = (target, key) => {
    warn(
      `Property or method "${key}" is not defined on the instance but ` +
      'referenced during render. Make sure that this property is reactive, ' +
      'either in the data option, or for class-based components, by ' +
      'initializing the property. ' +
      'See: https://vuejs.org/v2/guide/reactivity.html#Declaring-Reactive-Properties.',
      target
    )
  }

  const warnReservedPrefix = (target, key) => {
    warn(
      `Property "${key}" must be accessed with "$data.${key}" because ` +
      'properties starting with "$" or "_" are not proxied in the Vue instance to ' +
      'prevent conflicts with Vue internals' +
      'See: https://vuejs.org/v2/api/#data',
      target
    )
  }

  const hasProxy =
    typeof Proxy !== 'undefined' && isNative(Proxy)

  if (hasProxy) {
    const isBuiltInModifier = makeMap('stop,prevent,self,ctrl,shift,alt,meta,exact')
    config.keyCodes = new Proxy(config.keyCodes, {
      set (target, key, value) {
        if (isBuiltInModifier(key)) {
          warn(`Avoid overwriting built-in modifier in config.keyCodes: .${key}`)
          return false
        } else {
          target[key] = value
          return true
        }
      }
    })
  }

  const hasHandler = {
    has (target, key) {
      const has = key in target
      const isAllowed = allowedGlobals(key) ||
        (typeof key === 'string' && key.charAt(0) === '_' && !(key in target.$data))
      if (!has && !isAllowed) {
        if (key in target.$data) warnReservedPrefix(target, key)
        else warnNonPresent(target, key)
      }
      return has || !isAllowed
    }
  }

  const getHandler = {
    get (target, key) {
      if (typeof key === 'string' && !(key in target)) {
        if (key in target.$data) warnReservedPrefix(target, key)
        else warnNonPresent(target, key)
      }
      return target[key]
    }
  }

  initProxy = function initProxy (vm) {
    if (hasProxy) {
      // determine which proxy handler to use
      const options = vm.$options
      const handlers = options.render && options.render._withStripped
        ? getHandler
        : hasHandler
      vm._renderProxy = new Proxy(vm, handlers)
    } else {
      vm._renderProxy = vm
    }
  }
}

export { initProxy }

```