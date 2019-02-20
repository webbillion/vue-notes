# 关于`Vnode`和`Virtual Dom`
这一文件夹下构造了`virtual dom`系统，它由`Snabbdom`改造而来，关于`Snabbdom`和`virtual dom`，详细可以看[vue2源码学习开胃菜——snabbdom源码学习](https://segmentfault.com/a/1190000009017324)。下面做一个简单的介绍。
## [VNode](./vnode.md)
真实的`DOM`节点拥有非常多的属性，完整更新或添加`DOM`的开销是非常大的，但实际使用和在`Vue`渲染时我们用不到那么多不同的属性，就抽离出用到的属性，用`js`的方式创建`DOM`树，如：
```html
<div class="s">
  <p title="p1">p1</p>
  <p>p2</p>
</div>
```
将被解析成：
```javascript
node = {
  // 虚拟节点关联的真实`Dom`节点
  el: div,
  // 节点类型
  type: 1,
  // 节点标签
  tag: 'div',
  // 静态类
  staticClass: 's',
  // 子节点
  children: [
    {
      type: 1,
      tag: 'p',
      //属性列表
      attrsList: [
        {
          name: 'title',
          value: 'p1'
        }
      ],
      children: [
        {
          type: 3,
          text: 'p1',
          static: true
        }
      ]
    }
    // ...等等
  ]
}
```
`VNode`和示例代码中的`node`差不多，只不过拥有更多的属性。但它其实一点都不复杂，复杂的是用到它的那些方法。
## Virtual Dom
`Virtual Dom`，即虚拟`Dom`，由`VNode`构成的节点树，其节点和真实`Dom`节点一一对应。将`VNode`渲染成真实`Node`这个过程叫做`patch`。
## patch
它对比节点前后变化，决定是创建新节点，还是移除或者对原来的节点进行修改。

在`patch`之前有几个点需要说明。
### nodeOps
虚拟`Dom`是与平台无关的，也就是说如何渲染成怎样的`node`是由平台提供的`api`决定的，即`nodeOps`（或者说叫`DOMApi`），它包括创建元素(`createElement`) 移除元素(`removeElement`)等`api`。
### modules
里面有`attrs` `class`等模块，用来更新或创建元素的属性，以及事件监听等内容。比如说，将新节点的`id`属性的值赋给旧节点关联的`el`。它通常在`create`和`update`钩子中被调用。
### patchVnode
调用`modules`中的函数进行更新。
### hook
在进行创建元素或移除元素等之前或之后要调用的钩子，它包含在`vnode.data.hook`中，进行准备工作或者收尾工作。
### 复用
如果每次节点更新都删除原来的元素，创建新的元素，开销会比较大，所以对于能复用（这里暂且理解为标签相同）的元素，修改其属性，来达到优化性能的目的 —— 虽然都这么说，但是确实如此吗？经过实际测试，发现还真是如此。
```javascript
let num = 10000
// 复用
for (let i = 0; i < num; i += 1) {
    let div = document.createElement('div')
    div.style = 'width: 10px;height: 20px;background-color: #f00;'
    div.innerHTML = '<div>所发生的<p>fsdfds<p></div>'
    document.body.appendChild(div)
}
// 延时是为了等待之前创建的元素渲染完毕
setTimeout(() => {
    var divs = document.querySelectorAll('div')
    console.time('f1')
    for (let i = 0; i < num; i += 1) {
        let div = divs[i]
        div.style = 'width: 30px;height: 10px;background-color: #0f0;'
        div.innerHTML = '<div><h3>的发射点</h3></div>'
    }
    console.timeEnd('f1')
}, 10000)
// 不复用
// 。。这里是创建div的步骤
setTimeout(() => {
  var divs = document.querySelectorAll('div')
    console.time('f3')
    for (let i = 0; i < num; i += 1) {
        let div = divs[i]
        div.remove()
        div = document.createElement('div')
        div.style = 'width: 30px;height: 10px;background-color: #0f0;'
        // div.innerHTML = '<div><h3>的发射点</h3></div>'
        document.body.appendChild(div)
    }
    console.timeEnd('f3')  
}, 10000)
// 复用的话确实时间更短
// 如果去掉div.innerHTML这一段，差距会更加明显
```
### 大概的流程如下：
1. 它接受新旧两个节点作为参数。
2. 对比它是否是同类型的（如何以及为什么请看对应章节），不是的话判定为不能复用，进入第3步，否则进入第4步。
3. 不能复用就重新创建(`nodeOps.createElement`)，但是创建之前要调用`hook.init`钩子，创建元素后返回。
4. 能复用进行`patchVnode`，并对其子节点进行比较。                                  
5. 比较子节点这一步叫做`diff`，详情请看对应章节，目的是找出子节点中能复用的节点，对其`patchVnode`，否则移除旧节点或者创建新节点。
6. 完毕。

比如：
```html
<!-- 旧 -->
<div id="div1">
  <p id="p1"></p>
  <span id="span1"></span>
</div>
<!-- 新 -->
<div id="div2">
  <img id="img1">
  <p id="p2"></p>
</div>
```
1. 对比`div1`和`div2`，它们都是`div`，可以复用。
2. `patchVnode`：`div1.id = div2.id`。
3. 对子节点`[p1, span1]`和`[img1, p2]`进行`diff`比较。
   1. `p1`能复用，进行`patchVnode`: `p1.id = p2.id`，没有子节点，完毕。
   2. `img1`没有可复用的，创建元素。
   3. `span1`没有被处理到，说明在新节点中被删除，移除`span1`。
4. 完毕。

整个`Virtual Dom`和`patch`流程大致如此，可以进入源码细细分析了。
## 完整的流程
在进入下面学习之前，我觉得有必要理一下整个过程。
下面展示一个组件从定义到渲染的全过程。
1. 编译
   1. 将模板(`template`)解析成语法树(`AST`)。详情看[这里]()。
    ```javascript
    let Child = {
      name: 'child',
      template: '<div ><span>{{ localMsg }}</span><button @click="change">click</button></div>'
      // 其它省略
    }
    // 会解析成如下`ast`，为了篇幅这里使用压缩过的json
    let ast = {"type":1,"tag":"div","attrsList":[],"attrsMap":{},"children":[{"type":1,"tag":"child","attrsList":[{"name":":msg.sync","value":"msg"},{"name":"title","value":"title"}],"attrsMap":{":msg.sync":"msg","title":"title"},"parent":"[循环引用]","children":[],"plain":false,"hasBindings":true,"events":{"update:msg":{"value":"msg=$event"}},"attrs":[{"name":"msg","value":"msg"},{"name":"title","value":"\"title\""}],"static":false,"staticRoot":false}],"plain":true,"static":false,"staticRoot":false}
    ```
    2. 解析语法树生成渲染函数`render`。它会用到[$createElement]()创建`VNode`并返回，当然其中也用到了一些[辅助函数]()，比如`_l`处理`v-for`渲染一个列表。详情看[这里]()。上面模板的渲染函数如下。
    ```javascript
    // 这里使用的是vue-loader生成的
    function() {
      var _vm = this
      var _h = _vm.$createElement
      var _c = _vm._self._c || _h
      return _c(
        // tag
        "div",
        // children
        [
          // tag
          _c("child", { // data，即`VNodeData`
            // 属性
            attrs: { msg: _vm.msg, title: "title" },
            // 事件
            on: {
              "update:msg": function($event) {
                _vm.msg = $event
              }
            }
          })
        ],
        1
      )
    }
    ```
2. 创建实例并挂载
   1. 状态解析那些都跳过。
   2. 进入`mount`，实际上是调用了`vm._update(vm._render)`。在`vm._update`里调用了`vm.__patch__`来比较前后节点的不同并映射到真实`DOM`上。
```javascript
a = new Vue({
	el: '#app',
  template: '<div><child :msg.sync="msg" title="title"></child></div>',
  data() {
  	return {
    	msg: 'hello'
    }
  },
  components: {
  	Child
  },
  mounted() {
  	console.log('root mounted')
  }
})
```