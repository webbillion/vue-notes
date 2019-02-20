---
vue-note-completed: true
---
# core/instance.state.js
## stateMixin
### 作用
为`Vue`的原型添加`$data`等属性，`$set`等方法。
### 过程
所以定义都指在`Vue.prototype`上定义。
1. 定义`dataDef`和`propsDef`，将对`$data`和`$props`的`get`访问代理到`_data`和`_props`上。这两个属性在[这里]()设置。如果非生产环境，对`$data`和`$props`的`set`访问设置警告，即不允许设置。
2. 添加`$set`和`$delete`方法，用处参考[global-api/index.js](../global-api/index.md)，来自[set]()和[del]()。
3. 添加`$watch`方法
- 作用
     <br>在`watch`里没有定义的话，可以用这个方法进行监听。
- 参数
  - `expOrFn` 表达式或者函数。
  - `cb` 回调。
  - `options` 选项。
- 返回
  - 取消监听的函数。
- 过程
  1. 定义`vm = this`。
  2. 如果回调是一个纯对象，将参数传入，调用并返回[createWatcher]()。它的作用是对参数进行一些处理后再调用`vm.$watch`。
  3. 否则初始化选项并添加`user = true`，这个属性用来选择调用`expOrFn`出错的时的错误处理方式。
  4. 创建一个`watcher`，即[观察者]()。这里简单理解为，就是监听`vm`的`expOrFn`属性的变化，并调用`cb`。如果选项中有`immediate`，就立即调用一次回调。
  5. 返回用于取消监听的函数。
## initState
### 作用
初始化状态，依次是：
- `props`
- `methods`
- `data`
- `computed`
- `watch`

在过程中分别调用对应的初始化函数`init*`，如果存在的话。从这里也能看出，我们在什么时候能用什么选项，比如在`data(){}`中是可以调用`methods`和访问`props`的。除此之外需要注意的点有：

1. 即使没有`data`选项，也要初始化一个`vm._data = {}`空对象，并进行观测[observe]()，它的作用就是观察一个对象。目的是作为根数据，否则无法进行后续操作，比如在`render`中进行判断时。
2. `watch`属性需要不等于`nativeWatch`，这个`nativeWatch`是`Firefox`浏览器中对象原型上有的一个属性，所以这里需要排除。经过测试，`Firefox`已经移除这个属性了。
3. - [ ] 对于`initData`，只传入了`vm`，其它函数则传入了`vm`和对应的选项，暂时不知道为啥。
## initProps
### 过程
1. 从选项中获取`propsData`，即父组件传递的数据,如果没有置为空对象。
2. 在实例上定义`_props`为空对象，并保留它的引用`props`，它作用是代理对`props`的访问。
3. 在实例选项上定义`_propsKeys`为空数组，并保留引用`keys`，这是为了缓存`props`的属性，在[`props`更新]()的时候用这个数组来遍历比较，而不是用动态的对象检查它枚举的键。
4. 定义`isRoot`，是否是根组件，即没有父组件就为真。
5. 如果不是根组件，就调用`toggleObserving(false)`，对于`props`中的值不做深度观测，只观测第一层。这是因为`props`数据是从外部获取的。
6. 接着遍历`propsOptions`中的所有属性。
   1. 将`key`放入`keys`中缓存。
   2. 调用[validateProp](../util/props.md)验证有效性并获取当前`key`的初始值`value`，同时如果这个`value`是一个对象的话，它也会被变成一个`obrserver`。
   3. 如果是生产环境，直接调用`defineReactive`将其变为响应式属性并进入第Ⅵ步。否则添加一些开发时提示。
   4. 调用`hyphenate`（字符串拼接）获取真实的`key`值，简单理解为将`value`变成`_value`，验证是否是保留属性（`key` `ref`等），除了这些还可以在配置中添加额外的验证函数，一般由平台重写，参考[config.isReservedAttr](../config.md)。如果是的话给出警告。
   5. 在调用`defineReactive`时传入一个函数，在`props`中的值被设置时触发，如果当前组件不是根组件，而且当前没有在更新子组件（`isUpdatingChildComponent`，很容易想到在更新子组件时会将其设为`false`），即我们手动设置`props`时给出警告。
   6. 如果当前`key`没有在`vm`中被使用，将对`vm.key`的访问和设置代理到`vm._props`上。
