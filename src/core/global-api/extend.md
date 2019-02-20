---
vue-note-completed: true
---
# core/globa-api/extend.js
## 作用
以Vue为基础创建子类，我理解为创建一个未实例化的组件。
## 使用
```javascript
const Component = Vue.extend({
  template: '',
  //...
})
new Component().$mount('#component')
```
## 解析
首先给`Vue`添加`cid`属性并置为0，此属性注释有提到，每一个构造函数，即`Vue`和和其子类，都有一个唯一的cid，这么做的目的是为了能够继承和缓存。
- [ ] 缓存能够理解，继承和`cid`的关系目前还不清楚。

创建初始值为1的变量`cid`，应该后面每扩展一个子类都会加一。<br>
接着在`Vue`上添加`extend`方法。这里有注释说类继承。
### 参数
- `extendOptions` 可选，扩展选项，即一个组件需要的选项，包括`data`等。
### 返回。
- 返回子类
### 过程
1. 定义变量。
  - `Super` 存储自身。
  - `SuperId` 存储自身的`cid`。
  - `cachedCtors` `extendOptions`的`_Ctor`属性，如果没有则置为空对象，应该是`constructor`的意思吧。
2. 如果`cachedCtors`中有自身的`cid`，则直接返回。否则进入第3步。
3. 获取选项的`name`，如果不是生产环境就对`name`做验证，规则是组件名规范。
4. 定义子类`Sub`，它的构造函数和[Vue](../instance/index.md)函数体相同，只是没了提示。
5. 接着令`Sub`的原型继承`Super`的原型，自此，`Sub`实例和`Super`实例有相同的功能。然后修整`Sub.prototype.添加constructor`的指向。
   - [ ] 关于原型链继承之后再修正构造函数指向的问题，都是好久之前看的了，差点没反应过来是在干嘛。可能需要复习一下了。
6. 为`Sub`添加`cid`属性，同时`cid`自增1.
7. 合并`Super.options`和`extendOptions` 到`Sub.options`上，即合并父类的传入的选项。
   - [ ] 这里牵扯到[合并策略]()和[mergeOptions]()
8. `Sub['super'] = Super`
   - [ ] 没明白为什么不用`Sub.super`
9. 如果在选项中传入了`props`或者`computed`，调用`initProps`或`initComputed`，这两个方法在下面有解释，这里暂不深究。但是显然能猜到的一点是，本来这两个过程应该是在我们使用`new Vue()`之后才会发生的，这里却还没有实例化。作者这里注释了这么做的目的：
> 对于props和computed两个属性，我们在扩展Vue时就在其原型上定义了代理，这是为了避免每次创建扩展实例时都调用Object.defineProperty。

- [x] 我暂时对这段话的理解为，用`Vue.extend`创建了一个未实例化的组件`Component`，之后可以多次`new Component()`来实例化多个相同构成的组件，对所有实例来说，`props`和`copmputed`属性的处理结果是一样的，所以在这里就先处理了。具体是为什么还不是很懂。
> 在看到[initState](../instance/state.md)这一块后，我明白了这一块，在初始化`props`的过程中，是对`props`中的属性一个一个进行代理的，在这里对`extend`时传入的`props`进行初始化后，实例化组件时，就只需要对`props`中新传入的项进行代理即可。
10. 为`Sub`添加`Super`的`extend`、`mixin`、`use`三个方法。
11. 为`Sub`添加`Super`的`component`、`directive`、`filter`三个资源注册方法，这是为了扩展类可以有它们私有的资源。比如说扩展类可以有它才能使用的组件。
    - [ ] 这之后是指什么时候。
12. 如果`name`存在，将其添加到在`Sub`的组件里，目的是为了能使用递归组件。
    - [ ] 自身拥有自身好理解。调用`Sub.extend`可以再创建一个子类`Sub2`，如果`Sub`有`name`，`Sub2`没有，则`Sbu2`的组件库里名为`${name}`的组件仍然是`Sub`，所以需要做这个修正。
