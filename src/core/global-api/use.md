---
vue-note-completed: true
---
# core/global-api/use.js
## 作用
给`Vue`添加`use`方法，用来安装插件。这个插件通常会注册一些指令或者组件，也可能在原型上添加方法供所有实例使用。
## 使用
```javascript
Vue.use(plugin, options)
```
## 解析
首先引入了[toArray](../util/index.md#toArray)方法，做类数组到数组的转换。<br>
接着定义函数`initUse`，它接受实现了`GlobalAPI`接口的Vue参数，此参数其它地方不再另行说明。在上面定义`use`方法。<br>
### 参数
- `plugin` 要安装的插件，可以是一个函数，也可以是实现了`install`方法的对象。
### 返回
- 返回自身
### 过程
1. 获取已安装的插件列表`this._installedPlugins`，如果没有此属性则在这里初始化为空数组。
2. 如果此插件已安装直接返回`this`。否则进入下一步。
3. 使用`const args = toArray(arguments, 1)`来处理额外参数，也就是通常我们使用的`Vue.use(plugin, options)`中的`options`。然后将自身也就是`Vue`也加入到`args`中，最后传入`plugin`或者`plugin.isntall`并调用。
4. 将此插件放入已安装插件列表中。
5. 返回`this`。
## 问题
1. 处理额外参数只有这一种办法吗？这里为什么不用`options`做可选参数？是为了不限定参数数量吗？
## 代码
```javascript
/* @flow */

import { toArray } from '../util/index'

export function initUse (Vue: GlobalAPI) {
  Vue.use = function (plugin: Function | Object) {
    const installedPlugins = (this._installedPlugins || (this._installedPlugins = []))
    if (installedPlugins.indexOf(plugin) > -1) {
      return this
    }

    // additional parameters
    const args = toArray(arguments, 1)
    args.unshift(this)
    if (typeof plugin.install === 'function') {
      plugin.install.apply(plugin, args)
    } else if (typeof plugin === 'function') {
      plugin.apply(null, args)
    }
    installedPlugins.push(plugin)
    return this
  }
}

```