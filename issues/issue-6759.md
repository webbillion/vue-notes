[issue](https://github.com/vuejs/vue/issues/6759)

[commit](https://github.com/vuejs/vue/commit/405d8e9f4c3201db2ae0e397d9191d9b94edc219)

[笔记引用](/src/core/instance/lifecycle.md)
## 问题描述
摧毁实例后，没有摧毁所有相关数据，导致内存泄露。
## 修改之前
### 分析
```javascript
// vm.$vnode还保存着对parent的引用。就不会被回收。
// 所以会造成内存泄露。
// TODO:暂时还不知道什么不把vm的所有属性直接值空。
```
## 修改之后

### 分析
```javascript
if (vm.$vnode) {
  // 置为null后会被自动回收
  vm.$vnode.parent = null
}
```