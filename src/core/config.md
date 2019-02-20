---
vue-note-completed: true
---
# core/config.md
### 作用
导出了一个配置选项对象，包括合并策略等。
## 解析
有以下属性：
- `optionMergeStrategies` 对象，默认为无继承的`null`，选项合并策略，应该是[mergeOption]()中会用到的，如何处理选项合并，注释提到默认采用[core/util /options]()。
- `silent` 布尔值，默认为`false`，是否打印警告信息。猜测用在`warn`函数内部
- `productionTip` 布尔值, 值和环境相关, 是否展示一些提示。我注意到其它地方判断是否需要提示是通过`NODE_ENV`来判断的，而不是通过这个值。一个原因应该是那些函数是和`Vue`无关的。
  - [ ] 那么这个值会在哪里用到呢？
- `devtools` 布尔值，值和环境相关，用来决定是否开启`devtools`，也就是我们使用的调试工具在生产环境下是不可用的。应该是就没有暴露数据接口吧。
- `performance` 布尔值，默认为`false`，是否记录性能，即是否在渲染过程中调用`performance.mark`来记录性能。在[_init](./instance/init.md)和[_update](./instance/lifecycle.md)等中被使用。
- `errorHandler` 函数，默认为`null`，对`watcher`的错误处理函数。
- `warnHandler` 函数，默认为`null`，对`watcher`的警告处理函数。
- `ignoredElements` 字符串或者正则数组，默认为空，忽略自定义元素，即那些既不是html包含的，也不是定义组件的的元素名称，添加后不会报错。
- `keyCodes` 对象，默认为无继承的`null`，用来自定义`v-on`的键位别名。即`@click.enter`中的`enter`这种。

以下是平台相关判定的函数，默认为[no]()，即一个返回`false`的函数,或者[noop]()，即返回一个空对象的函数，或者[identity]()，即返回传入值的函数。

---

- `isReservedTag` `no`，检查是否是保留标签，这个会被平台依赖重写，就是在`web`平台能用的标签在其它平台不一定能用。
- `isReservedAttr` `no`，检查是否是保留属性，其它同上。
- `isUnknownElement` `no`，检查是否是未知元素，其它同上。
- `getTagNamespace` `noop`
  - [ ] 获取一个元素的命名空间？还不懂是干嘛的。
- `parsePlatformTagName` `identity` 在特定平台上获取真正的标签名，应该是`div`标签在`web`中就是`div`，在`weex`中可能有所不同。
- `mustUseProps` `no`，检查一个属性是否必须使用`props`来绑定，比如`value`。
---
- `async`  布尔值，默认为`true`，是否异步执行更新。平时使用时都是异步的，如果设置为同步性能会显著降低，这个值只有在写测试时才会改为`false`，更多请参考[Vue Test Utils]()。
- `_lifecycleHooks`: 来自[LIFECYCLE_HOOKS]()，生命周期钩子函数数组。注释说因为遗留原因。
## 代码
```javascript
/* @flow */

import {
  no,
  noop,
  identity
} from 'shared/util'

import { LIFECYCLE_HOOKS } from 'shared/constants'

export type Config = {
  // user
  optionMergeStrategies: { [key: string]: Function };
  silent: boolean;
  productionTip: boolean;
  performance: boolean;
  devtools: boolean;
  errorHandler: ?(err: Error, vm: Component, info: string) => void;
  warnHandler: ?(msg: string, vm: Component, trace: string) => void;
  ignoredElements: Array<string | RegExp>;
  keyCodes: { [key: string]: number | Array<number> };

  // platform
  isReservedTag: (x?: string) => boolean;
  isReservedAttr: (x?: string) => boolean;
  parsePlatformTagName: (x: string) => string;
  isUnknownElement: (x?: string) => boolean;
  getTagNamespace: (x?: string) => string | void;
  mustUseProp: (tag: string, type: ?string, name: string) => boolean;

  // private
  async: boolean;

  // legacy
  _lifecycleHooks: Array<string>;
};

export default ({
  /**
   * Option merge strategies (used in core/util/options)
   */
  // $flow-disable-line
  optionMergeStrategies: Object.create(null),

  /**
   * Whether to suppress warnings.
   */
  silent: false,

  /**
   * Show production mode tip message on boot?
   */
  productionTip: process.env.NODE_ENV !== 'production',

  /**
   * Whether to enable devtools
   */
  devtools: process.env.NODE_ENV !== 'production',

  /**
   * Whether to record perf
   */
  performance: false,

  /**
   * Error handler for watcher errors
   */
  errorHandler: null,

  /**
   * Warn handler for watcher warns
   */
  warnHandler: null,

  /**
   * Ignore certain custom elements
   */
  ignoredElements: [],

  /**
   * Custom user key aliases for v-on
   */
  // $flow-disable-line
  keyCodes: Object.create(null),

  /**
   * Check if a tag is reserved so that it cannot be registered as a
   * component. This is platform-dependent and may be overwritten.
   */
  isReservedTag: no,

  /**
   * Check if an attribute is reserved so that it cannot be used as a component
   * prop. This is platform-dependent and may be overwritten.
   */
  isReservedAttr: no,

  /**
   * Check if a tag is an unknown element.
   * Platform-dependent.
   */
  isUnknownElement: no,

  /**
   * Get the namespace of an element
   */
  getTagNamespace: noop,

  /**
   * Parse the real tag name for the specific platform.
   */
  parsePlatformTagName: identity,

  /**
   * Check if an attribute must be bound using property, e.g. value
   * Platform-dependent.
   */
  mustUseProp: no,

  /**
   * Perform updates asynchronously. Intended to be used by Vue Test Utils
   * This will significantly reduce performance if set to false.
   */
  async: true,

  /**
   * Exposed for legacy reasons
   */
  _lifecycleHooks: LIFECYCLE_HOOKS
}: Config)

```