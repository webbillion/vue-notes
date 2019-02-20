# 一些问题
在本章中遇到了很多`parent`，在这里做一些简单的总结。
- `vm` 组件实例，即由`new Vue`创建。
- `vnode` 虚拟节点实例，由`new Vnode`创建。
- `vm.$parent` 父组件。
- `vm._vnode` 组件自身渲染出的`vnode`。
- `vnode.parent` 节点在父组件渲染树中的节点。**不等同于**父组件的`vnode`。
- `vm.$vnode` 和`vm._vnode.parent`相同。