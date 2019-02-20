---
vue-note-completed: true
---
# core/vdom/create-component.md
## createComponent
### 作用
创建一个自定义组件的`Vnode`。
### 参数
- `Ctor` 组件的构造函数，也可以是对象，或者是一个异步组件的工厂函数，最后都会被解析为构造函数。
- 其它参数参考[createElement](./create-element.md)。
### 过程
1. 如果`Ctor`没有定义，直接返回。
2. 获取初始，即没有经过任何扩展的构造函数`baseCtor` = `context.$options._base`，它来自[global-api/index.js](../global-api/index.md)。
3. 如果`Ctor`是一个对象，调用`extend`对其进行扩展。它来自[global-api/extend.js](../global-api/extend.md)。
4. 如果经过上面的步骤后`Ctor`仍然不是函数，非生产环境给出警告。
5. 定义变量`asyncFactory`，如果是异步组件的话将`Ctor`赋值给它。怎么判断的呢？异步组件的工厂函数是没有`cid`属性，而扩展而来的构造函数是有的，参考[global-api/extend.js](../global-api/extend.md)。如果是一个异步组件：
   1. 调用[resolveAsyncComponent](./helpers/resolve-async-component.md)将工厂函数内真正的组件解析出来，它会调用函数，然后根据它的状态来返回结果，在这一步，如果定义了`loading`且设置了`delay`为0，会返回`loading`组件的构造函数，否则都应该是`undefined`，进入下一步。
   2. 调用[createAsyncPlaceholder](./helpers/resolve-async-component.md)，创建一个用于异步组件占位的`VNode`并返回，这个`vnode`会保存所有数据，但是是一个注释节点。
