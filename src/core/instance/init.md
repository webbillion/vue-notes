---
vue-note-completed: true
---
# core/instance/init.js
## 作用
为`Vue`的原型添加`_init`方法，`new Vue()`时就会调用这个方法进行处理。
## 解析
首先引入一些方法和变量。接着导出`initMixin`，为`Vue.prototype`添加`_init`方法。
### 参数
- `options` 选项，可选，即`props`这些。
### 返回
- `vm` 组件实例 函数本身不返回什么，但是因为是`new`的，所以等同于返回这个。
### 过程
1. 定义`vm`保存上下文。
2. 添加`uid`，每个实例有唯一的`uid`。
3. 定义`startTag`、`endTag`，开始标签和结束标签，如果在[config](../config.md)中开启了记录性能，用来调用[mark]()记录。
4. 添加`_isVue`属性为`true`，用来避免自身被观察。
   - [x] 牵扯到[观察者系统](../observer/README.md)。
5. 如果`options`存在且`options._isComponent`，即是内部组件实例化。内部组件的意思是在`components`属性中添加的组件，这样的话创建这个组件时传入的`options`是包含父组件的一些信息的。则调用[initInternalComponent](#initInternalComponent)进行性能优化。因为动态合并选项是很慢的，而且内部组件所以没有特殊选项需要处理。
   - [ ]  如何是性能优化暂时没有懂。
6. 否则调用[mergeOption]()合并选项并在`vm`上添加`$options`属性。需要在选项中包含自定义属性时会有用处。
7. 是生产环境的话定义`vm._renderProxy`为`vm`自己，否则调用[initProxy]()，这个属性是用来在开发时提示你访问了没有定义的属性或者方法之类的。
8. 定义`vm._self`为自己，暴露真实的实例。
9. 接着调用了以下函数对`vm`进行处理，这里也能看到一个`vue组件`是如何一步一步实例化的：
   - [initLifecycle(vm)](./lifecycle.md) 初始化生命周期
   - [initEvents(vm)](./events.md) 处理事件系统。
   - [initRender(vm)](./render.md) 初始化渲染函数。
   - [callHook(vm, 'beforeCreate')](./lifecycle.md) 调用钩子函数，这时`beforeCreate`被调用。
   - [initInjections(vm)](./inject.md) 在解析`data/props`之前处理注入。
   - [initState(vm)](./state.md) 初始化`data/props`。
   - [initProvide(mv)](./inject.md) 在解析`data/props`之后处理`provide`。
   - `callHook(vm, 'created')` 调用钩子函数，这时`created`被调用。
9. 如果有，结束性能记录。
10. 如果选项中有`el`，调用[$mount]()进行挂载。
11. 完毕。
## 其它函数
### initInternalComponent
对内部组件进行初始化。其中涉及的一些属性可以参考[这里]()。
#### 参数
 - `vm` 组件实例
 - `options` `InternalComponentOptions`内部组件选项
#### 过程
1. 定义`opts`变量是以`vm.constructor.options`也就是`Vue`或者`Vue.extend()`的选项为原型，即`opts`并将`vm.$options`也设置为这个值。这样`opts`就有和`vm`构造函数有相同的选项了。
2. 在`opts`上添加属性，它们是
   - `parent` = `options.parent`，即父组件。即
   ```html
    <!-- a组件  -->
    <template>
      <b></b>
    </template>
    <!-- 则b.parent = a -->
   ```
   - `_parentVnode` = `options._parentVnode`，父组件的虚拟节点。
   - 定义`vnodeComponentOptions` = `options._parentVnode.vnodeComponentOptions`
   - `propsData` = `vnodeComponentOptions.propsData`,父组件的虚拟节点的`propsData`选项，即这个组件定义的`props`会从这里拿到。
   - `_parentListeners` = `vnodeComponentOptions.listeners`，父组件对子组件监听的事件。
   - `_renderChildren` = `vnodeComponentOptions.children`，即本子组件的子组件，如下，一个`vnode`数组。
    ```html
    <!-- a组件  -->
    <template>
      <b>
        <img>
        <input>
      </b>
    </template>
    <!-- 则_renderChildren=[img, input] -->
   ```
   - `_componentTag` = `vnodeComponentOptions.tag`，父组件标签名？

如果`options.render`存在的话
-   - `render` = `options.render`，渲染函数，数据变化时会调用这个函数重新渲染，
        - [ ] 参考[render]()。
    - `staticRenderFns` = `options.staticRenderFns`，静态渲染函数数组，优化性能所用，负责那些静态的dom内容，
      - [ ] 参考[staticRenderFns]()。
### resolveConstructorOptions
解析构造函数的`options`
#### 参数
  - `Ctor` 构造函数，即`Vue`和其子类
#### 返回
  - `options` 构造函数及其父类和祖先的`options`
#### 过程
1. 定义`options`为`Ctor.options`。
2. 如果`Ctor.super`存在，即它是由[扩展](../global-api/extend.md)得到的。进入下一步，否则直接返回
3. 获取它的`options`并保存为`superOptions`。
4. 定义`cachedSuperOptions`为`Ctor.superOptions`，它在[extend](../global-api/extend.md)时被设置，也在在下面重新设置`superOptions`。
5. 如果最新的`superOptions`和缓存结果不一样，重新设置`Ctor.superOptions`。
6. 检查是否有任何后期修改或者添加的选项，之所以这么做是因为[issue4976](https://github.com/vuejs/vue/issues/4976)，如果有兴趣了解，可以看[issues/issue-4976.md](/issues/issue-4976.md)。如果有的话将`Ctor.extendOptions`和`modifiedOptions`合并成一个对象。`resolveModifiedOptions`函数见下文。
7. 合并`superOptions`和`Ctor.extendOptions`两个选项。合并对象是和合并选项是不同的，其它地方不再说明。并将合并后的选项赋值给`options`和`Ctor.options`。
8. 如果选项有命名，则将其添加到`components`中，前文提到过是为了递归组件。
9. 返回`options`。
这段可能不是很好理解，举例说明说如下。
```javascript
const TestOptions = {
  name: 'test',
  data: () => ({a: 1})
}
const Test = Vue.extend(TestOptions)
new Test()
// 则在resolveConstructorOptions过程中
let options = Test.options // 包括Vue.options和TestOptions
if (Test.super) { // Test.super === Vue，所以往下走
  const superOptions = resolveConstructorOptions(Vue) // 因为Vue没有super，所以就是Vue.options
  const cachedSuperOptions = Test.superOptions // undefined
  if (superOptions !== cachedSuperOptions) { // 所以往下走
    Test.superOptions = superOptions // 也就是Vue.options
     /* const modifiedOptions = resolveModifiedOptions(Test)
     if (modifiedOptions) {
        extend(Ctor.extendOptions, modifiedOptions)
      } */
     // 暂时跳过这一步
     // Test.extendOptions，即是TestOptions
     // 最后将Test.options置为mergeOptions(Vue.options, TestOptions)
     options = Test.options = mergeOptions(superOptions, Test.extendOptions)
     // 组件名跳过
  }
  return options
}
```
这个函数的作用，是解析构造函数的选项，回想在[Vue.extend](../global-api/extend.md)中,上面例子的`Test.options`，本来就等于`mergeOptions(Vue.options, TestOptions)`，这里再次做了相同的工作，看到这里我明白了，反正我是现在才明白，也就是，在实例化组件时，如果父类的选项有变化，则重新合并父类和扩展的选项。也是到了这里才明白`extend`中对于`superOptions`的注释是什么意思。问题是，什么情况下会发生变化呢？这里的变化是引用发生变化才行，即对`Super.options`重新赋值才可以。
 - [x] 我暂时还没想到应用场景。
 - [x] 相反当调用`Vue.component`注册组件后，需要重新合并，但是它并没有改变引用。
 - [x] 在[vdom/create-component.js](../vdom/create-component.md)看到了注释才想起，在使用`Vue.mixin`后，会对`options`重新赋值。

另一点值得注意的是，在修复[issue4976](https://github.com/vuejs/vue/issues/4976)之前，`superOptions`不是像现在一样递归去获取的，而是直接`superOptions = Ctor.super.options`，也就是说
```javascript
const Test = Vue.extend()
const Test2 = Test.extend()
// 即使
Vue.options = {}
// 在实例化Test2的时候，Test的options并没有变化，也不会进入后续流程，应该
```
### resolveModifiedOptions
这个函数比较简单，就是比较`Ctor.options`、`Ctor.extendOptions`、`Ctor.sealedOptions`，找出在`Ctor.options`中有，而`Ctor.sealedOptions`中没有或者不同的属性，再交由`dedupe`处理。
### dedupe
在上面找出不同后之所以不直接返回变化，主要是为了不让合并后的选项中的生命周期相关钩子函数被重复调用，或者不被调用。如果属性是数组的话就做一个合并和去重即可。
## 代码
```javascript
/* @flow */

import config from '../config'
import { initProxy } from './proxy'
import { initState } from './state'
import { initRender } from './render'
import { initEvents } from './events'
import { mark, measure } from '../util/perf'
import { initLifecycle, callHook } from './lifecycle'
import { initProvide, initInjections } from './inject'
import { extend, mergeOptions, formatComponentName } from '../util/index'

let uid = 0

export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    // a uid
    vm._uid = uid++

    let startTag, endTag
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      startTag = `vue-perf-start:${vm._uid}`
      endTag = `vue-perf-end:${vm._uid}`
      mark(startTag)
    }

    // a flag to avoid this being observed
    vm._isVue = true
    // merge options
    if (options && options._isComponent) {
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      initProxy(vm)
    } else {
      vm._renderProxy = vm
    }
    // expose real self
    vm._self = vm
    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)
    callHook(vm, 'beforeCreate')
    initInjections(vm) // resolve injections before data/props
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')

    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      vm._name = formatComponentName(vm, false)
      mark(endTag)
      measure(`vue ${vm._name} init`, startTag, endTag)
    }

    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}

export function initInternalComponent (vm: Component, options: InternalComponentOptions) {
  const opts = vm.$options = Object.create(vm.constructor.options)
  // doing this because it's faster than dynamic enumeration.
  const parentVnode = options._parentVnode
  opts.parent = options.parent
  opts._parentVnode = parentVnode

  const vnodeComponentOptions = parentVnode.componentOptions
  opts.propsData = vnodeComponentOptions.propsData
  opts._parentListeners = vnodeComponentOptions.listeners
  opts._renderChildren = vnodeComponentOptions.children
  opts._componentTag = vnodeComponentOptions.tag

  if (options.render) {
    opts.render = options.render
    opts.staticRenderFns = options.staticRenderFns
  }
}

export function resolveConstructorOptions (Ctor: Class<Component>) {
  let options = Ctor.options
  if (Ctor.super) {
    const superOptions = resolveConstructorOptions(Ctor.super)
    const cachedSuperOptions = Ctor.superOptions
    if (superOptions !== cachedSuperOptions) {
      // super option changed,
      // need to resolve new options.
      Ctor.superOptions = superOptions
      // check if there are any late-modified/attached options (#4976)
      const modifiedOptions = resolveModifiedOptions(Ctor)
      // update base extend options
      if (modifiedOptions) {
        extend(Ctor.extendOptions, modifiedOptions)
      }
      options = Ctor.options = mergeOptions(superOptions, Ctor.extendOptions)
      if (options.name) {
        options.components[options.name] = Ctor
      }
    }
  }
  return options
}

function resolveModifiedOptions (Ctor: Class<Component>): ?Object {
  let modified
  const latest = Ctor.options
  const extended = Ctor.extendOptions
  const sealed = Ctor.sealedOptions
  for (const key in latest) {
    if (latest[key] !== sealed[key]) {
      if (!modified) modified = {}
      modified[key] = dedupe(latest[key], extended[key], sealed[key])
    }
  }
  return modified
}

function dedupe (latest, extended, sealed) {
  // compare latest and sealed to ensure lifecycle hooks won't be duplicated
  // between merges
  if (Array.isArray(latest)) {
    const res = []
    sealed = Array.isArray(sealed) ? sealed : [sealed]
    extended = Array.isArray(extended) ? extended : [extended]
    for (let i = 0; i < latest.length; i++) {
      // push original options and not sealed options to exclude duplicated options
      if (extended.indexOf(latest[i]) >= 0 || sealed.indexOf(latest[i]) < 0) {
        res.push(latest[i])
      }
    }
    return res
  } else {
    return latest
  }
}

```