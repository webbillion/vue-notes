### isPatchable
#### 作用 && 过程
这个函数用来判定组件节点层层嵌套后，最后一个不是组件节点的节点，是否是为文本节点或注释节点的。也只有不是，才能够进行`patch`。实际上只有在高阶组件中才会多次进入循环体。
```html
  <!-- child1 -->
  <child2 title="child2"></child2>
  <!-- child2 -->
  <template>文本节点</template>
```
> 如上例所示，在初始化`child1`中`child2`组件的节点时，其最终转换成的`el`是文本节点，所以的属性和事件等都用不上，不必进入属性更新这一步——除了`ref`属性。至于原因，很显然嘛。
## patchVnode
### 作用
修补节点啦。
### 参数
- `oldVnode` 旧节点。
- `vnode` 新节点。
- `insertedVnodeQueue` 待`insert`的队列。
- `ownerArray` 父节点的`children`数组。
- `index` 在父节点`children`中的索引。
- `removeOnly` 只移除。
### 过程
1. 如果新旧节点相同，直接返回。
   - [ ] 什么时候会相同？
2. 如果新节点已有元素存在，并且传入了`ownerArray`，克隆`vnode`并替换其在父节点`children`中的位置。这一步同样是为了解决那些已经使用过的`vnode`被当作新节点使用时的问题。
   > 注释说这也解除了使用不能多个相同`slot`的限制。
   - [ ] 什么情况会出现？
3. 将旧节点的`elm`赋值给`vnode.elm`，并保留引用。
4. 接着判断旧节点是否是异步组件占位节点，如果是：
   1. 工厂函数是否加载完毕？是的话调用[hydrate]()将二者混合起来。
   2. 否则将新节点的`isAsyncPlaceholder`设为`true`。
   3. 返回。
5. 不是的话又要做另一个判断，是否是静态渲染树，包含的条件是：
   - `vnode.isStatic`为真，即新节点是一个静态节点或者一次性节点。
   - `oldVnode.isStatic`为真，同上。
   - 新旧节点`key`值相同。
   - 新节点是克隆节点或者一次性节点。
6. 那么直接将旧节点的组件实例赋值给新节点，直接返回即可。很显然静态节点不需要变。
7. 接着如果有`prepatch`钩子，调用[hook.prepatch]()做一些更新前的准备工作，它主要是更新了节点上的组件实例的属性。
8. 定义`olcCh`和`ch`为旧新节点的`children`引用。
9. 如果`vnode`可`patch`，依次调用`cbs.update`进行元素的属性更新。如果`hook.update`存在，也调用它。——这也就是为什么，在组件上使用`class`等属性，可以直接赋值给组件关联的`dom`元素的原因。
10. 如果`vnode.text`不存在，即没有文本，做以下的分支判断：
    - 新旧节点都有子节点：
    1.  且不相等的话调用[updatechildren]()对子节点进行更新。
    - 只有新节点有子节点：
    1. 非生产环境下检查`key`值是否重复。
    2. 旧节点有文本的话将关联元素的文本设为空。
    3. 调用[addVnodes]()将子节点全部重新创建并插入。
    - 只有旧节点有子节点：
    1. 调用[removeVnodes]()将其移除。
    - 都没有子节点，但是旧节点有文本：
    1. 将关联元素的文本设为空。
 > - [ ] 这里有一个问题，有`text`属性不是就是文本节点吗？没有的话就不是，它们的标签都不同，即不可复用，怎么进入到这一步的。