7. 切换观测开关，`toggleObserving(true)`。
8. 完毕。
## initData
### 过程
1. 获取选项中的`data`，根据它是函数还是对象获取其真实的值，并将其设置`vm._data`，我们知道`data`是有几种形式的。
   1. 如果是函数，调用`getData(data, vm)`获取其返回的值，它的作用参考下文。
   2. 是对象保留原值。
   3. 为空初始化为空对象。
2. 如果得到的`data`不是一个纯对象，将其置为空对象，同时开发环境下给出警告。
3. 接着遍历`data`的所有属性，如果这个属性
    1. 在`props`中有定义了。
    2. 在`methods`中有定义了。
    3. 非生产环境下给出对应的警告。
4. 如果不是保留属性，即不是以`_`或者`$`开头，将对`vm.key`的访问和设置代理到`vm._data`上。
5. 将`data`变为`observer`，这里传入了第二个参数，即它是根数据，用来计数，具体参考对应分析。
## getData
如果传入的`data`是一个函数，调用此函数获取数据。
### 过程
1. 调用`pushTarget()`，这么做是为了在获取`data`时禁用依赖收集，来自[issue#7573](https://github.com/vuejs/vue/issues/7573)，有兴趣可以参考[issues/issue-7573](/issues/issue-7573.md)。
2. 尝试调用`data.call(vm, vm)`，也就是说，其实我们在组件中定义`data`函数时，其实它是可以接受一个参数的，也就是它自己，虽然我从来没这么用过。
3. 如果出错了处理错误，并返回一个空对象。
4. 完毕。
## initComputed
在开始定义函数之前声明了一个变量`computedWatcherOptions = { lazy: true }`，即计算属性配置，`lazy`即是官网提到的，计算属性可以被缓存。
### 过程
1. 为`vm._computedWatchers`赋值一个完全的空对象，并保留引用`watchers`。
2. 获取`isSSR`，即是否是服务端渲染，在服务端渲染时计算属性的`setter`是不生效的。
3. 接着遍历传入的`computed`对象：
    1. 获取值`userDef = computed[key]`，它可能是一个函数，也可能是一个由`get` `set`属性的对象。是函数就相当于只有`get`。
    2. 获取`getter`。
    3. 非生产环境下如果`getter`为空，给出警告。
    4. 如果不是服务端渲染，为这个`key`创建一个`watcher`。
    5. 如果这个`key`在`vm`中不存在，调用`defineComputed`（参见下文）将对`vm.key`的访问和设置都代理到`userDef`上。
    6. 如果它已经存在了，就看它有没有在`$data`(等同于`_data，下同`)和`$props`中定义（因为在前面这个属性可能是被代理到`_props`和`_data`上了，至于为什么不检查在`methods`中有没有，可能是因为觉得没有必要吧），如果有，给出对应的提示。
4. 完毕。
## defineComputed
在此之前了解一个变量`sharedPropertyDefinition`，在最开始的时候定义的，通过设置它的`get`和`set`，来使用`Object.defineProperty`。
### 过程
1. 首先定义`shouldCache`，是否应该缓存，非服务端渲染时才缓存。
2. 如果`userDef`是一个函数，定义`sharedPropertyDefinition.get`：
     1. 如果应该缓存，调用[createComputedGetter](#createComputedGetter)创建`getter`。
     2. 否则直接使用[createGetterInvoker](#createGetterInvoker)创建`getter`。
3. 设置`set`为`noop`，即什么也不做。
4. 如果`userDef`是一个对象，和是一个函数的设置差不多，只是在是否缓存的判断上除了需要`shouldCache`，还需要`userDef.cache`。
5. 如果非生产环境且`set`为`noop`，将`set`设置为一个警告函数，即如果没有给`computed`属性指定`setter`，就不要手动去改变它。
6. 调用`Object.defineProperty`将对`vm.key`的访问和设置代理到`sharedPropertyDefinition`上。
7. 完毕。
## createComputedGetter
它返回真正的`getter`函数。
### 过程
1. 获取当前`key`的`watcher`。
2. 如果`watcher.dirty`为真，即开启脏检查，这个属性的初始值和`computedWatcherOptions.lazy`的值相同，即为真。调用`watcher.evaluate`，即进行一次求值并将`dirty`设为`false`。可以看到，如果`dirty`为`false`，即依赖的其它属性没有变化，将不会调用`watcher.evaluate`，也就是不会再次求值，而是直接返回之前的值`watcher.value`，也就做到了缓存。
3. 接着如果当前`Dep.target`有值，调用`watcher.depend`，将当前`watcher`所依赖的属性都放入`Dep.target`的`deps`中。即，比如，我们在模板中依赖了计算属性`a`，而`a`又依赖`data.b`和`data.c`，将`renderWatcher`放入对应`dep`的`sbus`中<br><br>
   > - [ ] 一个问题是，即使不做这一步，也能正确响应依赖的变化，正如用`createGetterInvoker`一样。那么这么做的目的是什么呢？<br>
4. 返回`watcher.value`。
5. 完毕。
## createGetterInvoker
返回一个绑定了作用域的并有参数的`getter`。在之前，如果需要缓存数据，用`watcher`来调用计算属性的函数的话，它是接受`vm`作为第一个参数的，和`data`函数一样。服务端渲染中是不需要缓存的，因此也就获取不到第一个参数，可能为了保证一致性，或者因为有人提出来了总得解决，就在这里处理了。它来自[issue#8977](https://github.com/vuejs/vue/issues/8977)。
## initMethods
它比较简单，就是检查`methods`中的值是否是函数，或者在`props`中已有定义，或者是保留的，给出对应警告。然后将其绑定作用域并设置到`vm`上。
## initWatch
和`initMethods`差不多，遍历`watch`中的属性依次调用`createWatcher`即可，需要注意的是这里的`handler`可能是一个数组，参考[选项合并]()。
## createWatcher
规范参数将其传入`vm.$watch`中，规范点有
- `handler` 可以是一个函数，可以是一个对象，常用于深度观测。
```javascript
watch: {
  a: {
      deep: true,
      handler () {

      }
  }
}
```
除此之外也可以是一个字符串，这样`handler`将会等于`vm[handler]`，即我们可以传入在`methods`中定义好的方法名，反正我是现在才知道。
## 代码
```javascript
/* @flow */

import config from '../config'
import Watcher from '../observer/watcher'
import Dep, { pushTarget, popTarget } from '../observer/dep'
import { isUpdatingChildComponent } from './lifecycle'

import {
  set,
  del,
  observe,
  defineReactive,
  toggleObserving
} from '../observer/index'

import {
  warn,
  bind,
  noop,
  hasOwn,
  hyphenate,
  isReserved,
  handleError,
  nativeWatch,
  validateProp,
  isPlainObject,
  isServerRendering,
  isReservedAttribute
} from '../util/index'

const sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: noop,
  set: noop
}

export function proxy (target: Object, sourceKey: string, key: string) {
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}

export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}

function initProps (vm: Component, propsOptions: Object) {
  const propsData = vm.$options.propsData || {}
  const props = vm._props = {}
  // cache prop keys so that future props updates can iterate using Array
  // instead of dynamic object key enumeration.
  const keys = vm.$options._propKeys = []
  const isRoot = !vm.$parent
  // root instance props should be converted
  if (!isRoot) {
    toggleObserving(false)
  }
  for (const key in propsOptions) {
    keys.push(key)
    const value = validateProp(key, propsOptions, propsData, vm)
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      const hyphenatedKey = hyphenate(key)
      if (isReservedAttribute(hyphenatedKey) ||
          config.isReservedAttr(hyphenatedKey)) {
        warn(
          `"${hyphenatedKey}" is a reserved attribute and cannot be used as component prop.`,
          vm
        )
      }
      defineReactive(props, key, value, () => {
        if (!isRoot && !isUpdatingChildComponent) {
          warn(
            `Avoid mutating a prop directly since the value will be ` +
            `overwritten whenever the parent component re-renders. ` +
            `Instead, use a data or computed property based on the prop's ` +
            `value. Prop being mutated: "${key}"`,
            vm
          )
        }
      })
    } else {
      defineReactive(props, key, value)
    }
    // static props are already proxied on the component's prototype
    // during Vue.extend(). We only need to proxy props defined at
    // instantiation here.
    if (!(key in vm)) {
      proxy(vm, `_props`, key)
    }
  }
  toggleObserving(true)
}

