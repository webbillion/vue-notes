---
vue-note-completed: true
---
# core/index.js
## 作用
这是Vue的核心代码，应该是对Vue构造函数最后的处理,其它地方使用的Vue都是从这儿导出的<br>
## 解析
在文件中主要做了以下几点：
- 引入Vue构造函数，参考[instance/index.js](./instance/index.md)。
- `initGlobalAPI(Vue)` 对Vue构造函数添加了一些全局的API。具体参见[global-api/index.js](./global-api/index.md)。
- 在Vue和它的prototype上添加了一些属性，具体有
  - `$isServer` 只读，是否是服务端渲染，`get`方法被设置为[isServerRendering](./util/env.md#isServerRendering)。
  - `$ssrContent` 只读，服务端渲染上下文，`get`方法被设置为
    ```javascript
    get() {
      return this.$vnode && this.$vnode.ssrContext
    }
    ```
    - [ ] 这个`$vnode`在哪儿，以及有什么用。
  - `FunctionalRenderContext` 函数式渲染上下文，和`vdom`有关。`value`被设置为[FunctionalRenderContext](./vdom/create-functional-component)。
  - `version` 版本，这里只有字符串`__VERSION__`，没有版本号。
- 导出Vue构造函数。
- 完毕。
## 思考
我的用词现在还不太专业哈，对设计模式懂得不多。从这里以及后面一些代码来看，采用了对Vue构造函数和它的原型不断添加属性和混入方法的方式来添加功能。应该是为了解耦对功能进行分块。
1. 这些功能是如何分割的？它们之间如何相互联系？
2. 这是一种什么设计模式。
## 代码
```javascript
import Vue from './instance/index'
import { initGlobalAPI } from './global-api/index'
import { isServerRendering } from 'core/util/env'
import { FunctionalRenderContext } from 'core/vdom/create-functional-component'

initGlobalAPI(Vue)

Object.defineProperty(Vue.prototype, '$isServer', {
  get: isServerRendering
})

Object.defineProperty(Vue.prototype, '$ssrContext', {
  get () {
    /* istanbul ignore next */
    return this.$vnode && this.$vnode.ssrContext
  }
})

// expose FunctionalRenderContext for ssr runtime helper installation
Object.defineProperty(Vue, 'FunctionalRenderContext', {
  value: FunctionalRenderContext
})

Vue.version = '__VERSION__'

export default Vue

```