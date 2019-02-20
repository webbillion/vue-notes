# core/observer/watcher.js
## Watcher
### 属性
- `vm` 绑定的组件实例
- `expression` 在非生产环境下是函数或表达式的字符串形式，用来。。
- `cb` 依赖变动后要调用的函数，在我们调用`$watch`时传入的函数就是这个。
- `id`
- `deep` 是否深度观测，在`watch`函数中会用到。
- `user` 是否是用户定义的`watcher`。
- `lazy` 是否是惰性求值，是的话在初始化的时候不会调用`get`。
- `sync` 依赖变动后是否同步更新，测试的时候使用。
- `dirty` 是否脏检查，在计算属性中，会根据它的值决定是否重新调用`get`。
- `active`
- `deps` 依赖了哪些属性。
- `newDeps` 重新收集依赖时新的依赖属性，用来和旧的对比，移除不再依赖的属性。
- `depIds`
- `newDepIds`
- `before` 在调用`cb`前调用的函数，可选。被使用来触发`update`。
- `getter` 求值的函数，如`render`。