11. 有文本，但是和旧节点的文本不同，重新设置。
12. 如果有`postpatch`钩子，调用[hook.postpatch]()，这个钩子在同样在`mergeVNodeHook`中添加，比如在`directive`中，触发后会调用`componentUpdated`。
- [ ] 为什么是在这一步，不知道，给我稳。
## updateChildren
这里就是传说中的`diff`算法啦。
### 作用
寻找可以复用的子元素进行`patchVnode`，否则创建或移除。
### 参数
- `parentElm` 子节点的父元素。
- `oldCh` 旧的子节点。
- `newCh` 新的子节点。
- `insertedVnodeQueue` 待调用`insert`钩子的队列。
- `removeOnly` 是否只移除，见下文`canMove`。
### 过程
关于如何`diff`，更细致的分析请搜索[深入Vue2.x的虚拟DOM diff原理]()，找不到原帖就不贴链接了。大概流程是，从新旧子节点的两端逐渐向中间对比，可以复用则复用，需要移动就移动，否则创建或移除，直到新或旧节点首尾指针都相遇，即全部遍历完毕，就结束。
1. 定义变量：
   - `oldStartIdx`  旧的子节点的首端指针，也称为旧节点的开始索引。
   - `oldEndIdx` 旧的子节点的尾端指针，也成为旧节点的结束索引。
   - `oldStartVnode` 旧的子节点的首端指针指向的节点，以下简称旧的开始节点。
   - `oldEndVnode` 旧的子节点的尾端指针指向的节点，以下简称旧的结束节点。
   - 以`new`开头的同上。
   - `oldKeyToIdx` 用[createKeyToOldIdx]()创建的旧节点的`key => index`映射。
   - `vnodeToMove` 待移动的节点。
   - `refElm` 相邻元素，在移动元素时做插入坐标。
   - `canMove` `!removeOnly`，在需要移动元素时，值为真才能进行插入，即移动操作。这是`transtion-group`中传递过来的参数，也就是靠它才能确保元素在离开的时候处于正确的位置。
2. 在非生产环境下检查有没有重复的`key`值。接着进入一个`while`循环，循环的条件是新旧节点的开始索引都小于等于结束索引，比如旧节点的开始索引大于了结束索引，说明旧的子节点遍历完毕了，新的子节点剩下的节点都需要创建，反之需要移除。
3. 循环内部寻找可以复用的节点并不断移动指针，直到不满足循环条件为止，约定一些术语和过程如下：
   - 指针向后移动，指索引加1，反之减1。
   - 指针变动后开始和结束节点自动重新赋值，不再单独说明，比如`oldEndIndx++`，则`oldEndVnode`自动等于`oldCh[oldEndx]`。
   - 下面的分支只会进入其中一个。
   1. 旧的开始节点没有定义，旧的首端指针向后移动，旧节点可能移动过了，在下面就会被移动。
   2. 旧的结束节点没有定义，旧的尾端指针向前移动，同上。
   3. 比较旧的开始节点和新的开始节点，可以复用直接复用，然后将新旧首端指针向后移动。
   4. 比较旧的结束节点和新的结束节点，可以复用直接复用，然后将新旧尾端指针向前移动。
   > 上面的很容易理解，新的开始节点和结束节点和旧的比较，能复用就直接复用。
   5. 比较旧的开始节点和新的结束节点，可以复用的话进行复用，如果`canMove`为真，将旧的开始节点的关联元素插入到旧的结束节点的位置。旧的开始指针向后移动，新的结束节点向前移动。
   6. 比较旧的结束节点和新的开始节点，同上。
   > 这两个条件是什么意思呢，思考如下情况
   > <br>旧节点 [p, div, p, span, h1]
   > <br>新节点 [h1, h3, h4, h2, p]
   > <br>新旧相比只是首尾互换，如果没有5和6的判断，直接进入第7步，因为没有`key`值，要循环比较到最后才能找到h1可以复用的节点。加了这一步则可以少很多操作。
   > <br>其次是`canMove`为真才移动元素的位置，在[transition-group]()组件定义内，它重写了组件的[_update]()函数，单独调用了[__patch__]()，并传入`vremoveOnly`为`true`，之后再调用原来的`_update`，即在`transtition-group`组件内，实际上`patch`了两次。想象上面的节点都包含在`transition-group`里面，第一次`patch`时为了动画需要，显然是不能改变元素在`dom`中的位置的。
   1. 如果以上条件不能满足，对于新的开始节点`newStartVnode`，如果节点带有`key`值，尝试找到旧子节点中拥有同样`key`值的元素，没有`key`值的话遍历没有处理的旧子节点，尝试找到`sameVnode`为真即可以复用的的元素。上述过成中找到的节点赋值给`idxInOld`，如果它未定义，说明需要重新创建元素。如果它有值，处理方式同第6步，同时将`idxInOld`节点在旧的子节点中的位置替换为`undefined`，即第1 2步会出现的情况。最后，新的首端指针后移。