13. 接着在`Sub`上添加了对`Super.options`和`extendOptions`的引用，同时复制了一份`Sub.options`到`Sub`上，这是为了在之后实例化时检查`Super`的`options`是否有更新。参考[_init](../instance/init.md)。
14. 缓存构造函数，`cachedCtors[SuperId] = Sub`。这时回到第2步，什么情况下会出现到不了第3步的情况？想来想去，应该用同一个父类，且是将同一份（指引用不变）`extendOptions`多次传入时才会这样。
   - [ ] 为什么这么做？怎么会想到在这里对这个做缓存的，不做有什么影响？
15. 返回`Sub`
### 其它函数
#### initProps
它遍历了选项中的`props`属性每个键，并将其代理，达到的效果是
```javascript
const Sub = Vue.extend({
  props: {
    a: String
  }
})
const sub = new Sub()
sub.a
// === Sub.prototype._props.a
```
> 在实例化的时候，就只需要对新的`props`进行初始化了。
#### initComputed
和上面类似，达到效果是
```javascript
const Sub = Vue.extend({
  computed: {
    a () {
      return 1
    }
  }
})
const sub = new Sub()
sub.a
// === Sub.prototype.a

```
#### proxy
它定义在[proxy](../instance/state.md#proxy)
#### defineComputed
它定义在[defineComputed](../instance/state.md#defineComputed)
### 代码
```javascript
/* @flow */

import { ASSET_TYPES } from 'shared/constants'
import { defineComputed, proxy } from '../instance/state'
import { extend, mergeOptions, validateComponentName } from '../util/index'

export function initExtend (Vue: GlobalAPI) {
  /**
   * Each instance constructor, including Vue, has a unique
   * cid. This enables us to create wrapped "child
   * constructors" for prototypal inheritance and cache them.
   */
  Vue.cid = 0
  let cid = 1

  /**
   * Class inheritance
   */
  Vue.extend = function (extendOptions: Object): Function {
    extendOptions = extendOptions || {}
    const Super = this
    const SuperId = Super.cid
    const cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {})
    if (cachedCtors[SuperId]) {
      return cachedCtors[SuperId]
    }

    const name = extendOptions.name || Super.options.name
    if (process.env.NODE_ENV !== 'production' && name) {
      validateComponentName(name)
    }

    const Sub = function VueComponent (options) {
      this._init(options)
    }
    Sub.prototype = Object.create(Super.prototype)
    Sub.prototype.constructor = Sub
    Sub.cid = cid++
    Sub.options = mergeOptions(
      Super.options,
      extendOptions
    )
    Sub['super'] = Super

    // For props and computed properties, we define the proxy getters on
    // the Vue instances at extension time, on the extended prototype. This
    // avoids Object.defineProperty calls for each instance created.
    if (Sub.options.props) {
      initProps(Sub)
    }
    if (Sub.options.computed) {
      initComputed(Sub)
    }

    // allow further extension/mixin/plugin usage
    Sub.extend = Super.extend
    Sub.mixin = Super.mixin
    Sub.use = Super.use

    // create asset registers, so extended classes
    // can have their private assets too.
    ASSET_TYPES.forEach(function (type) {
      Sub[type] = Super[type]
    })
    // enable recursive self-lookup
    if (name) {
      Sub.options.components[name] = Sub
    }

    // keep a reference to the super options at extension time.
    // later at instantiation we can check if Super's options have
    // been updated.
    Sub.superOptions = Super.options
    Sub.extendOptions = extendOptions
    Sub.sealedOptions = extend({}, Sub.options)

    // cache constructor
    cachedCtors[SuperId] = Sub
    return Sub
  }
}

function initProps (Comp) {
  const props = Comp.options.props
  for (const key in props) {
    proxy(Comp.prototype, `_props`, key)
  }
}

function initComputed (Comp) {
  const computed = Comp.options.computed
  for (const key in computed) {
    defineComputed(Comp.prototype, key, computed[key])
  }
}

```