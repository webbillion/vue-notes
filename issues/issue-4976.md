[issue](https://github.com/vuejs/vue/issues/4976)

[commit](https://github.com/vuejs/vue/commit/8e854a9ed1b606890b53637f201432174bb7508a)

[笔记引用](/src/core/instance/init.md)
## 问题描述
之前的处理会丢弃在后期构造函数中注入的选项，导致使用`vue-class-component`时无法热重载，也无法使用`CSS Modules`。意思是在`Test = Vue.extend()`后，再对`Test.options`进行修改，实例化`Test`时新增的一部分不会生效。
```javascript
const Test = Vue.extend({
  a: 1
})
// Test.options { a: 1 }
Test.options.beforeCreate = [
  () => console.log('beforeCreate')
]
// 更新选项
Test.mixin({})
console.log(Test.options.beforeCreate) // [f]
new Vue({
  render: h => h(Test)
}).$mount('#app')
console.log(Test.options.a) // 1
console.log(Test.options.beforeCreate) // undefined
```
- [ ] 至于它和热重载之间的关系，还不是很了解。
## 修改之前
### 分析
```javascript
if (Ctor.super) {
  const superOptions = Ctor.super.options
  const cachedSuperOptions = Ctor.superOptions
  const extendOptions = Ctor.extendOptions
    if (superOptions !== cachedSuperOptions) {
      Ctor.superOptions = superOptions
      extendOptions.render = options.render
      extendOptions.staticRenderFns = options.staticRenderFns
      extendOptions._scopeId = options._scopeId
      // 可以看到，在这一步里将options置为superOptions和extendOptions合并的选项了
      // 而后续添加的属性是保存在super.options中的，所以给移除了
      options = Ctor.options = mergeOptions(superOptions, extendOptions)
    }
}
```
## 修改之后
### 分析
```javascript
if (Ctor.super) {
    // 所以这里不再是直接获取选项，而是调用这个函数来获取
    const superOptions = resolveConstructorOptions(Ctor.super)
    const cachedSuperOptions = Ctor.superOptions
    if (superOptions !== cachedSuperOptions) {
      // super option changed,
      // need to resolve new options.
      Ctor.superOptions = superOptions
      // check if there are any late-modified/attached options (#4976)
      const modifiedOptions = resolveModifiedOptions(Ctor)
      // update base extend options
      if (modifiedOptions) {
        extend(Ctor.extendOptions, modifiedOptions)
      }
      options = Ctor.options = mergeOptions(superOptions, Ctor.extendOptions)
      if (options.name) {
        options.components[options.name] = Ctor
      }
    }
  }
```