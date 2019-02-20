---
vue-note-completed: true
---
# core/instance/lifecycle.js
## activeInstance
当前正在更新的实例。被`setActiveInstance`改变。
## isUpdatingChildComponent
是否正在更新子组件，在`updateChildComponent`中被改变。这个属性用来在开发时给出一些提示。
## lifecycleMixin
为`Vue`原型添加一系列生命周期相关方法，它们是
### _update
#### 作用
顾名思义，更新。
#### 参数
- `vnode` 新的`Vnode`，即虚拟`dom`。
- `hydrating` 布尔值，可选，是否是。。吸水？
   - [ ] 暂不清楚
####  过程
1. 定义一系列变量：
   - `vm` `this`，实例自身。
   - `prevEl` `vm.$el`，实例之前的元素引用。
   - `prevVnode` `vm._vonde` 实例之前的`vnode`
   - `restoreActiveInstance` 调用[setActiveInstance](#setActiveInstance)将自身置为当前正在更新的实例，返回一个能恢复之前实例的函数。
2. 将`vm._vode`置为新的`vnode`。
3. 如果实例之前没有`vnode`，即第一次渲染，调用`vm.__patch__`传入第一次渲染时应该传入的参数，否则比较两个`vnode`来生产元素。`vm.__patch__`用来渲染`vnode`成`dom`，或者比较两个`vnode`返回新的`dom`，详情参考[这里](../vdom/patch.js)。需要**注意**的是，这个`__patch__`方法是由运行时环境给`Vue`的原型添加的，并不在核心代码(core)内。
     - [ ] 为什么，`vdom`不是在核心代码里吗？
4. 当前实例更新完毕，调用`restoreActiveInstance`重置。
5. 接着更新元素上`__vue__`属性，即元素对实例的引用。
6. 如果父组件是高阶组件(HOC，不了解的可以搜索一下)，将父组件的`$el`也置为刚才生成的`dom`，这是因为高阶组件是没有自己的元素的。如何判断是高阶组件呢？即`vm.$parent._vnode === vm.$vnode`，即父组件的`_vnode`和当前实例的`$vnode`相同，`$vnode`属性是组件在父组件渲染中的`vnode`，在[_render](./init.md)时定义，如果父组件是高阶组件的话，这两者应该是相等的。
7. 这里更新了，但是却没由看到更新相关的钩子调用，因为它被放到`scheduler`中了，来确保父组件触发钩子时子组件已经更新完毕了。它的作用以及是如何确保的请参考[这里]()。
8. 完毕。
### $forceUpdate
#### 作用
强制更新，调用`renderWatcher`，重新渲染。
### $destory
#### 作用
摧毁当前实例。
#### 过程
1. 如果`vm._isBeingDestoryed`，即正在摧毁，直接返回，这个属性在下面设置。可能是为了防止摧毁时多次调用此函数吧，但是这个函数也不是，应该不是，异步的，暂时不清楚什么情况会多次调用。
2. 触发`befreDestory`钩子。设置`_isBeingDestoryed`为`true`。
3. 如果父组件存在且它不是正在摧毁中，且它不是一个抽象组件，将自己从父组件中移除。
4. 取消`renderWachter`和其它`watcher`的监听。
5. 如果`_data`有观察者，将它的`vmCount`减1.
6. 设置`_isDestoryed`为`true`，即已摧毁。
7. 调用`__patch__`将`vnode`置空。
8. 触发`destoryed`钩子。
9. 移除所有事件监听。
10. 移除`vm.$el`对实例的引用。
11. 移除`vm.$vnode.parent`，为了避免循环引用造成内存泄露。它来自[issue#6759]()，感兴趣可以点击[issues/issue-6759](issues/issue-6759.md)查看详情。
## initLifecycle
### 作用
初始化生命周期相关。主要是定义一些属性。
### 过程
1. 获取选项`options`并定义`parent`为选项中的父组件。
2. 如果有父组件且本身不是抽象组件，一直向上查找，直到父组件不是抽象组件为止，即不是`keep-alive`这些。然后将`vm`放入父组件的`$children`中。
3. 在`vm`上定义以下属性：
   - `$parent` 父组件，即刚刚找到的`parent`。
   - `$root` 根组件，如果有父组件，取父组件的`$root`，否则为自己。
   - `$children` 子组件，初始化为空数组。
   - `$refs` 对`dom`的引用，初始化为空对象，我们常用的`this.$refs`就来自这里。
   - `_watcher` `renderWatcher`的引用，初始化为`null`。
   - `_inactive` 处于未激活状态，初始化为`null`，我现在是这么理解的,目前是给`vue-router`用的。
   - `_directInactive` 直接未激活，初始化为`false`
     - [ ] 我猜是它是不是`keep-alive`的直接子组件。
   - `isMouted` 是否已挂载，初始化为`false`，在初次渲染后变为`true`。
   - `_isDestoryed` 是否已摧毁，初始化为`false`，调用`$destory`后变为`true`。
   - `_isBeingDestoryed` 是否正在摧毁，初始化为`false`，在摧毁过程中变化。
## setActiveInstance
### 作用
设置当前正在更新的实例`activeInstance`，在实例更新完成后将其恢复为上一个值。
## updateChildComponent
### 作用
更新子组件
### 参数
- `vm` 要更新的子组件
- `propsData` 传递给子组件的`props`数据。
- `listeners` 传递给子组件的监听事件。
- `parentVnode` 已经挂载了的父组件的`vnode`。
- `renderChildren` 子组件的子组件`vnode`数组。
### 过程
1. 非生产环境设置正在更新子组件`isUpdatingChildComponent`为`true`。
2. 接着定义`hasChildren`，有没有`slot`子组件，如何判定呢？满足以下条件之一即可：
   - `renderChildren` 有静态的新的`slot`子组件。
   - `vm.$options._renderChildren` 原来有`slot`子组件。
   - `parentVnode.data.scopedSlots` 父组件有新的`scoped`内容。
   - `vm.$sopedSlots` 不为空对象，即之前有`scoped`。
3. 这四个属性究竟是什么，下面做个简单的介绍，详细内容可翻阅对应章节。`hasChildren`这个属性在本函数的最后面才会用到。
```javascript
let Child = {
	name: 'child',
  template: '<div><span>{{ localMsg }}</span><button @click="change">click</button><slot name="fd"><div>这是一个默认slot</div></slot><slot name="fd1" :props="localMsg"></slot></div>',
  data: function() {
  	return {
    	localMsg: this.msg
    }
  },
  props: {
  	msg: String
  },
  methods: {
  	change() {
    	this.$emit('update:msg', this.msg + 'world')
    }
  }
}

new Vue({
	el: '#app',
  template: '<div><child :msg.sync="msg"><p slot="fd">新的slot</p><p slot="fd1" slot-scope="props">带scope的slot</p></child></div>',
  beforeUpdate() {
  	alert('update twice')
  },
  data() {
  	return {
    	msg: 'hello'
    }
  },
  components: {
  	Child
  }
})
// 以上面的代码为例，在更新Child组件时，上述四个值分别是
// [Vnode]，即<p slot="fd">新的slot</p>的vnode，下同
// [Vnode]
// {fd1: function}  一个函数，下同
// {fd1: function}
```
4. 接着更新三个属性为`parentVnode`，它们是：
   - `vm.$options._parentVnode` `vm`选项中的父节点。
   - `vm.$vode` 当前实例在渲染树中的`vnode`。
   - `vm._vnode.parent` 当前实例的`vnode`的父节点。
5. 将选项中的`_renderchildren`置为`renderChildren`。
6. 从父节点中获取属性`attrs`，并设置监听事件`listeners`。
7. 更新`props`和`propsData`，这一部分的内容参考[initProps](./state.md#initProps)。值得注意的是，这里遍历`props`的`key`时，是遍历的`_propsKeys`数组，这和[前面](./state.md#initProps)相呼应。另外在`2.5.21`版本写这段代码的人似乎被`flow`给为难了，忍不住抱怨了一句`wtf flow?`。
8. 然后更新`listeners`，这一部分内容参考[initEvents](./events.md/initEvents)。
9. 如果`hasChildren`为真，即有`slot`子组件，令`vm.$slots` = `resolveSlots()`，这个函数的作用是解析新的`slots`，详情点击[这里]()。然后强制更新——因为`slots`属性不是响应式的。
10. 非生产环境设置正在更新子组件`isUpdatingChildComponent`为`false`。
11. 完毕。
## mountComponent
### 作用
挂载组件到指定元素上。
### 参数
- `vm` 要挂载的组件
- `el` 指定的元素
- `hydrating` 额，，
### 过程
1. 令`vm.$el`等于`el`。
2. 如果`vum.$options`中没有`render`，对其赋值`createEmptyVNode`，一个可以创建空节点的函数。
3. 没有`render`，在非生产环境下给出对应警告。
4. 调用`beforeMout`钩子。
5. 定义`updateComponent`，根据生产环境与否进行赋值，主要不同是在生产环境下开启了性能记录。参考[_init](./init.md)。但都会做一个工作，就是`vm._update(vm._render(), hydrating)`。它的作用，参考上文。
6. 为`updateComponent`创建一个`watcher`，并传入`before`选项，用来触发`beforeUpdate`，同时传入第五个参数，表明这是一个`renderWachter`。这里有一段注释，大意是说在`Watcher`的构造函数里我们初始化了`vm._watcher`，以便组件在`mouted`生命周期后能够调用`$forceUpdate`。
7. 将`hydrating`设为`false`。
8. 如果实例没有`$vnode`，即它在是顶层元素，设置`vm._isMouted`为`true`，触发`mounted`生命周期。
    > 需要注意的是，如果它有`$vnode`，它的`mounted`又是在哪里被触发的呢？答案是子组件渲染时的`vnode`的钩子函数中的`insert`中触发的，详情参考[create-component#insert](../vdom/create-component.md)。至于为什么这么做，根组件是需要我们手动挂载的，子组件由根组件来挂载，所以让子组件自己来调用这个钩子。
    - [ ] 还是不太明白。
9.  返回`vm`。
10. 完毕。
## isInInactiveTree
判断组件是否在未激活的组件树中。遍历其上层组件，如果有一个组件的`_inactive`为真，返回`true`，否则返回`false`。
## activateChildComponent
### 作用
激活子组件。
### 参数
- `vm` 要激活的组件
- `direct` 直接？
### 过程
1. 如果`direct`为真，事实上在看到的源码部分，它总是为真：
   1. 将`vm._directInactive`重置为`false`。
   2. 如果上层组件处于未激活状态，则什么也不做直接返回。
2. 如果`vm._directInactive`为真，什么也不做，直接返回。
3. 如果`vm._inactive`为真，或者为`null`，即初始化时候的值。
   1. 将`vm._inactive`设为`false`。
   2. 对子组件依次调用本函数。
   3. 调用`activated`钩子。
4. 完毕。
## deactivateChildComponent
### 作用
组件在`keep-alive`下不直接摧毁，而是让子组件处于未激活状态。
### 参数
- `vm` 要激活的组件
- `direct` 直接？
### 过程
1. 如果`direct`为，真事实上在看到的源码部分，它总是为真：
   1. 将`vm._directInactive`重置为`true`。
   2. 如果上层组件处于未激活状态，则什么也不做直接返回。
2. 如果组件处于未激活状态，设置`vm._inactive` 为`true`。
3. 对子组件依次调用本函数。
4. 调用`deactivated`钩子。
5. 完毕。
>  - [ ] 它们在[create-component#hook]()中被调用，但是仍然不清楚`direct`参数有什么用。
> 梳理一下整个过程。
> 1. 在节点`insert`时
>    1. `direct`为真，将`vm._directInactive`设为`false`。
>    2. 然后`isInInactiveTree(vm)`也返回`false`，可以进入下一步。
>    3. `vm._inactive`为`null`，所以可以进入下一步。
>    4. 设为`false`，触发`activated`。
> 2. 在节点`destroy`时
>    1. `direct`为真，将`vm._directInactive`设为`true`。
>    2. 然后`isInInactiveTree(vm)`也返回`false`，可以进入下一步。
>    3. `vm._inactive`为`false`，所以可以进入下一步。
>    4. 设为`true`，触发`deactivated`。
> 3. `direct`的主要作用就是为`true`的话会判断上层组件是否激活，即自身的激活状态和上层组件的激活状态相关，否则不相关。但是一直都是传的`true`，所以还是不太明白。
### 参数
## callHook
### 作用
调用生命周期函数，触发生命周期事件。
### 过程
这个过程比较简单，就是遍历选项中指定生命周期的处理函数依次调用，并放在`try catch`中。如果这个组件被监听了生命周期相关的事件，调用`$emit`触发相应事件，参考[events.js](./events.md)。关于`pushTarget`，它来自[issue#7573](https://github.com/vuejs/vue/issues/7573)，有兴趣可以参考[issues/issue-7573](/issues/issue-7573.md)。
## 代码
```javascript
/* @flow */

import config from '../config'
import Watcher from '../observer/watcher'
import { mark, measure } from '../util/perf'
import { createEmptyVNode } from '../vdom/vnode'
import { updateComponentListeners } from './events'
import { resolveSlots } from './render-helpers/resolve-slots'
import { toggleObserving } from '../observer/index'
import { pushTarget, popTarget } from '../observer/dep'

import {
  warn,
  noop,
  remove,
  handleError,
  emptyObject,
  validateProp
} from '../util/index'

export let activeInstance: any = null
export let isUpdatingChildComponent: boolean = false

export function setActiveInstance(vm: Component) {
  const prevActiveInstance = activeInstance
  activeInstance = vm
  return () => {
    activeInstance = prevActiveInstance
  }
}

export function initLifecycle (vm: Component) {
  const options = vm.$options

  // locate first non-abstract parent
  let parent = options.parent
  if (parent && !options.abstract) {
    while (parent.$options.abstract && parent.$parent) {
      parent = parent.$parent
    }
    parent.$children.push(vm)
  }

  vm.$parent = parent
  vm.$root = parent ? parent.$root : vm

  vm.$children = []
  vm.$refs = {}

  vm._watcher = null
  vm._inactive = null
  vm._directInactive = false
  vm._isMounted = false
  vm._isDestroyed = false
  vm._isBeingDestroyed = false
}

export function lifecycleMixin (Vue: Class<Component>) {
  Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    const restoreActiveInstance = setActiveInstance(vm)
    vm._vnode = vnode
    // Vue.prototype.__patch__ is injected in entry points
    // based on the rendering backend used.
    if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    restoreActiveInstance()
    // update __vue__ reference
    if (prevEl) {
      prevEl.__vue__ = null
    }
    if (vm.$el) {
      vm.$el.__vue__ = vm
    }
    // if parent is an HOC, update its $el as well
    if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
      vm.$parent.$el = vm.$el
    }
    // updated hook is called by the scheduler to ensure that children are
    // updated in a parent's updated hook.
  }

  Vue.prototype.$forceUpdate = function () {
    const vm: Component = this
    if (vm._watcher) {
      vm._watcher.update()
    }
  }

  Vue.prototype.$destroy = function () {
    const vm: Component = this
    if (vm._isBeingDestroyed) {
      return
    }
    callHook(vm, 'beforeDestroy')
    vm._isBeingDestroyed = true
    // remove self from parent
    const parent = vm.$parent
    if (parent && !parent._isBeingDestroyed && !vm.$options.abstract) {
      remove(parent.$children, vm)
    }
    // teardown watchers
    if (vm._watcher) {
      vm._watcher.teardown()
    }
    let i = vm._watchers.length
    while (i--) {
      vm._watchers[i].teardown()
    }
    // remove reference from data ob
    // frozen object may not have observer.
    if (vm._data.__ob__) {
      vm._data.__ob__.vmCount--
    }
    // call the last hook...
    vm._isDestroyed = true
    // invoke destroy hooks on current rendered tree
    vm.__patch__(vm._vnode, null)
    // fire destroyed hook
    callHook(vm, 'destroyed')
    // turn off all instance listeners.
    vm.$off()
    // remove __vue__ reference
    if (vm.$el) {
      vm.$el.__vue__ = null
    }
    // release circular reference (#6759)
    if (vm.$vnode) {
      vm.$vnode.parent = null
    }
  }
}

export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el
  if (!vm.$options.render) {
    vm.$options.render = createEmptyVNode
    if (process.env.NODE_ENV !== 'production') {
      /* istanbul ignore if */
      if ((vm.$options.template && vm.$options.template.charAt(0) !== '#') ||
        vm.$options.el || el) {
        warn(
          'You are using the runtime-only build of Vue where the template ' +
          'compiler is not available. Either pre-compile the templates into ' +
          'render functions, or use the compiler-included build.',
          vm
        )
      } else {
        warn(
          'Failed to mount component: template or render function not defined.',
          vm
        )
      }
    }
  }
  callHook(vm, 'beforeMount')

  let updateComponent
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    updateComponent = () => {
      const name = vm._name
      const id = vm._uid
      const startTag = `vue-perf-start:${id}`
      const endTag = `vue-perf-end:${id}`

      mark(startTag)
      const vnode = vm._render()
      mark(endTag)
      measure(`vue ${name} render`, startTag, endTag)

      mark(startTag)
      vm._update(vnode, hydrating)
      mark(endTag)
      measure(`vue ${name} patch`, startTag, endTag)
    }
  } else {
    updateComponent = () => {
      vm._update(vm._render(), hydrating)
    }
  }

  // we set this to vm._watcher inside the watcher's constructor
  // since the watcher's initial patch may call $forceUpdate (e.g. inside child
  // component's mounted hook), which relies on vm._watcher being already defined
  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  hydrating = false

  // manually mounted instance, call mounted on self
  // mounted is called for render-created child components in its inserted hook
  if (vm.$vnode == null) {
    vm._isMounted = true
    callHook(vm, 'mounted')
  }
  return vm
}

export function updateChildComponent (
  vm: Component,
  propsData: ?Object,
  listeners: ?Object,
  parentVnode: MountedComponentVNode,
  renderChildren: ?Array<VNode>
) {
  if (process.env.NODE_ENV !== 'production') {
    isUpdatingChildComponent = true
  }

  // determine whether component has slot children
  // we need to do this before overwriting $options._renderChildren
  const hasChildren = !!(
    renderChildren ||               // has new static slots
    vm.$options._renderChildren ||  // has old static slots
    parentVnode.data.scopedSlots || // has new scoped slots
    vm.$scopedSlots !== emptyObject // has old scoped slots
  )

  vm.$options._parentVnode = parentVnode
  vm.$vnode = parentVnode // update vm's placeholder node without re-render

  if (vm._vnode) { // update child tree's parent
    vm._vnode.parent = parentVnode
  }
  vm.$options._renderChildren = renderChildren

  // update $attrs and $listeners hash
  // these are also reactive so they may trigger child update if the child
  // used them during render
  vm.$attrs = parentVnode.data.attrs || emptyObject
  vm.$listeners = listeners || emptyObject

  // update props
  if (propsData && vm.$options.props) {
    toggleObserving(false)
    const props = vm._props
    const propKeys = vm.$options._propKeys || []
    for (let i = 0; i < propKeys.length; i++) {
      const key = propKeys[i]
      const propOptions: any = vm.$options.props // wtf flow?
      props[key] = validateProp(key, propOptions, propsData, vm)
    }
    toggleObserving(true)
    // keep a copy of raw propsData
    vm.$options.propsData = propsData
  }

  // update listeners
  listeners = listeners || emptyObject
  const oldListeners = vm.$options._parentListeners
  vm.$options._parentListeners = listeners
  updateComponentListeners(vm, listeners, oldListeners)

  // resolve slots + force update if has children
  if (hasChildren) {
    vm.$slots = resolveSlots(renderChildren, parentVnode.context)
    vm.$forceUpdate()
  }

  if (process.env.NODE_ENV !== 'production') {
    isUpdatingChildComponent = false
  }
}

function isInInactiveTree (vm) {
  while (vm && (vm = vm.$parent)) {
    if (vm._inactive) return true
  }
  return false
}

export function activateChildComponent (vm: Component, direct?: boolean) {
  if (direct) {
    vm._directInactive = false
    if (isInInactiveTree(vm)) {
      return
    }
  } else if (vm._directInactive) {
    return
  }
  if (vm._inactive || vm._inactive === null) {
    vm._inactive = false
    for (let i = 0; i < vm.$children.length; i++) {
      activateChildComponent(vm.$children[i])
    }
    callHook(vm, 'activated')
  }
}

export function deactivateChildComponent (vm: Component, direct?: boolean) {
  if (direct) {
    vm._directInactive = true
    if (isInInactiveTree(vm)) {
      return
    }
  }
  if (!vm._inactive) {
    vm._inactive = true
    for (let i = 0; i < vm.$children.length; i++) {
      deactivateChildComponent(vm.$children[i])
    }
    callHook(vm, 'deactivated')
  }
}

export function callHook (vm: Component, hook: string) {
  // #7573 disable dep collection when invoking lifecycle hooks
  pushTarget()
  const handlers = vm.$options[hook]
  if (handlers) {
    for (let i = 0, j = handlers.length; i < j; i++) {
      try {
        handlers[i].call(vm)
      } catch (e) {
        handleError(e, vm, `${hook} hook`)
      }
    }
  }
  if (vm._hasHookEvent) {
    vm.$emit('hook:' + hook)
  }
  popTarget()
}

```