---
vue-note-completed: true
---
# core/vdom/vnode.js
阅读本篇之前可简单看看[vnode概述](./README.md)。
## VNode
### 作用
定义并导出了`Vnode`类。
### 属性
- `tag` 节点标签名
- `data` 节点数据，它包含了`props` `on` `attrs`等属性，详情可看[这里]()。
- `children` 子节点组成的数组。
- `text` 节点文本，在节点是文本节点（包括普通文本和注释）是起作用。
- `elm` 关联的元素，真实的`DOM` 节点。
- `ns` 标签的命名空间，目前已知有两个，一个是`svg`，一个是`math`，即普通元素节点是没有这个属性的。
- `context` 节点上下文，通常是一个组件，指定它的作用域。
- `key` 就是我们在`v-for`中使用的`key`，用来决定是更新还是新增`DOM`节点。
- `componentOptions` 节点的组件选项，主要包含了`propsData`和`children`等属性。
- `componentInstance` 组件实例，在[createComponentInstanceForVnode](./create-component.md)中被赋值。
- `parent` 父节点，不过：
```html
<parent>
  <child></child>
</parent>
```
> `child`组件本身渲染出的`vnode`的`parent`，并不等于`parent`的`vnode`，而是`child`在父组件渲染树中的`vnode`。
--- 
以下是`strictly internal`的属性，外部用不上的属性？
- `raw` 是否包含原生`html`，仅服务端为`true`。
- - [ ] 这一段存疑。
- `isStatic` 是否被悬挂为静态的节点，它在[markStaticNode]。(../instance/render-helpers/render-static.md#markStaticNode)中被重写。
- `isRootInsert` 在[transition]()组件中`enter`过程检查时使用，在[createElm]()中被赋值。
- `isComment` 是否是注释节点。
- `isCloned` 是否是克隆而来的节点，见下文。
- `asyncFactory` 异步组件的工厂函数，使用异步组件时会用到。
- `asyncMeta` 使用异步组件时的一些选项。
- `isAsyncPlaceholder` 是否是异步组件加载时展示的节点。
- `ssrContext` 服务端渲染后给出的上下文
- - [ ] 用于在浏览器中衔接。
- `fnContext` 函数上下文，函数式节点真正的上下文，一个组件实例。关于函数式节点，参考[create-functional-componen.js](./create-functional-component.md)。
- `fnOptions` 函数选项，包含`propsData`和`children`等，用于`ssr`缓存。
- `devtoolsMeta` 开发工具信息，同来在`devtool`中存储函数式上下文。
- `fnScopedId` 带`slot-soped`的`slot`会被编译成函数。这个`id`即是对应`slot`的名称。
### constructor
将接受的参数赋值给对应属性，同时初始化其它属性。
#### 参数
- `tag`
- `data`
- `children`
- `text`
- `elm`
- `context`
- `componentOptions`
- `asyncFactory`
#### 过程
赋值及初始化，除了特别说明，将参数直接赋值，其它的都初始化为`undefined`或者`false`。
- `key` = `data.key`
- `isRootInsert` = `true`
### get child
这个函数要弃用了，不再说明。
## createdEmptyVNode
简单的封装，创建一个注释节点当空节点。接受`text`文本参数。
## createTextVNode
同上，创建一个文本节点，这里将接受的参数用`String`转换。另外就是这两个封装调用方式不同，不知道是为什么。
## cloneVNode
### 作用
克隆一个节点，主要用于静态节点和`slot`节点可能会在多个地方使用。浅克隆，即不克隆子节点。
### 过程
1. 首先将构造函数需要的参数都传进去创建一个新的`VNode`，其中`children`参数需要使用`children.slice`，即虽然新节点的`children`和传入节点的`children`虽然内容相同，但引用不同。这里的[#7975](https://github.com/vuejs/vue/issues/7975)是因为新的克隆策略，取消了深度克隆的选项，简单来说，如果克隆了新一个节点，然后摧毁原节点，因为它们有相同的子组件引用，会导致新节点没有子节点了。
   - [ ] 在哪里摧毁的，又如何摧毁的？暂时还不知道。好像只有`originNode.children.length = 0`才会这样吧？
2. 然后对属性赋值，它们是
   - `ns`
   - `isStatic`
   - `key`
   - `isComment`
   - `fnContext`
   - `fnOptions`
   - `fnScopeId`
   - `asyncMeta`
   - [ ] 为什么只对这些属性赋值？应该是因为其它属性和它所在的渲染树有关？

---
看到这里，有点不知道该如何继续下去了，回想整个过程，重点应该在`render`和`__patch__`上。`render`从何而来？即两个关键的问题，`vnode`在哪里创建，它又是如何转换成真实`dom`的。不对，应该是先看`createElment`。