function initData (vm: Component) {
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    )
  }
  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) {
      proxy(vm, `_data`, key)
    }
  }
  // observe data
  observe(data, true /* asRootData */)
}

export function getData (data: Function, vm: Component): any {
  // #7573 disable dep collection when invoking data getters
  pushTarget()
  try {
    return data.call(vm, vm)
  } catch (e) {
    handleError(e, vm, `data()`)
    return {}
  } finally {
    popTarget()
  }
}

const computedWatcherOptions = { lazy: true }

function initComputed (vm: Component, computed: Object) {
  // $flow-disable-line
  const watchers = vm._computedWatchers = Object.create(null)
  // computed properties are just getters during SSR
  const isSSR = isServerRendering()

  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    if (process.env.NODE_ENV !== 'production' && getter == null) {
      warn(
        `Getter is missing for computed property "${key}".`,
        vm
      )
    }

    if (!isSSR) {
      // create internal watcher for the computed property.
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }

    // component-defined computed properties are already defined on the
    // component prototype. We only need to define computed properties defined
    // at instantiation here.
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    } else if (process.env.NODE_ENV !== 'production') {
      if (key in vm.$data) {
        warn(`The computed property "${key}" is already defined in data.`, vm)
      } else if (vm.$options.props && key in vm.$options.props) {
        warn(`The computed property "${key}" is already defined as a prop.`, vm)
      }
    }
  }
}

