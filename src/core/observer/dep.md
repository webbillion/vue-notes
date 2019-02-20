# core/observer/dep.js
## Dep
### 作用
为响应式属性创建一个依赖，能收集当前`watcher`，并在属性被设置时通知`watcher`。
### 属性
- `target` 静态`static`属性，存放当前需要收集依赖的`watcher`。
- `id` `uid++`得到。
- `subs` 订阅者数组，一个`watcher`组成的数组，在属性被设置时依次通知其中的`watcher`。
### 方法
#### addSub
向`subs`中添加一个订阅者`watcher`。
#### removeSub
移除指定的订阅者`watcher`。
#### depend
向`target`，即正在收集依赖的`watcher`，添加自身作为其依赖之一。
#### notify
依次调用订阅者`watcher`的`update`方法。需要注意的一点是，在非生产环境中，而且不是异步，要对订阅者做一个关于`id`大小的排序，这是为了它们能在同步时以正确的顺序触发。因为没有异步，调度程序`scheduler`就不会对其排序。