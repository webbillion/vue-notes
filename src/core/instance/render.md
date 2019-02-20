---
vue-note-completed: true
---
# core/instance/render.js

## renderMixin
为`Vue`的原型添加渲染相关的方法。主要有
### installRenderHelpers
添加很多辅助方法，主要是给编译模板使用的，点击查看[详情](./render-helpers/index.md)。
### $nextTick
#### 作用
在本次渲染完成后调用传入的回调。里面是直接调用了[nextTick]()。
### _render
#### 作用
渲染一个`vnode`并返回。
#### 过程
1. 定义一些变量：
   - `vm` 对自身的引用。
   - `render` 选项中的`render`，即渲染成一个真实的`dom`。
   - `_parentVnode` 在父组件渲染树中的`vnode`，在[这里]()被传入。
2. 如果`_parentVnode`存在，将`vm.$scopedSlots`定义为`_parentVnode.data.scopedSlots`或者`emptyObject`。这个属性是在父组件使用了使用`slot-scope`属性的内容。来源同上。
3. 设置`vm.$vnode`为`_parentVnode`，这是为了在渲染函数中能够访问。但是这里需要注意的是：
  ```html
  <!-- parent -->
  <div>
    <child></child>
  </div>
  ```
  > 渲染`child-1`时的`_parentVnode`，不是`parent`，也不是`div`，而是`child-1`在父组件渲染树中的`vnode（VNode {tag: "vue-component-1-child"})`，具体以后分析。
