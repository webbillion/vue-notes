# 响应式
> - 本文目的：对响应式原理和相关`Api`有大概的理解，以便理解响应式这一章更详细的源码分析。
> - 同时如果在阅读其它部分源码时遇到响应式相关的内容，暂时不想深入了解，又不想因此阻碍继续分析，看了本文应该能达到这个目的。
> - 如果已经有大概了解了，请直接阅读[正文](./index.md)。
> - 阅读本文之前你需要对`Object.defineProperty`有所了解。<br>
> - 此外不管是本版本用的`Object.defineProperty`还是3.0版本用的`Proxy`，原理都大致相同。

## 什么是响应式？
在`vue`中，我们指的响应式是指数据发生变化时能重新渲染，或者触发对应的`watch`函数，或者对`computed`属性重新计算，这些总结起来都是一样的，即数据变化时触发对应的函数。
## 如何做
用`Object.defineProperty`拦截属性的`set`操作，很容易做到这一点。<br>
**注意：** 因为这里只是写一个大概的过程，所以在下面的代码中都只对`target.a`进行监听，多个属性多做一些工作即可。
```javascript
let target = {
  _a: 1
}
function cbWhenSetA(val) {
  console.log(`我被设置为${val}`)
}
Object.defineProperty(target, 'a', {
  get () {
    return this._a
  },
  set(val) {
    this._a = val
    // 设置a属性时调用此函数
    cbWhenSetA(val)
  }
})
target.a = 3 // 我被设置为3
```
但是这显然离想要达到的效果相去甚远，我们首先希望数据变化时可以触发不止一个回调，其次希望不仅可以手动设置，也可以自动收集依赖。
### 多个回调
这个其实简单，只要将要调用的函数都放入一个数组，数据变动时依次调用即可。
```javascript
let target = {
  _a: ''
}
function cbWhenSetA(val) {
  console.log(`我被设置为${val}`)
}
function cbWhenSetA1(val) {
  console.log(`我被设置为${val}`)
}
let listeners = [
  cbWhenSetA,
  cbWhenSetA1
]
Object.defineProperty(target, 'a', {
  value: 'a',
  get () {
    return this._a
  },
  set(val) {
    this._a = val
    listeners.forEach(listener => listener())
  }
})
```
### 手动设置
接着上文，一个手动观察的函数也就很容易成形。
```javascript
// 其它代码接上面
function watchA(cb) {
  listeners.push(cb)
}
watchA((val) => console.log(`我是第三个回调函数`))
```
取消观察也同理。
### 自动收集依赖
收集依赖，意思是如果有函数访问了`target.a`，就自动将这个函数放入队列中，数据变动时依次触发即可。

很容易想到需要对`get`操作进行拦截。
```javascript
Object.defineProperty(target, 'a', {
  get () {
    // 需要在这里将访问此属性的函数放入队列中，问题是如何获取这个函数呢？
    listeners.push()
    return this._a
  }
  // set略过
}
```
这是我们需要访问`target.a`的函数
```javascript
function aPlusOne () {
  console.log(`现在a + 1 = ${target.a + 1}`)
}
```
为了在`get`操作中添加这个函数，又因为可能还会添加只添其它函数，所以我们需要对这个函数的引用
```javascript
let currentListener
Object.defineProperty(target, 'a', {
  get () {
    listeners.push(currentListener)
    return this._a
  }
  // set略过
}
// 然后这样使用
currentListener = aPlusOne
aPlusOne() // a + 1 = 2
// 然后修改
target.a = 3
// 就会触发aPlusOne
// a + 1 = 4
```
`Vue`中的响应式最基本的原理大概就是这样，在更深入之前规范一下在`Vue`中的概念和术语，我自己是这么理解的：
- 需要响应的函数，如`render`函数，叫做观察者`watcher`，由`Watch`类创建。
- 观察者访问到的属性，叫做依赖`dep`，由`Dep`类创建，此外`Dep`类有一个静态属性`target`放置着当前需要收集依赖的`watcher`。
- 一个依赖，即一个属性，一个`dep`，可能有多个`watcher`需要响应，叫做依赖的订阅者，`subs`。
- 一个观察者可能会依赖多个属性，`deps`。
- 被依赖的属性，如`this.a`，中的a，被访问时如果有观察者存在，将其放入`dep.subs`中。被设置时能通知观察者，叫做响应式的属性，`reactive`，用`defineReactive`来实现，同时会创建一个`dep`。
- 将一个对象的所有属性都变为响应式的，这个对象即是`observer`，由`Observer`类创建，中文嘛。。。好像也叫观察者。

那么`Vue`中一套响应式系统大概流程如下：

模板
```javascript
new Vue({
  template: '<div>{{ dateTime }}</div>',
  data: {
    dateTime: new Date()
  },
  created () {
    setInterval(() => {
      this.dateTime = new Date()
    })
  }
})
```
1. 将`this.data`传入`new Observer(this.data)`，对`data`每一个属性都进行`defineReactive`，将其变成响应式的，这里讨论`dateTime`。
2. `defineReactive(this.data, dateTime)`，创建一个依赖`dateTimeDep = new Dep()`，拦截对`dateTime`的访问，如果`dateTime`被访问时有观察者存在，将这个观察者放入`dep.subs`中。
3. 为`render`函数创建一个观察者`renderWatcher = new Watcher()`。
4. 调用`renderWatcher.get`，会将`Dep.target`设置为`renderWatcher`，同时调用`render`。
5. `render`在运行过程中访问了`dateTime`，因此`renderWatcher`会被放入`dateTimeDep.subs`中。
6. 定时器触发后会设置`dateTime`，所以`dateTimeDep`会依次通知`dateTimeDep.subs`中的`watcher`，也包括`renderWatcher`，即会触发`renderWatcher.update`，也就是重新运行`render`。
7. 对于计算属性的设置同理。
### defineReactive
这个函数的作用就是将一个属性的值变为可响应式的，如果这个属性被访问，且当前有一个`watcher`，它就会将这个`watcher`放到自己的订阅列表中。如果这个属性被设置，就会通知订阅列表中的`watcher`。
### Dep
`Dep`类，依赖，它有一个静态属性`target`用来保存当前`watcher`。
### Watcher
`Watcher`类，观察者，即我们在属性变动时要调用的函数，如`render`。只不过它接受更复杂的参数，比如属性名和组件实例，也做了很多额外的处理。在它的内部有一个`deps`数组，即它访问的所有响应式属性所生成的`Dep`实例。
### Observer
`Observer`类，观察者，将一个对象中的所有属性都变为响应式的。
### observe
这个函数的作用是观察一个对象。
### toggleObserving
这个函数的作用是切换在`observe`函数内是否进行观测的开关。即如果切换为`false`,调用`observe`也不会进行观测。

## 其它
当然其中还有很多复杂的处理，具体的请看其它文章。但是看到这里也足够我们对`vue`中的响应式有个初步的理解了，能更好地深入分析相关源码，看其它部分源码的时候遇到这些不会太过懵逼。