---
vue-note-completed: true
---
# core/global-api/mixin.js
## 作用
为`Vue`添加`mixin`方法，用来全局注册一个混入。
- [ ] 关于混入，此处不详解。
## 使用
```javascript
Vue.mixin({
  created() {
    // do something
  }
})
```
## 解析
将传入的混入用`mergeOptions`方法合并到`options`上，没有了。
```javascript
/* @flow */

import { mergeOptions } from '../util/index'

export function initMixin (Vue: GlobalAPI) {
  Vue.mixin = function (mixin: Object) {
    this.options = mergeOptions(this.options, mixin)
    return this
  }
}

```