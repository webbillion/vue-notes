# core/vdom/patch.js
这一篇构造了`virtual dom`系统，前置文章[这里](./README.md)。因为代码过长，所以分几个部分讲解。

- [pre](./patch_pre.md) 在`createPatchFunction` 前定义的。
   - [sameVnode]() 判断节点是否类型相同，即能否复用。
   - [sameInputType]() 在节点`tag`为`input`时判断能否复用。
   - [createKeyToOldIdx]() 创建`key -> index`的索引映射。
- [invoke](./patch_invoke.md) 调用钩子函数相关。
  - [invokeDestroyHook]() 调用`destroy`钩子和回调。
  - [invokeCreateHooks]() 调用`create`钩子和回调，并插入到`insert`队列中。
- [patch](./patch_create&&update.md) 创建元素相关。
  - [createElm]() 创建新元素。
  - [createComponent]() 创建新元素时会先尝试创建一个组件。
  - [initComponent]() 创建组件后进行初始化，包括设置`vnode.elm`，调用`create`钩子等。
  - [createChildren]() 对节点的子节点依次调用`createElm`。
  - [insert]() 将元素插入指定位置。
- [patch](./patch_patch.md) 实际`patch`。
  - [isPatchable]() 判断一个节点是否可`patch`，不是则不需要进行`ref`之外的属性的更新。
  - [patchVnode]() 调用`prepatch`钩子后调用`update`钩子进行更新。
  - [updateChildren]() 在`patchVnode`过程中寻找可以复用的子元素进行`patchVnode`，否则创建或移除。

## createPatchFunction
### 作用
它用来创建一个`patch`函数，回顾在[_update](../instance/lifecycle.md)中我们提到过`__patch__`函数，它将`vnode`渲染成真实`dom`，是由运行时环境添加到`Vue.prototype`上的，运行时环境就是调用本函数创建了一个`patch`函数。为什么要由运行时环境来添加？这是因为在渲染成真实`dom`之前，`vnode`是无关平台的（当然过程中也处理了一些平台相关的东西），而`vue`是可以在多个平台下使用的，在不同平台下有不同的映射方案，所以需要运行时来决定如何`patch`。
### 参数
虽然是一个，但是等于是两个
- `modules` 模块，什么模块？在新旧`vnode`间如何更新属性和事件的模块。参考[这里]()。
- `nodeOps` 节点选项，它提供了如何操纵节点的方法，比如`removeChild`。参考[这里]()。
### 过程
1. 定义变量`cbs`，用来保存在需要触发钩子时，依次调用函数。
2. 遍历`hooks`和`modules`，向`cbs`中添加值，`web`平台下遍历完成后其值如下：
```javascript
cbs = {
  create: [
    modules.attrs.create,
    modules.class.create
  ]
  // ...
}
```
3. 基于上面的东西，定义了很多函数，暂时跳过。
4. 返回真正的`patch`函数。
### patch
#### 作用
比较新旧节点，更新或创建，返回新的元素`el`。
#### 参数
- `oldVnode` 旧节点
- `vnode` 新节点
- `hydrating` - [ ] 服务端相关，跳过
- `removeOnly` 是否只移除，这个参数最终在[updateChildren]中被使用，是留给`transtion-group`使用的标志。
#### 过程
1. 如果新节点未定义，直接返回。其中如果旧节点有定义的话，说明这个节点被删除了。调用[invokeDestoryHook]()函数，触发`destory`钩子做相应处理。
2. 接着定义了两个变量：
   - `isInitialPatch` 是否是第一次运行`patch`
   - `insertedVnodeQueue` 插入节点的队列，在调用`insert`钩子时，会等节点及其子节点都插入完成了再依次调用此队列的钩子。
