---
vue-note-completed: true
---
# core/global-api/assets.js
### 作用
为Vue添加`component`等资源方法。
### 使用
```javascript
Vue.component(Component)
```
### 解析
遍历`ASSETS_TYPES`数组中的值，即`component`、`directive`、`filter`，为`Vue`添加对应方法。
#### 参数
- `id` 对应id，要求是字符串
- `definition` 可选，对应定义，要求是对象或者函数
#### 返回
- 对象或者函数或者什么都不返回
#### 过程
1. 只传入`id`则返回选项中对应的值。否则进入下一步。
2. 非生产环境和函数是`component`时验证组件名称。
3. 接着验证两种种情况：
  - 是组件且组件定义是纯对象，即`toString`返回[Object Object]的对象，接着定义组件名称，然后调用`this.options._base.extend`扩展，这个`_base`在[global-api/index.js](./index.md)提到过，就是`Vue`。这里为什么用`._base`而不是`this`呢？应该是因为`component`这个方法是可以在扩展后的子类上调用的，如果用`this`，则是在此基础上再扩展。参考[extend](./extend.md)
  - 是指令且指令是一个函数，这是注册指令的一种方法，即只传入函数，则将指令的`bind`和`update`都当做这个函数。
4. 将定义传入`options`对应属性中。
5. 返回定义。
### 代码
```javascript
/* @flow */

import { ASSET_TYPES } from 'shared/constants'
import { isPlainObject, validateComponentName } from '../util/index'

export function initAssetRegisters (Vue: GlobalAPI) {
  /**
   * Create asset registration methods.
   */
  ASSET_TYPES.forEach(type => {
    Vue[type] = function (
      id: string,
      definition: Function | Object
    ): Function | Object | void {
      if (!definition) {
        return this.options[type + 's'][id]
      } else {
        /* istanbul ignore if */
        if (process.env.NODE_ENV !== 'production' && type === 'component') {
          validateComponentName(id)
        }
        if (type === 'component' && isPlainObject(definition)) {
          definition.name = definition.name || id
          definition = this.options._base.extend(definition)
        }
        if (type === 'directive' && typeof definition === 'function') {
          definition = { bind: definition, update: definition }
        }
        this.options[type + 's'][id] = definition
        return definition
      }
    }
  })
}

```