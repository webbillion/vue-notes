## 函数内定义
### createElm
在函数开始之前定义了一个变量
#### creatingElmInPre
即使用了`v-pre`指令的元素个数
#### 作用
创建新元素
#### 参数
- `vnode` 需要创建元素的`vnode`。
- `insertedVnodeQueue` 参见上文。
- `parentElm` 父元素，如果有的话将创建的元素插入。
- `refElm` 关联元素，如果有的话插入到它之前。
- `nested` 是否嵌套。
- `ownerArray` 父节点的`children`数组，如果有。
- `index` 在父节点的`children`数组中的索引。
#### 过程
1. 如果`vnode.elm`有定义，即已经被渲染过，有了`dom`元素，并且传入了`ownerArray`参数，对传入的`vnode`进行克隆，并替换它原本在父节点中的位置。这么做是因为如果`vnode`被当做需要插入`elm`的元素来看，而它在之前又已经渲染过了，只替换它的`elm`属性的话会有潜在的问题存在。
   - [ ] 具体有什么问题，后续再来回答。
2. 设置`vnode.isRootInsert`为`nested`的反值，即是嵌套的话，就不是根元素。这是用在`transition`的`enter`钩子中检查的。
3. 接着传入这些参数到[createComponent](#createComponent)中，它用来创建一个组件。如果返回值为真，直接返回。返回值为假说明`vnode`不是组件节点，进入下一步。
4. 获取`vnode`的`tag` `children` `data`，接着进行判断，如果`tag`有定义：
   1. 在非生产环境下，如果使用了`v-pre`指令，`creatingElmInVPre`加一，然后检测是否是未知标签并给出提示。
   2. 对于有命名空间和没有命名空间的情况进行区分，分别调用对用的函数创建元素并赋值给`vnode.elm`，在这一步，`dom`才真正创建了。对于`web`平台，可以简单理解为`document.createElment(tag)`，但是这里为什么要传入`vnode`呢？因为在`tag`是`select`的情况下，需要对其`multiple`属性提前设置。
      - [ ] 至于为什么，我还不懂。
   3. 接着调用[setScope](#setScope)，为`css scoped`设置属性。
   4. 判断是否是`weex`平台，是的情况暂时忽略不提，下面只看`web`平台。
   5. 调用[createChildren](#createChildren)为子节点创建元素。
   6. 如果`data`存在，调用[invokeCreatedHooks](#invokeCreatedHooks)并传入`insertedVnodeQueue`，触发`insert`钩子。
   7. 调用[insert](#insert)，将元素插入到应插入的未知
   8. 同第一步，令`creatingElmInVPre`减一。
5. 判断如果是注释节点，创建注释节点并插入。
6. 否则认为是文本节点，创建文本节点并插入。
7. 完毕。
### createComponent
####  作用
创建组件的元素。
#### 过程
1. 定义`i`为`vnode.data`。
2. 如果`i`存在的话——它什么时候会不存在？回顾前文，首先它不是一个组件的`vnode`，其次它没有任何的属性。但结合下面来看这里仅仅是判断它是不是组件而已。进入下一步。否则结束。
3. 首先定义`isReactivated`，即是再激活，需要节点已经有实例且处于`keep-alive`下。
4. 接着如果有`init`钩子就调用。回顾[hook-init](./create-component.md)，节点没有实例的话会创建一个实例并将其挂载，挂载完成之后为它创建了`$el`元素。所以如果`vnode`是一个节点组件，其`componentInstance`属性应该有值，进入下一步。否则结束。即，在`createElm`中，不是组件节点的话返回值为假。
5. 调用[initComponent](#initComponent)，对组件做一些初始化的工作，包括设置`vnode.elm`，调用`create`钩子，`setScope`等。
6. 插入元素到指定位置。
7. 如果节点属于再次激活，调用`reactivateComponent`做激活处理。
8. 返回`true`。
9. 完毕。
### initComponent
#### 作用
初始化组件，包括设置`vnode.elm`，调用`create`钩子，`setScope`等。
#### 过程
1. 如果`vnode.data.pendingInsert`有值，将其放到`insertedVnodeQueue`中，并将原来的值设为`null`。`pendingInsert`，它存放着组件子组件待调用的`insert`钩子，在[invokeInsertHook](#invokeInsertHook)中，如果当前节点有父节点`parent`，设置`parent.data.pendingInsert`为`insertedVnodeQueue`，也就是说`insert`钩子函数会在根节点`patch`完成后统一触发。
2. 设置`vnode.elm`为节点对应实例的挂载元素。
3. 接着判定`isPatchable(vnode)`的值，即当前节点是否能`patch`，这个函数和如何判定参考下文。如果可以：
   1. 调用[invokeCreateHooks](#invokeCreateHooks)，触发`create`相关钩子，进行属性赋值和事件监听等工作。
   2. 调用`setScope`，设置`css scope`相关。
4. 如果不可以：
   1. - [ ] 这里有[issue#3455]()。在空的根节点中跳过属性更新，除了`ref`。调用[registerRef]()，它的作用是更新节点在父组件中的`ref`引用。
   2. 将当前`vnode`放入待调用`insert`钩子的队列中。
### insert
#### 作用 && 过程
将元素插入指定位置，没有父元素什么都不做，只有父元素将其插入到最后，有父元素和关联元素且关联元素的父元素也是这个父元素，将其插入到关联元素之前。

### createChildren
#### 作用 && 过程
对`vnode`的子节点创建元素，如果`children`是一个数组，非生产环境下检查`key`值是否有重复，接着依次调用`createElm`。否则判断其`text`属性是否是原始值，如果是，说明这是一文本节点，创建文本节点并插入到元素后。