3. 如果`oldVnode`没有定义，说明节点是第一次`patch`（回想在[_update](../instance/lifecycle.md)中调用`patch`时，如果之前没有`vnode`，则第一个参数传入`vm.$el`，- [ ] 应该是服务端渲染过的话这个元素是存在的，否则不存在）。设置`isInitialPatch`为`true`，调用[createElm](#createElm)创建新元素，这里还传入了`insertedVnodeQueue`作为参数。否则进入第4步.
4. 通过`nodeType`来判定`oldVnode`是否是真实节点，并将值保存在`isRealElement`中。接着如果不是真实节点，即它是一个组件节点，并且新旧节点可以复用：
   1. 进行[patchVnode](#patchVnode)更新。否则进入下一步。
   2. 如果是真实节点：
      1. 检查节点类型为`1`，即普通元素节点，并且有[SSR_ATTR]()属性，说明这是一个由服务端渲染好了的节点，那么移除它的这个属性，并将`hydrating`设为`true`，代表接下来的渲染中，都会知道这是在接管服务端渲染后的内容。
      2. 接着判定`hydrating`是否为真，即是否为服务端渲染，如果不是，调用[emptyNodeAt](#emptyNodeAt)，这个函数会创建一个和真实元素标签相同的节点，并将元素赋值给节点的`elm`，用这个节点`oldVnode`。如果是，进入下一步。
      3. 尝试调用[hydrate](#hydrate)将服务端的内容和浏览器的内容混合起来。如果成功了，调用[invokeInsertHook](#invokeInsertHook)调用`insert`钩子。返回`oldVnode`，即原来的真实节点，`patch`函数结束。如果是非生产环境，给出警告。
5. 只有在是服务端渲染并且混合成功后，不会来到这一步。其它情况下，到这里时，已经将`oldVnode`处理得符合标准了。定义两个变量：
   - `oldElm` `oldVnode.elm` 原先的`dom`元素。
   - `parentElm` `oldElm`的父元素。

6. 来到这一步即说明旧节点无法复用，所以调用`createElm`创建新的元素。这里有个参数`oldElm._leaveCb ? null : parentElm`，关于它
   - [ ] [issue#4590]()，解决的问题是当`transition`和`keep-alive`和`HOC`一起使用时出现的问题，即元素在`levaing transting`时，不要将它插入。
7. 创建新元素后，要做几个判断，它们分别是：
   1. `vnode.parent`是否有定义，即节点是否有父节点，如果有：
      1. 定义`ancestor`，祖先元素，为`vnode.parent`，定义`patchable`为[isPatchable(vnode)](#isPatchable)的值，即是否可应用属性更新。
      2. 如果`ancestor`存在的话，进入一个循环体，在循环体最后会令`ancestor = ancestor.parent`。
      3. 调用`cbs.destory`中的钩子，实际上是对祖先节点之前的`ref`和`directive`进行移除。然后令`ancestor.elm` = `vnode.elm`。
      > 想一下`vnode.parent`这个属性是怎么来的？它在[updateChildComponent](../instance/lifecycle.md)中被定义，并不是说只有根节点才没有`parent`属性，而是只有组件的`vm._vnode`才有`parent`属性，而且它等同于`vm.$vnode`，即组件在使用它的父组件的节点树中的占位，如果`vm._vnode.parent`仍然有`parent`属性，说明它也是一个组件的`vm._vnode`，这种情况，只有**高阶组件**才会出现。

      ><br>所以上面说了这么多，一言蔽之，这段代码的意思是对高阶组件和组件里，组件节点的所有祖先节点进行操作。之所以是在创建子元素时对父节点进行操作，是因为对`tag: vue-componet-3-child`这样的父节点进行`patch`时认为它是可复用的，但在进行真正的`dom`映射时是重新创建了`dom`的，而不是复用，实际上算是重新创建了，因此`create`钩子和其它属性等要单独设置和调用。
      > - [ ] 暂时这么理解。
      4. 接着如果`patchable`为真，同样对祖先元素调用`create`钩子，注意这里的参数，是`(emptyNode, ancestor)`，相当于对一个空节点进行更新的操作，实际上`cbs.create`和`cbs.update`中是同一个函数，所以这里等于调用`cbs.update`。
      5. 定义`insert` = `ancestor.data.hook.insert`，如果`insert.merged`为真，从索引1开始依次调用`insert.fns`中的钩子函数。
      > - [ ] [issue#6513]()<br>
      > 有的钩子函数`create`在调用时会为`data.hook.insert`添加函数，将原来的函数和新增的函数合并到`insert.fns`中，比如指令的[create]()钩子，在其内部，调用了[mergeVNodeHook]()，它就像上面那样做，并设置`insert.merged = true`。
      > <br>从索引1开始是因为索引0，也就是原来的[hook.insert]()是用来触发`mounted`生命周期或者激活组件之类的，这里不需要再次触发？注释说是为了避免再次触发`mounted`，但在其内部实现里，多次调用`insert`也不会多次触发啊。
      1. 如果`patchable`为假，只对`ref`重新注册。原理同上。
   2. 如果父元素有定义，调用[removeVnodes](#removeVnodes)将旧节点移除，并触发`destory`和`remove`钩子。是的，当你看到`removeVnodes`函数时会发现，这个`parentElm`参数在里面没有用到。
   3. 如果没有父元素且`oldVnode.tag`为真，即不为注释和文本节点，调用`invokeDestroyHook`，触发`destory`钩子。
8. 调用[invokeInsertHook](#invokeInsertHook)，依次触发所有节点的`insert`钩子。子节点的`mounted`生命周期，也在这里才被触发。
9. 返回新的元素。
10. 完毕。
## 其它
上面的内容模模糊糊看完了可能还是有许多疑惑，我现在也是，不过怎么也算清楚大概的流程了，剩下的应该需要集合`modules`内的东西才能看明白。看过之后再回望，也许就清楚了。
## 梳理下还没有写的函数，后续再补
- - [ ] `removeAndInvokeRemoveHook`
- - [ ] `hydrate` 
- - [ ] `invokeInsertHook` 
## 代码
```javascript
export function createPatchFunction (backend) {
  let i, j
  const cbs = {}

  const { modules, nodeOps } = backend

  for (i = 0; i < hooks.length; ++i) {
    cbs[hooks[i]] = []
    for (j = 0; j < modules.length; ++j) {
      if (isDef(modules[j][hooks[i]])) {
        cbs[hooks[i]].push(modules[j][hooks[i]])
      }
    }
  }
  // ...函数定义
  return function patch (oldVnode, vnode, hydrating, removeOnly) {
    if (isUndef(vnode)) {
      if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
      return
    }

    let isInitialPatch = false
    const insertedVnodeQueue = []

    if (isUndef(oldVnode)) {
      // empty mount (likely as component), create new root element
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)
    } else {
      const isRealElement = isDef(oldVnode.nodeType)
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        // patch existing root node
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
      } else {
        if (isRealElement) {
          // mounting to a real element
          // check if this is server-rendered content and if we can perform
          // a successful hydration.
          if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
            oldVnode.removeAttribute(SSR_ATTR)
            hydrating = true
          }
          if (isTrue(hydrating)) {
            if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
              invokeInsertHook(vnode, insertedVnodeQueue, true)
              return oldVnode
            } else if (process.env.NODE_ENV !== 'production') {
              warn(
                'The client-side rendered virtual DOM tree is not matching ' +
                'server-rendered content. This is likely caused by incorrect ' +
                'HTML markup, for example nesting block-level elements inside ' +
                '<p>, or missing <tbody>. Bailing hydration and performing ' +
                'full client-side render.'
              )
            }
          }
          // either not server-rendered, or hydration failed.
          // create an empty node and replace it
          oldVnode = emptyNodeAt(oldVnode)
        }

        // replacing existing element
        const oldElm = oldVnode.elm
        const parentElm = nodeOps.parentNode(oldElm)

        // create new node
        createElm(
          vnode,
          insertedVnodeQueue,
          // extremely rare edge case: do not insert if old element is in a
          // leaving transition. Only happens when combining transition +
          // keep-alive + HOCs. (#4590)
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        )

        // update parent placeholder node element, recursively
        if (isDef(vnode.parent)) {
          let ancestor = vnode.parent
          const patchable = isPatchable(vnode)
          while (ancestor) {
            for (let i = 0; i < cbs.destroy.length; ++i) {
              cbs.destroy[i](ancestor)
            }
            ancestor.elm = vnode.elm
            if (patchable) {
              for (let i = 0; i < cbs.create.length; ++i) {
                cbs.create[i](emptyNode, ancestor)
              }
              // #6513
              // invoke insert hooks that may have been merged by create hooks.
              // e.g. for directives that uses the "inserted" hook.
              const insert = ancestor.data.hook.insert
              if (insert.merged) {
                // start at index 1 to avoid re-invoking component mounted hook
                for (let i = 1; i < insert.fns.length; i++) {
                  insert.fns[i]()
                }
              }
            } else {
              registerRef(ancestor)
            }
            ancestor = ancestor.parent
          }
        }

        // destroy old node
        if (isDef(parentElm)) {
          removeVnodes(parentElm, [oldVnode], 0, 0)
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode)
        }
      }
    }

    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
    return vnode.elm
  }
}
```