export function defineComputed (
  target: any,
  key: string,
  userDef: Object | Function
) {
  const shouldCache = !isServerRendering()
  if (typeof userDef === 'function') {
    sharedPropertyDefinition.get = shouldCache
      ? createComputedGetter(key)
      : createGetterInvoker(userDef)
    sharedPropertyDefinition.set = noop
  } else {
    sharedPropertyDefinition.get = userDef.get
      ? shouldCache && userDef.cache !== false
        ? createComputedGetter(key)
        : createGetterInvoker(userDef.get)
      : noop
    sharedPropertyDefinition.set = userDef.set || noop
  }
  if (process.env.NODE_ENV !== 'production' &&
      sharedPropertyDefinition.set === noop) {
    sharedPropertyDefinition.set = function () {
      warn(
        `Computed property "${key}" was assigned to but it has no setter.`,
        this
      )
    }
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}

function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}

function createGetterInvoker(fn) {
  return function computedGetter () {
    return fn.call(this, this)
  }
}

function initMethods (vm: Component, methods: Object) {
  const props = vm.$options.props
  for (const key in methods) {
    if (process.env.NODE_ENV !== 'production') {
      if (typeof methods[key] !== 'function') {
        warn(
          `Method "${key}" has type "${typeof methods[key]}" in the component definition. ` +
          `Did you reference the function correctly?`,
          vm
        )
      }
      if (props && hasOwn(props, key)) {
        warn(
          `Method "${key}" has already been defined as a prop.`,
          vm
        )
      }
      if ((key in vm) && isReserved(key)) {
        warn(
          `Method "${key}" conflicts with an existing Vue instance method. ` +
          `Avoid defining component methods that start with _ or $.`
        )
      }
    }
    vm[key] = typeof methods[key] !== 'function' ? noop : bind(methods[key], vm)
  }
}

function initWatch (vm: Component, watch: Object) {
  for (const key in watch) {
    const handler = watch[key]
    if (Array.isArray(handler)) {
      for (let i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i])
      }
    } else {
      createWatcher(vm, key, handler)
    }
  }
}

function createWatcher (
  vm: Component,
  expOrFn: string | Function,
  handler: any,
  options?: Object
) {
  if (isPlainObject(handler)) {
    options = handler
    handler = handler.handler
  }
  if (typeof handler === 'string') {
    handler = vm[handler]
  }
  return vm.$watch(expOrFn, handler, options)
}

export function stateMixin (Vue: Class<Component>) {
  // flow somehow has problems with directly declared definition object
  // when using Object.defineProperty, so we have to procedurally build up
  // the object here.
  const dataDef = {}
  dataDef.get = function () { return this._data }
  const propsDef = {}
  propsDef.get = function () { return this._props }
  if (process.env.NODE_ENV !== 'production') {
    dataDef.set = function () {
      warn(
        'Avoid replacing instance root $data. ' +
        'Use nested data properties instead.',
        this
      )
    }
    propsDef.set = function () {
      warn(`$props is readonly.`, this)
    }
  }
  Object.defineProperty(Vue.prototype, '$data', dataDef)
  Object.defineProperty(Vue.prototype, '$props', propsDef)

  Vue.prototype.$set = set
  Vue.prototype.$delete = del

  Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: any,
    options?: Object
  ): Function {
    const vm: Component = this
    if (isPlainObject(cb)) {
      return createWatcher(vm, expOrFn, cb, options)
    }
    options = options || {}
    options.user = true
    const watcher = new Watcher(vm, expOrFn, cb, options)
    if (options.immediate) {
      try {
        cb.call(vm, watcher.value)
      } catch (error) {
        handleError(error, vm, `callback for immediate watcher "${watcher.expression}"`)
      }
    }
    return function unwatchFn () {
      watcher.teardown()
    }
  }
}

```