4. 定义`vnode`，保存渲染自己的结果。
5. `try` 调用`render.call(vm._renderProxy, vm.$createElement)`，关于`_renderProxy`，即渲染时作用域代理，生产环境下就是`vm`自己，详情可以看[proxy.js](./proxy.md)，而`$createElement`是作为`render`函数的参数的，如果看过文档，应该知道`redner`函数接受一个`h`作为创建元素的参数，像这样来使用`return h('div')`。`$createElement`就其这样的作用，它在[initRender](#initRender)中被定义，这儿简单认为它是一个能创建`dom`的函数即可。
6. `catch` 首先处理错误，然后将`vm._vnode`赋值给`vnode`。`_vnode`定义在`initRender`，初始值为`null`。更新的时候会重新赋值，这里的意思是保留之前渲染的`vnode`。如果是非生产环境的话，其中还要再次`try`调用`vm.$options.renderError`，这个函数的作用是将错误信息渲染成`vnode`，定义在[这里]()。最后都是为了防止出现空白的渲染结果。
7. 如果`vnode`不是`Vnode`的实例，为`vnode`赋值一个空的节点。这其中有两种可能，一是第一次渲染时就出错了，`_vnode`为`null`；二是在上面的`render`中返回了一个数组，也就是当前组件没有一个根节点，而`Vue`是不允许这么做的，所以非生产环境下会给出提示。
   - [ ] 为什么？个人认为，如果允许的话，首先需要一个节点对这些节点进行包装，然后要处理更多的边界情况，得不偿失。
8. 令`vnode.parent` = `_parentVnode`。
9. 返回`vnode`。
10. 完毕。
## initRender
### 作用
给`vm`实例初始化`render`相关的属性和方法，同时将`vm`的一些属性定义为响应式的。
### 过程
1. 在`vm`上定义很多属性，它们是：
   - `_vnode` 节点自身的`vnode`，初始值为`null`。
   - `_staticTrees` 静态`dom`树，使用`v-once`指令是会为它赋值来做到缓存，初始值为`null`。
2. 定义了一些变量引用方便在下面使用，分别是：
   - `options` `vm.$options`，即创建时的选项。
   - `parentVnode` `options_parentVnode`，在父组件渲染中的节点。
   - `renderContext` `parentVnode.context`，渲染上下文，详情看[这里]()。
   - `parentData` `parentVnode.data` 由父节点传递的数据，里面包含`attr`等。
3. 然后继续在`vm`上定义属性：
   - `$slots` `slot`的`vnode`，由[resolveSlots]()解析得到。
   - `$sopedSlots` 父组件中用了`slot-scope`属性的`slot`，里面是一些函数，初始值为空对象。
   - `_c` 创建元素，它是给从模板编译而来的`render`函数使用的，当然手写`render`函数也能调用。
   - `$createElement` 创建元素，它是作为参数传递给`render`函数的。
    > 这两个属性的初始值都是将`vm`和传入的参数传递给[createElement](../vdom/create-element.md)并调用的函数，只不过第五个参数`_c`是`false`，`$createElement`是`true`，这个参数叫`alwaysNormalize`，用来决定如何渲染子组件。具体请看对应分析。
4. 下面是就是将`$attrs`和`$listeners`定义为响应式的，且值来源于`parentData.attrs`和`options._parentListeners`或者空对象。而且不对它们的属性值做监测。即修改`vm.$attrs.attr`不会触发响应。此外在非生产环境时加入只读警告。
> 这两个属性我们一般是用不到的，之所以暴露这个两个属性，是为了更容易地创建高阶组件。也就是说高阶组件内用到这两个属性时，如果它发生了变化，可以触发更新。关于高阶组件和这一段的关系
>  - [ ] 有兴趣的可以看[这里]()。
5. 完毕。
## 代码
```javascript
/* @flow */

import {
  warn,
  nextTick,
  emptyObject,
  handleError,
  defineReactive
} from '../util/index'

import { createElement } from '../vdom/create-element'
import { installRenderHelpers } from './render-helpers/index'
import { resolveSlots } from './render-helpers/resolve-slots'
import VNode, { createEmptyVNode } from '../vdom/vnode'

import { isUpdatingChildComponent } from './lifecycle'

export function initRender (vm: Component) {
  vm._vnode = null // the root of the child tree
  vm._staticTrees = null // v-once cached trees
  const options = vm.$options
  const parentVnode = vm.$vnode = options._parentVnode // the placeholder node in parent tree
  const renderContext = parentVnode && parentVnode.context
  vm.$slots = resolveSlots(options._renderChildren, renderContext)
  vm.$scopedSlots = emptyObject
  // bind the createElement fn to this instance
  // so that we get proper render context inside it.
  // args order: tag, data, children, normalizationType, alwaysNormalize
  // internal version is used by render functions compiled from templates
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
  // normalization is always applied for the public version, used in
  // user-written render functions.
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)

  // $attrs & $listeners are exposed for easier HOC creation.
  // they need to be reactive so that HOCs using them are always updated
  const parentData = parentVnode && parentVnode.data

  /* istanbul ignore else */
  if (process.env.NODE_ENV !== 'production') {
    defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, () => {
      !isUpdatingChildComponent && warn(`$attrs is readonly.`, vm)
    }, true)
    defineReactive(vm, '$listeners', options._parentListeners || emptyObject, () => {
      !isUpdatingChildComponent && warn(`$listeners is readonly.`, vm)
    }, true)
  } else {
    defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, null, true)
    defineReactive(vm, '$listeners', options._parentListeners || emptyObject, null, true)
  }
}

export function renderMixin (Vue: Class<Component>) {
  // install runtime convenience helpers
  installRenderHelpers(Vue.prototype)

  Vue.prototype.$nextTick = function (fn: Function) {
    return nextTick(fn, this)
  }

  Vue.prototype._render = function (): VNode {
    const vm: Component = this
    const { render, _parentVnode } = vm.$options

    if (_parentVnode) {
      vm.$scopedSlots = _parentVnode.data.scopedSlots || emptyObject
    }

    // set parent vnode. this allows render functions to have access
    // to the data on the placeholder node.
    vm.$vnode = _parentVnode
    // render self
    let vnode
    try {
      vnode = render.call(vm._renderProxy, vm.$createElement)
    } catch (e) {
      handleError(e, vm, `render`)
      // return error render result,
      // or previous vnode to prevent render error causing blank component
      /* istanbul ignore else */
      if (process.env.NODE_ENV !== 'production' && vm.$options.renderError) {
        try {
          vnode = vm.$options.renderError.call(vm._renderProxy, vm.$createElement, e)
        } catch (e) {
          handleError(e, vm, `renderError`)
          vnode = vm._vnode
        }
      } else {
        vnode = vm._vnode
      }
    }
    // return empty vnode in case the render function errored out
    if (!(vnode instanceof VNode)) {
      if (process.env.NODE_ENV !== 'production' && Array.isArray(vnode)) {
        warn(
          'Multiple root nodes returned from render function. Render function ' +
          'should return a single root node.',
          vm
        )
      }
      vnode = createEmptyVNode()
    }
    // set parent
    vnode.parent = _parentVnode
    return vnode
  }
}

```