4. 循环结束后，判断新旧子节点内是否还有未处理的节点，通过开始和结束索引的大小来判断，如果新子节点内有节点未处理，调用[addVnodes]()批量添加；如果旧子节点内有节点未处理，调用[removeVnodes]()批量删除。
## 代码
```javascript
function patchVnode (
    oldVnode,
    vnode,
    insertedVnodeQueue,
    ownerArray,
    index,
    removeOnly
  ) {
    if (oldVnode === vnode) {
      return
    }

    if (isDef(vnode.elm) && isDef(ownerArray)) {
      // clone reused vnode
      vnode = ownerArray[index] = cloneVNode(vnode)
    }

    const elm = vnode.elm = oldVnode.elm

    if (isTrue(oldVnode.isAsyncPlaceholder)) {
      if (isDef(vnode.asyncFactory.resolved)) {
        hydrate(oldVnode.elm, vnode, insertedVnodeQueue)
      } else {
        vnode.isAsyncPlaceholder = true
      }
      return
    }

    // reuse element for static trees.
    // note we only do this if the vnode is cloned -
    // if the new node is not cloned it means the render functions have been
    // reset by the hot-reload-api and we need to do a proper re-render.
    if (isTrue(vnode.isStatic) &&
      isTrue(oldVnode.isStatic) &&
      vnode.key === oldVnode.key &&
      (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
    ) {
      vnode.componentInstance = oldVnode.componentInstance
      return
    }

    let i
    const data = vnode.data
    if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
      i(oldVnode, vnode)
    }

    const oldCh = oldVnode.children
    const ch = vnode.children
    if (isDef(data) && isPatchable(vnode)) {
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
      if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
    }
    if (isUndef(vnode.text)) {
      if (isDef(oldCh) && isDef(ch)) {
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
      } else if (isDef(ch)) {
        if (process.env.NODE_ENV !== 'production') {
          checkDuplicateKeys(ch)
        }
        if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
        addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
      } else if (isDef(oldCh)) {
        removeVnodes(elm, oldCh, 0, oldCh.length - 1)
      } else if (isDef(oldVnode.text)) {
        nodeOps.setTextContent(elm, '')
      }
    } else if (oldVnode.text !== vnode.text) {
      nodeOps.setTextContent(elm, vnode.text)
    }
    if (isDef(data)) {
      if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
    }
  }
   function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
    let oldStartIdx = 0
    let newStartIdx = 0
    let oldEndIdx = oldCh.length - 1
    let oldStartVnode = oldCh[0]
    let oldEndVnode = oldCh[oldEndIdx]
    let newEndIdx = newCh.length - 1
    let newStartVnode = newCh[0]
    let newEndVnode = newCh[newEndIdx]
    let oldKeyToIdx, idxInOld, vnodeToMove, refElm

    // removeOnly is a special flag used only by <transition-group>
    // to ensure removed elements stay in correct relative positions
    // during leaving transitions
    const canMove = !removeOnly

    if (process.env.NODE_ENV !== 'production') {
      checkDuplicateKeys(newCh)
    }

    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      if (isUndef(oldStartVnode)) {
        oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
      } else if (isUndef(oldEndVnode)) {
        oldEndVnode = oldCh[--oldEndIdx]
      } else if (sameVnode(oldStartVnode, newStartVnode)) {
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        oldStartVnode = oldCh[++oldStartIdx]
        newStartVnode = newCh[++newStartIdx]
      } else if (sameVnode(oldEndVnode, newEndVnode)) {
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        oldEndVnode = oldCh[--oldEndIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
        oldStartVnode = oldCh[++oldStartIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
        oldEndVnode = oldCh[--oldEndIdx]
        newStartVnode = newCh[++newStartIdx]
      } else {
        if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
        idxInOld = isDef(newStartVnode.key)
          ? oldKeyToIdx[newStartVnode.key]
          : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
        if (isUndef(idxInOld)) { // New element
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        } else {
          vnodeToMove = oldCh[idxInOld]
          if (sameVnode(vnodeToMove, newStartVnode)) {
            patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
            oldCh[idxInOld] = undefined
            canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
          } else {
            // same key but different element. treat as new element
            createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
          }
        }
        newStartVnode = newCh[++newStartIdx]
      }
    }
    if (oldStartIdx > oldEndIdx) {
      refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
      addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
    } else if (newStartIdx > newEndIdx) {
      removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx)
    }
  }
```