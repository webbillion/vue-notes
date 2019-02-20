---
vue-note-completed: true
---
# core/global-api/index.js
## 作用
为`Vue`添加全局方法和配置。
## 解析
这里导出了`initGlobalAPI`函数，它主要做了：
- 在Vue上添加**只读**`config`，它来自[config](../config.md)
- 添加一系列方法和选项，它们是
  - `util` 工具方法，一些在源码中经常用到的函数，如`warn`输出警告，作者在这里注释避免依赖这些方法，除非清楚这些风险。我理解为最好别用。
  - `set` 来自[set](../observer/index.md#set)，向响应式对象中添加一个属性，并确保这个新属性同样是响应式的，因为 Vue 无法探测普通的新增属性。我阅读的版本是2.5.17，作者提到在3.0版本会用`Proxy`来代替`DefineProperty` 这个方法可能会没什么用了。此外响应式相关应该会有较大改动吧，目前也不知道是否细读这一部分。
  - `delete`来自[del](../observer/index.md#del)，同上，只不过用来删除。
  - `nextTick` 来自[nextTick](../util/next-tick)，修改数据之后使用这个方法来获取更新后的DOM。
  - `options` 选项，这里是使用`Object.create(null)`来创建`options`的，其它地方好像也有多处用到，这句代码创建了一个没有任何属性的对象：
    > 那些属性Vue自己重写了一份，原因可能是为了代码风格的统一（函数式编程）。
  —— 来自[夜未央](https://segmentfault.com/u/yeweiyang_58bd873985849)的回答。

接着对`options`做了一些操作：
1. 在`options`上添加了`ASSET_TYPES`数组内存在的对象，即资源类型，它们定义于`shared/constanst.js` 包括`component`、`directive`、`filter`三个值，即组件、指令、过滤器。 
   - [x] 它们究竟是做什么的，有待后续探究。
2. 又在`options`上添加了`_base=Vue` 这里有注释，意思是这用于标识“基础”构造函数，以在Weex的多实例场景中扩展所有纯对象组件。
3. 用`extend`（此方法即用来扩展对象属性的）将`builtInComponents`扩展到`options.component`上，这个`builtInComponents`定义于`core/components/index.js` 包含了一个[KeepAlive](../components/keep-alive.md)的东西，看到这里，前面的`options.components`应该就是内置组件了，其它选项也是内置的。猜想其它组件和指令也会注册到这里。

下面也是给Vue添加一些方法，不过使用`init*(Vue)`的方式，而不是像`set`方法一样直接添加。
- [ ] 这里我猜测是下面这些方法里用到里`this`，真正的原因还有待探索。

它们是：
- `initUse` 添加[use](./use.md)方法，注册插件和组件。
- `initMixin` 添加[mixin](./mixin.md)方法，全局混入。
- `initExtend` 添加[extend](./extend.md)，创建子类。
- `initAssetRegisters`, 添加一些注册资源的方法，即`ASSET_TYPES`中包含的:
   - `component` 注册或获取全局组件。
   - `directive` 注册或获取全局指令。
   - `filter` 注册或获取全局过滤器。
   - [详情](./assets.md)
## 代码
```javascript
/* @flow */

import config from '../config'
import { initUse } from './use'
import { initMixin } from './mixin'
import { initExtend } from './extend'
import { initAssetRegisters } from './assets'
import { set, del } from '../observer/index'
import { ASSET_TYPES } from 'shared/constants'
import builtInComponents from '../components/index'

import {
  warn,
  extend,
  nextTick,
  mergeOptions,
  defineReactive
} from '../util/index'

export function initGlobalAPI (Vue: GlobalAPI) {
  // config
  const configDef = {}
  configDef.get = () => config
  if (process.env.NODE_ENV !== 'production') {
    configDef.set = () => {
      warn(
        'Do not replace the Vue.config object, set individual fields instead.'
      )
    }
  }
  Object.defineProperty(Vue, 'config', configDef)

  // exposed util methods.
  // NOTE: these are not considered part of the public API - avoid relying on
  // them unless you are aware of the risk.
  Vue.util = {
    warn,
    extend,
    mergeOptions,
    defineReactive
  }

  Vue.set = set
  Vue.delete = del
  Vue.nextTick = nextTick

  Vue.options = Object.create(null)
  ASSET_TYPES.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
  })

  // this is used to identify the "base" constructor to extend all plain-object
  // components with in Weex's multi-instance scenarios.
  Vue.options._base = Vue

  extend(Vue.options.components, builtInComponents)

  initUse(Vue)
  initMixin(Vue)
  initExtend(Vue)
  initAssetRegisters(Vue)
}

```