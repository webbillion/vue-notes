---
vue-note-completed: true
---
# core/instance/index.js
## 作用
这里是Vue构造函数的出生地。
## 解析
1. 引入一些混合函数。
2. 定义Vue函数:
   - 接受`options`参数。
   - 不是生产环境且没有使用`new`关键字，给出警告。以后类似警告不再说明。
   - 调用初始化函数。这里的`_init`方法是在[initMixin](./init.md)中添加的。
3. 调用混合函数`*Mixin(Vue)`对Vue的原型进行改造，它们是：
   - `initMixin` 添加[_init](./init.md)方法，进行初始化。
   - `stateMixin`  添加`$props`、`$data`、`$watch`等属性和方法。用于`props`和`state`以及`watch`等。[详情](./state.md)。
   - `eventsMixin` 添加`$on`、`$off`、`$once`等方法，用于事件处理。[详情](./events.md)。
   - `lifecycleMixin` 添加`$destory`等和生命周期相关的方法。[详情](./lifecycle.md)。
   - `renderMixin` 添加`$nextTick`等和渲染相关的方法。[详情](./render.md)。
## 代码
```javascript
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'

function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)

export default Vue

```