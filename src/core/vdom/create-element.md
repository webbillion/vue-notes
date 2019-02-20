---
vue-note-completed: true
---
# core/vdom/create-element.js
## createElement
它是对下面`_createElement`的包装，提供了更灵活的接口，即传参方式更灵活一些。不做多的介绍。
## _createElement
### 作用
创建一个或多个`VNode`节点
### 参数
- `context` 上下文，就是一个组件，回到[initRender](../instance/render.md)，可以看到是将实例`vm`作为第一个参数绑定了的。
- `tag` 组件名，但它也可以是`Vue`构造函数或一个对象，即没有这个组件时用它来创建。
- `data` `VNode`的`data`数据。
- `children` 子节点数据。
- `normalizationType` 标准化类型，为`1(SIMPLE_NORMALIZE)`或者`2(ALWAYS_NORMALIZE)`，决定如何处理子节点。
### 过程
1. 如果`data`存在但是上面已经有`__ob__`属性了，即它是一个被观测的对象，返回一个空节点。非生产环境给出提示。什么时候会出现这种情况呢？应该是手写`render`时会导致这种错误吧。
2. 如果`data.is`属性存在，将`tag`设为`data.is`，即我们使用`component`组件时传入的`is`属性。
3. 如果`tag`为假，返回一个空节点。这是因为`component`组件的`is`属性可能被设为一个假值。
4. 对于`key`值，如果它不是一个原始值，非生产环境下给出警告，即我们使用`v-for`时（当然其它时候也可以传）传入的`key`值，不能是一个对象之类的。除了要求非生产环境，还要当前不是`weex`环境，或者`data.key`中不包含`@binding`属性，这应该都是和`wexx`相关的，略过不提。
5. 对于子节点是一个数组，且第一个元素是一个函数，即它是默认的`scoped slot`，参考[这里]()，为`data`重新设置`scopedSlots`，并将`children`设为空数组。
6. 接着根据`normalizationType`的值调用不同的函数规范化子节点，简单来说因为`children`的值不总是`VNode[]`这样的结构，需要将其统一成上述结构，其中又有几种情况，详情请看[normalize-children.js](./helpers/normalize-children.md)。
7. 然后定义了`vnode`和`ns`两个变量，`vnode`将会保存着最后返回的值，`ns`将会是当前`tag`的命名空间。
8. 如果`tag`是字符串类型，也是最常用的：
   1. 定义`Ctor`用来在需要的时候获取`Vue`构造函数。
   2. 为`ns`赋值，如果绑定的`vm`已有父节点且其`ns`属性为真，获取它的`ns`值，否则调用`config.getTagNamespace`，这个函数就是在`web`平台下，就是`tag`为`svg`或者`math`时返回`svg`或者`math`。以`svg`为例，`svg`标签下所有的节点的`ns`都为`sbg`。
   3. 判断`tag`是否是保留标签，在`web`平台下，就是已经在`html`规范下的标签，如`p`，是的话创建`vnode`，其中的`tag`值是通过`config.parsePlatformTagName`来转换的。比如大小写转换之类的。
   4. 如果`data`不存在，因为一个组件可能不需要任何数据，或者`data`存在但是`data.pre`为假，即没有使用`v-pre`指令，且`Ctor = resolveAsset(context.$options, 'components', tag)`为真，即`tag`已经注册到选项里了。`resolveAsset`的作用就是从`options`中取出指定的资源，在这里是返回`tag`组件的构造函数。详情看[resolveAsset]()。说明这是一个组件，调用`createComponent`创建一个组件的`VNode`。
   5. 以上条件都不满足，说明这是一个未知或者没有注册的组件。直接创建`VNode`。问题来了，记得使用没有注册的组件时，会给出提示的，这里却并没有任何提示。这是因为把检查未知组件这一步放到运行时处理了，注释提到了因为它可能在它的父节点规范子节点的时候为它分配命名空间
      - [ ] 即，虽然创建节点时没有这个组件，但是可能渲染时它就有了，`web`平台下检查在`createElm`中的[isUnknownElement](./patch.md)发生。
9. 如果`tag`不是字符串，那它可能是一个构造函数或者扩展选项，将其直接传入`createComponent`中。
10. 如果创建的`vnode`是一个数组，直接返回。
11. 如果`vnode`不存在，返回一个空节点。
12. 如果存在：
    1. 如果命名空间存在，调用`applyNS(vnode, ms)`，它的作用就是对`vnode`和其子节点的`ns`赋值，详情见下文。
    2. 如果`data`存在，即传入了数据，调用`registerDeepBindings`，它的作用是在`slot`上使用`:style`和`:class`且用的是对象语法时，能正确收集依赖并渲染。详情见下文。
    3. 返回`vnode`.
