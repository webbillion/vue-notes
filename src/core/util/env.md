# core/util/env.js
一些环境判断的辅助函数。
## isServerRendering
判断是否是服务端渲染
```javascript
// this needs to be lazy-evaled because vue may be required before
// vue-server-renderer can set VUE_ENV
let _isServer
export const isServerRendering = () => {
  if (_isServer === undefined) {
    /* istanbul ignore if */
    if (!inBrowser && !inWeex && typeof global !== 'undefined') {
      // detect presence of vue-server-renderer and avoid
      // Webpack shimming the process
      _isServer = global['process'].env.VUE_ENV === 'server'
    } else {
      _isServer = false
    }
  }
  return _isServer
}
```
注释提示这个需要懒加载判断，因为可能在`vue-server-renderer`设置`VUE_ENV`之前就已经加载好Vue了。<br>
环境值只会赋值一次，再次调用直接返回。<br>
判断不是浏览器，且不是weex，是node环境下，才会设置为server，注释提示不要避免用webpack时修改这个环境变量。