6. 重新解析构造函数的选项，因为可能中途使用了全局`mixin`，更多信息参考[resolveConstructorOptions](../instance/init.md)。
7. 如果`data.model`存在，即使用了`v-model`指令，调用`transformModel`对其进行转换，因为`v-model`实际上只是语法糖。转换的结果是将其赋值到了`data`的`props`和`on`上。
8. 接着调用`extractPropsFromVNodeData`从`data`中提取出`propsData`作为`props`的数据来源。主要是从`data.props`和`data.attrs`中拿到`Ctor.options.props`中定义的值。
9. 如果是一个函数式组件，传入所有参数调用并直接返回[createFunctionalComponent](./create-functional-component.md)。
10. 定义`listeners`保存`data.on`，这些是监听子组件的，而下文的`nativeOn`，即使用了`.native`修饰符的事件，则是监听到`dom`上的。在`dom`中监听的事件，会在[patch]()时进行处理，比如移除。
11. 接着判断是否是抽象组件，如果是，将`data`置空，只保留`slot`属性，`propsDAta`和`listeners`已经在上面提取出来了。因为抽象组件不需要那些东西。
12. 然后调用[installComponentHooks](#installComponentHooks)将组件管理钩子放到`data`上，这些钩子用来做什么？用来在`patch`两个节点时可以操作节点。
13. 接着创建一个占位节点，可以看到它的名字以`vue-component-`开头，假设组件叫`child`，在父组件中渲染它时，`vue-component-1-child`就是它在父组件渲染树中的`vnode`，但是`child`组件自身的节点并不是这个。可以看到在`new VNode`时传入了`componentOptions`，它的值包括了：
    - `Ctor` 当前组件的构造函数。
    - `propsData` 之前解析出的父组件传递给`props`的数据。
    - `listeners` 要监听的事件。
    - `tag` 标签名，组件名
    - `children` 子节点。
> 可以看到，字节点`children`并没有直接传给`VNode`的构造函数，而是传递到`componentOptions`中。
> - [ ] 这代表着什么呢？
14. 下面是关于`weex`的代码，暂时略过。
15. 返回`vnode`。
16. 完毕。
## installComponentHooks
### 作用
给`VNodeData`添加`hook`属性，即将`componentVNodeHooks`赋值给`data.hook`，另外就是最多可以给每个钩子添加两个函数。
## componentVNodeHooks
里面有用于`patch`的一些钩子函数。
### init
#### 作用
初始化`vnode`
#### 参数
- `vnode` 已经有数据的`VNode`实例。
- `hydrating` 用于服务端渲染好像，暂时不管。
#### 过程
1. 如果`vnode`上已经有`componentInstance`组件实例了，而且这个组件实例没有被摧毁，而且它属于`keepalive`，调用下文的`prepatch`，将两个节点进行比较，这是因为，将`keep-alive`的切换过程，当成一个打补丁的过程。
   - [ ] 为什么?可能要等到看到相关源码才明白。
2. 否则调用[createComponentInstanceForVnode](#createComponentInstanceForVnode)为`vnode`添加`componentInstace`属性，然后调用实例的挂载方法。
   - [ ] 关于`hydrating`，后续再考虑。
3. 完毕。
> 也就是说，我们调用`$mount`后，子组件的挂载是由这段代码来调用的。这同样意味着这段代码只会在生成子组件的节点时使用。
### prepatch
获取新节点的参数将其传入[updateChildComponent](../instance/lifecycle.md)，以更新`props` `listners` `vnode` `children`。这一步是为了在[_update]()中调用[__patch__]()做准备，所以叫做`prepatch`，即`patch`的前置工作。
### insert
#### 作用
- [ ] 插入节点？在`vnode`的元素被插入到`dom`中时触发。实际上是做了触发`mounted`生命周期，以及激活子组件。
#### 过程
1. 从`vnode`中获取`context`上下文和`componentInstance`组件实例。
2. 如果组件实例的`_isMounted`为`false`，将其改为`true`，并触发`mounted`生命周期钩子。
> 回想在[mountComponent](../instance/lifecycle.md)中，非根组件在挂载完成后是不会触发`mounted`钩子的。
3. 如果组件在`keep-alive`下：
   1. 如果`context._isMounted`为真，即`keep-alive`所在的组件已经挂载。调用[queueActivatedComponent](../instance/lifecycle.md)将组件放入激活队列中。这里的[vue-router-issue#1212] - [ ] 以后再说。
   2. 如果还没有挂载，调用[activateChildComponent](../instance/lifecycle.md)进行激活。
### destory
#### 作用 && 过程
摧毁节点，调用节点关联的实例的`$destroy`方法摧毁，或者在`keep-alive`下调用[deactivateChildComponent](../instance/lifecycle.md)方法将其置为未激活状态。
## createComponentInstanceForVnode
### 作用
为`vnode`创建一个组件实例。
### 参数
- `vnode` 组件的`vnode`。
- `parent` 组件的父组件，实际上只会是[activeInstace](../instance/lifecycle.md)，也就是当前正在更新的组件。回想一下在[_update](../instance/lifecycle.md)中设置了`activeInstance`后就进入了`__patch__`，正好对应了起来。
###  过程
1. 定义`options`为内部组件选项：
   - `_isComponent` 是否是内部组件，在[_init](../instance/init.md)中有用到，为`true`。
   - `_parentVnode` 在父组件渲染树中的`vnode`，值为`vnode`。在[_init](../instance/init.md)和[render](../instance/render.md)中都有使用。保存着父组件传递给子组件的信息。
   - `parent` 父组件实例。
2. 如果`vnode.data`中有`inlineTemplate`选项，即内联模板，编译时生成，将其`render`和`staticRenderFns`赋值给`options`。
3. 调用当前组件的构造函数并传入内部选项。
> 这里需要注意的是，如果有`inlineTemplate`，原有构造函数的`render`将会被覆盖。暂时不知道内联模板的应用场景。
4. 完毕。
   
## 代码
```javascript
/* @flow */

import VNode from './vnode'
import { resolveConstructorOptions } from 'core/instance/init'
import { queueActivatedComponent } from 'core/observer/scheduler'
import { createFunctionalComponent } from './create-functional-component'

import {
  warn,
  isDef,
  isUndef,
  isTrue,
  isObject
} from '../util/index'

import {
  resolveAsyncComponent,
  createAsyncPlaceholder,
  extractPropsFromVNodeData
} from './helpers/index'

import {
  callHook,
  activeInstance,
  updateChildComponent,
  activateChildComponent,
  deactivateChildComponent
} from '../instance/lifecycle'

import {
  isRecyclableComponent,
  renderRecyclableComponentTemplate
} from 'weex/runtime/recycle-list/render-component-template'

// inline hooks to be invoked on component VNodes during patch
const componentVNodeHooks = {
  init (vnode: VNodeWithData, hydrating: boolean): ?boolean {
    if (
      vnode.componentInstance &&
      !vnode.componentInstance._isDestroyed &&
      vnode.data.keepAlive
    ) {
      // kept-alive components, treat as a patch
      const mountedNode: any = vnode // work around flow
      componentVNodeHooks.prepatch(mountedNode, mountedNode)
    } else {
      const child = vnode.componentInstance = createComponentInstanceForVnode(
        vnode,
        activeInstance
      )
      child.$mount(hydrating ? vnode.elm : undefined, hydrating)
    }
  },

  prepatch (oldVnode: MountedComponentVNode, vnode: MountedComponentVNode) {
    const options = vnode.componentOptions
    const child = vnode.componentInstance = oldVnode.componentInstance
    updateChildComponent(
      child,
      options.propsData, // updated props
      options.listeners, // updated listeners
      vnode, // new parent vnode
      options.children // new children
    )
  },

  insert (vnode: MountedComponentVNode) {
    const { context, componentInstance } = vnode
    if (!componentInstance._isMounted) {
      componentInstance._isMounted = true
      callHook(componentInstance, 'mounted')
    }
    if (vnode.data.keepAlive) {
      if (context._isMounted) {
        // vue-router#1212
        // During updates, a kept-alive component's child components may
        // change, so directly walking the tree here may call activated hooks
        // on incorrect children. Instead we push them into a queue which will
        // be processed after the whole patch process ended.
        queueActivatedComponent(componentInstance)
      } else {
        activateChildComponent(componentInstance, true /* direct */)
      }
    }
  },

  destroy (vnode: MountedComponentVNode) {
    const { componentInstance } = vnode
    if (!componentInstance._isDestroyed) {
      if (!vnode.data.keepAlive) {
        componentInstance.$destroy()
      } else {
        deactivateChildComponent(componentInstance, true /* direct */)
      }
    }
  }
}

const hooksToMerge = Object.keys(componentVNodeHooks)

export function createComponent (
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  if (isUndef(Ctor)) {
    return
  }

  const baseCtor = context.$options._base

  // plain options object: turn it into a constructor
  if (isObject(Ctor)) {
    Ctor = baseCtor.extend(Ctor)
  }

  // if at this stage it's not a constructor or an async component factory,
  // reject.
  if (typeof Ctor !== 'function') {
    if (process.env.NODE_ENV !== 'production') {
      warn(`Invalid Component definition: ${String(Ctor)}`, context)
    }
    return
  }

  // async component
  let asyncFactory
  if (isUndef(Ctor.cid)) {
    asyncFactory = Ctor
    Ctor = resolveAsyncComponent(asyncFactory, baseCtor, context)
    if (Ctor === undefined) {
      // return a placeholder node for async component, which is rendered
      // as a comment node but preserves all the raw information for the node.
      // the information will be used for async server-rendering and hydration.
      return createAsyncPlaceholder(
        asyncFactory,
        data,
        context,
        children,
        tag
      )
    }
  }

  data = data || {}

  // resolve constructor options in case global mixins are applied after
  // component constructor creation
  resolveConstructorOptions(Ctor)

  // transform component v-model data into props & events
  if (isDef(data.model)) {
    transformModel(Ctor.options, data)
  }

  // extract props
  const propsData = extractPropsFromVNodeData(data, Ctor, tag)

  // functional component
  if (isTrue(Ctor.options.functional)) {
    return createFunctionalComponent(Ctor, propsData, data, context, children)
  }

  // extract listeners, since these needs to be treated as
  // child component listeners instead of DOM listeners
  const listeners = data.on
  // replace with listeners with .native modifier
  // so it gets processed during parent component patch.
  data.on = data.nativeOn

  if (isTrue(Ctor.options.abstract)) {
    // abstract components do not keep anything
    // other than props & listeners & slot

    // work around flow
    const slot = data.slot
    data = {}
    if (slot) {
      data.slot = slot
    }
  }

  // install component management hooks onto the placeholder node
  installComponentHooks(data)

  // return a placeholder vnode
  const name = Ctor.options.name || tag
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children },
    asyncFactory
  )

  // Weex specific: invoke recycle-list optimized @render function for
  // extracting cell-slot template.
  // https://github.com/Hanks10100/weex-native-directive/tree/master/component
  /* istanbul ignore if */
  if (__WEEX__ && isRecyclableComponent(vnode)) {
    return renderRecyclableComponentTemplate(vnode)
  }

  return vnode
}

export function createComponentInstanceForVnode (
  vnode: any, // we know it's MountedComponentVNode but flow doesn't
  parent: any, // activeInstance in lifecycle state
): Component {
  const options: InternalComponentOptions = {
    _isComponent: true,
    _parentVnode: vnode,
    parent
  }
  // check inline-template render functions
  const inlineTemplate = vnode.data.inlineTemplate
  if (isDef(inlineTemplate)) {
    options.render = inlineTemplate.render
    options.staticRenderFns = inlineTemplate.staticRenderFns
  }
  return new vnode.componentOptions.Ctor(options)
}

function installComponentHooks (data: VNodeData) {
  const hooks = data.hook || (data.hook = {})
  for (let i = 0; i < hooksToMerge.length; i++) {
    const key = hooksToMerge[i]
    const existing = hooks[key]
    const toMerge = componentVNodeHooks[key]
    if (existing !== toMerge && !(existing && existing._merged)) {
      hooks[key] = existing ? mergeHook(toMerge, existing) : toMerge
    }
  }
}

function mergeHook (f1: any, f2: any): Function {
  const merged = (a, b) => {
    // flow complains about extra args which is why we use any
    f1(a, b)
    f2(a, b)
  }
  merged._merged = true
  return merged
}

// transform component v-model info (value and callback) into
// prop and event handler respectively.
function transformModel (options, data: any) {
  const prop = (options.model && options.model.prop) || 'value'
  const event = (options.model && options.model.event) || 'input'
  ;(data.props || (data.props = {}))[prop] = data.model.value
  const on = data.on || (data.on = {})
  const existing = on[event]
  const callback = data.model.callback
  if (isDef(existing)) {
    if (
      Array.isArray(existing)
        ? existing.indexOf(callback) === -1
        : existing !== callback
    ) {
      on[event] = [callback].concat(existing)
    }
  } else {
    on[event] = callback
  }
}

```