13. 完毕。
## applyNS
### 作用
为`vnode.ns`赋值，并且在其子节点满足条件时，也调用此函数。
### 过程
1. 令`vnode.ns = ns`。
2. 如果标签为`forginObject`，重置`ns`等于`undefined`，`force`为`true`，但是，在上面的代码中能看到，它在`web`平台下好像是不会等于这个的。所以跳过。
3. 接着遍历其子节点，如果子节点有标签并且（没有命名空间或者这一段跳过），同样调用此函数。
4. 完毕。
## registerDeepBindings
### 作用 && 过程
- [ ] [issue#5318]()在`slot`上使用`:style`和`:class`且用的是对象语法时，能正确收集依赖并渲染。如果`data.style`或者`data.class`存在且是一个对象，调用`traverse`遍历它，收集深层依赖。[traverse]()就是读取一个对象的所有嵌套属性。
## 代码
```javascript
/* @flow */

import config from '../config'
import VNode, { createEmptyVNode } from './vnode'
import { createComponent } from './create-component'
import { traverse } from '../observer/traverse'

import {
  warn,
  isDef,
  isUndef,
  isTrue,
  isObject,
  isPrimitive,
  resolveAsset
} from '../util/index'

import {
  normalizeChildren,
  simpleNormalizeChildren
} from './helpers/index'

const SIMPLE_NORMALIZE = 1
const ALWAYS_NORMALIZE = 2

// wrapper function for providing a more flexible interface
// without getting yelled at by flow
export function createElement (
  context: Component,
  tag: any,
  data: any,
  children: any,
  normalizationType: any,
  alwaysNormalize: boolean
): VNode | Array<VNode> {
  if (Array.isArray(data) || isPrimitive(data)) {
    normalizationType = children
    children = data
    data = undefined
  }
  if (isTrue(alwaysNormalize)) {
    normalizationType = ALWAYS_NORMALIZE
  }
  return _createElement(context, tag, data, children, normalizationType)
}

export function _createElement (
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {
  if (isDef(data) && isDef((data: any).__ob__)) {
    process.env.NODE_ENV !== 'production' && warn(
      `Avoid using observed data object as vnode data: ${JSON.stringify(data)}\n` +
      'Always create fresh vnode data objects in each render!',
      context
    )
    return createEmptyVNode()
  }
  // object syntax in v-bind
  if (isDef(data) && isDef(data.is)) {
    tag = data.is
  }
  if (!tag) {
    // in case of component :is set to falsy value
    return createEmptyVNode()
  }
  // warn against non-primitive key
  if (process.env.NODE_ENV !== 'production' &&
    isDef(data) && isDef(data.key) && !isPrimitive(data.key)
  ) {
    if (!__WEEX__ || !('@binding' in data.key)) {
      warn(
        'Avoid using non-primitive value as key, ' +
        'use string/number value instead.',
        context
      )
    }
  }
  // support single function children as default scoped slot
  if (Array.isArray(children) &&
    typeof children[0] === 'function'
  ) {
    data = data || {}
    data.scopedSlots = { default: children[0] }
    children.length = 0
  }
  if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }
  let vnode, ns
  if (typeof tag === 'string') {
    let Ctor
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    if (config.isReservedTag(tag)) {
      // platform built-in elements
      vnode = new VNode(
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      )
    } else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component
      vnode = createComponent(Ctor, data, context, children, tag)
    } else {
      // unknown or unlisted namespaced elements
      // check at runtime because it may get assigned a namespace when its
      // parent normalizes children
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      )
    }
  } else {
    // direct component options / constructor
    vnode = createComponent(tag, data, context, children)
  }
  if (Array.isArray(vnode)) {
    return vnode
  } else if (isDef(vnode)) {
    if (isDef(ns)) applyNS(vnode, ns)
    if (isDef(data)) registerDeepBindings(data)
    return vnode
  } else {
    return createEmptyVNode()
  }
}

function applyNS (vnode, ns, force) {
  vnode.ns = ns
  if (vnode.tag === 'foreignObject') {
    // use default namespace inside foreignObject
    ns = undefined
    force = true
  }
  if (isDef(vnode.children)) {
    for (let i = 0, l = vnode.children.length; i < l; i++) {
      const child = vnode.children[i]
      if (isDef(child.tag) && (
        isUndef(child.ns) || (isTrue(force) && child.tag !== 'svg'))) {
        applyNS(child, ns, force)
      }
    }
  }
}

// ref #5318
// necessary to ensure parent re-render when deep bindings like :style and
// :class are used on slot nodes
function registerDeepBindings (data) {
  if (isObject(data.style)) {
    traverse(data.style)
  }
  if (isObject(data.class)) {
    traverse(data.class)
  }
}
```