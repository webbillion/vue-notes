---
vue-note-completed: true
---
# core/instance/events.js
## eventsMixin
为`Vue`的原型添加事件相关的方法。分别是：
- `$on`
- `$once`
- `$off`
- `$emit`

就是一套常见的事件处理机制，这里不再详解，对一些特殊的地方进行说明。
### $on
```javascript
const hookRE = /^hook:/
// ...
// 如果事件名以hook:开头，则添加标识，做性能优化
if (hookRE.test(event)) {
  vm._hasHookEvent = true
}
```
这个标识在[callHook](./lifecycle)中被使用，在生命周期函数被触发的同时，如果父组件做了对`@hook:`的监听，才会触发这一事件。
### $emit
```javascript
if (process.env.NODE_ENV !== 'production') {
  const lowerCaseEvent = event.toLowerCase()
  // 注册的时候是小写的事件名，调用的时候是大写，给出提示
  if (lowerCaseEvent !== event && vm._events[lowerCaseEvent]) {
    tip()
  }
}
// ...
// 然后是在调用函数的时候进行一个错误捕捉
try {
  cbs[i].apply(vm, args)
} catch (e) {
  handleError(e, vm, `event handler for "${event}"`)
}
```
## initEvents
###  目的
在组件实例化的时候处理父组件的事件监听。
### 过程
1. 将`vm._events`置为完全的空对象。
2. 添加`_hasHookEvent`标识为`false`，这个属性可能在`$on`方法中被赋值。
3. 获取选项中父组件的监听事件`_parentListeners`，并调用[updateComponentListeners](#updateComponentListeners)注册这些事件。举例如下：
```javascript
Test = Vue.extend({
  template: '<div></div>'
  })
  Vue.component('test', Test)
  new Vue({
    el: '#app',
    template: '<test @click="handleClick"></test>',
    methods: {
        handleClick() {
        }
    }
  })
// 则在实例化Test时，这个_parentListeners等于
{
  click: hanldeClick() {}
}
```
4. 调用注册这些事件。
5. 完毕。
### updateComponentListeners
#### 作用
更新或注册组件的监听事件。
#### 参数
- `vm` 要监听的组件实例。
- `listeners` 监听事件的名称和其处理函数
- `oldListeners` 更新事件的时候会重新注册，同时将旧的事件移除。
#### 过程
1. 对`target`赋值为`vm`，这是为了在接下来的`createOnceHandler`函数中使用。
2. 调用[updateListeners](../vdom/helpers/update-listeners.md)进行注册。关于此函数以及`add` `remove`的作用都可以在这里面查看。简单来说就是它实际上会遍历事件，然后调用`vm.$on`将这些事件进行注册。
3. 将`target`置为`undefined`。
4. 完毕。
### createOnceHandler
它创建了一个`once`函数的`hanlder`，和调用`once`不同的是，它不负责监听事件，只是在触发时取消监听。监听由`add`负责。
## 代码
```javascript
/* @flow */

import {
  tip,
  toArray,
  hyphenate,
  handleError,
  formatComponentName
} from '../util/index'
import { updateListeners } from '../vdom/helpers/index'

export function initEvents (vm: Component) {
  vm._events = Object.create(null)
  vm._hasHookEvent = false
  // init parent attached events
  const listeners = vm.$options._parentListeners
  if (listeners) {
    updateComponentListeners(vm, listeners)
  }
}

let target: any

function add (event, fn) {
  target.$on(event, fn)
}

function remove (event, fn) {
  target.$off(event, fn)
}

function createOnceHandler (event, fn) {
  const _target = target
  return function onceHandler () {
    const res = fn.apply(null, arguments)
    if (res !== null) {
      _target.$off(event, onceHandler)
    }
  }
}

export function updateComponentListeners (
  vm: Component,
  listeners: Object,
  oldListeners: ?Object
) {
  target = vm
  updateListeners(listeners, oldListeners || {}, add, remove, createOnceHandler, vm)
  target = undefined
}

export function eventsMixin (Vue: Class<Component>) {
  const hookRE = /^hook:/
  Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {
    const vm: Component = this
    if (Array.isArray(event)) {
      for (let i = 0, l = event.length; i < l; i++) {
        vm.$on(event[i], fn)
      }
    } else {
      (vm._events[event] || (vm._events[event] = [])).push(fn)
      // optimize hook:event cost by using a boolean flag marked at registration
      // instead of a hash lookup
      if (hookRE.test(event)) {
        vm._hasHookEvent = true
      }
    }
    return vm
  }

  Vue.prototype.$once = function (event: string, fn: Function): Component {
    const vm: Component = this
    function on () {
      vm.$off(event, on)
      fn.apply(vm, arguments)
    }
    on.fn = fn
    vm.$on(event, on)
    return vm
  }

  Vue.prototype.$off = function (event?: string | Array<string>, fn?: Function): Component {
    const vm: Component = this
    // all
    if (!arguments.length) {
      vm._events = Object.create(null)
      return vm
    }
    // array of events
    if (Array.isArray(event)) {
      for (let i = 0, l = event.length; i < l; i++) {
        vm.$off(event[i], fn)
      }
      return vm
    }
    // specific event
    const cbs = vm._events[event]
    if (!cbs) {
      return vm
    }
    if (!fn) {
      vm._events[event] = null
      return vm
    }
    // specific handler
    let cb
    let i = cbs.length
    while (i--) {
      cb = cbs[i]
      if (cb === fn || cb.fn === fn) {
        cbs.splice(i, 1)
        break
      }
    }
    return vm
  }

  Vue.prototype.$emit = function (event: string): Component {
    const vm: Component = this
    if (process.env.NODE_ENV !== 'production') {
      const lowerCaseEvent = event.toLowerCase()
      if (lowerCaseEvent !== event && vm._events[lowerCaseEvent]) {
        tip(
          `Event "${lowerCaseEvent}" is emitted in component ` +
          `${formatComponentName(vm)} but the handler is registered for "${event}". ` +
          `Note that HTML attributes are case-insensitive and you cannot use ` +
          `v-on to listen to camelCase events when using in-DOM templates. ` +
          `You should probably use "${hyphenate(event)}" instead of "${event}".`
        )
      }
    }
    let cbs = vm._events[event]
    if (cbs) {
      cbs = cbs.length > 1 ? toArray(cbs) : cbs
      const args = toArray(arguments, 1)
      for (let i = 0, l = cbs.length; i < l; i++) {
        try {
          cbs[i].apply(vm, args)
        } catch (e) {
          handleError(e, vm, `event handler for "${event}"`)
        }
      }
    }
    return vm
  }
}

```