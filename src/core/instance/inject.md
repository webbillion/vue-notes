---
vue-note-completed: true
---
# core/instance/inject.js
这里主要导出了几个函数用来初始化和解析`inject`和`provide`选项。
## initProvide
### 作用 && 选项
将`provide`选项挂载到`vm._privided`上，如果`provide`是一个函数，就将`vm`作为上下文并调用它。
## initInjections
### 作用
将`provided`中的数据解析到实例上，并将其定义为响应式的。
### 过程
1. 调用`resolveInject`获取解析之后的结果`result`。
2. 如果有的话，调用`toggleObserving(false)`，这是为了只观测`result`的第一层数据。
3. 遍历`result`获取其`key`，将`vm`的`key`属性定义为响应式的，值就是`result[key]`。如果是非生产环境，调用`defineReactive`时传入第四个参数，在设置`inject`中的属性时给出警告。
4. 重新打开检测开关。
5. 完毕。
## resolveInject
### 作用
根据`inject`选项从实例的`_provided`中取出出局。
### 过程
1. 如果`inject`不存在，什么也不做。
2. 定义`result`为没有继承的空对象，它就是要返回的结果。这里的注释是关于`flow`的，略过不提。
3. 定义`keys`为`inject`的`key`数组，这里需要注意的是，如果`hasSymbol`为真，即当前环境支持`Reflect`和`Symbol`，调用`Reflect.ownKeys`并过滤，只获取其可枚举的属性。否则调用`Object.keys`。区别之一是`Reflect.ownKeys`可以获取`Symbol`的属性名。但`Object.getOwnPropertySymbols`同样可以做到这一点，至于为什么用`Reflect`而且在没有`Reflect`时不调用`Object.getOwnPropertySymbols`。前者可能是为了逐渐向`es6`靠拢，后者就不太清楚了。
4. 接着从逐层遍历`vm`和其父元素，直到找到当前`provideKey` = `inject[key].form`在`vm`的`_provided`中有定义为止，注意这里的定义是有这个属性即可，无论其值真假，并将其赋值给`result[key]`。
5. 如果找遍了都没有，在`inject[key]`中寻找`default`选项，是函数就调用它，负责直接赋值给`result[key]`。
6. 如果也没有`default`选项，非生产环境下给出警告。
7. 返回`result`。
8. 完毕。
## 代码
```javascript
/* @flow */

import { hasOwn } from 'shared/util'
import { warn, hasSymbol } from '../util/index'
import { defineReactive, toggleObserving } from '../observer/index'

export function initProvide (vm: Component) {
  const provide = vm.$options.provide
  if (provide) {
    vm._provided = typeof provide === 'function'
      ? provide.call(vm)
      : provide
  }
}

export function initInjections (vm: Component) {
  const result = resolveInject(vm.$options.inject, vm)
  if (result) {
    toggleObserving(false)
    Object.keys(result).forEach(key => {
      /* istanbul ignore else */
      if (process.env.NODE_ENV !== 'production') {
        defineReactive(vm, key, result[key], () => {
          warn(
            `Avoid mutating an injected value directly since the changes will be ` +
            `overwritten whenever the provided component re-renders. ` +
            `injection being mutated: "${key}"`,
            vm
          )
        })
      } else {
        defineReactive(vm, key, result[key])
      }
    })
    toggleObserving(true)
  }
}

export function resolveInject (inject: any, vm: Component): ?Object {
  if (inject) {
    // inject is :any because flow is not smart enough to figure out cached
    const result = Object.create(null)
    const keys = hasSymbol
      ? Reflect.ownKeys(inject).filter(key => {
        /* istanbul ignore next */
        return Object.getOwnPropertyDescriptor(inject, key).enumerable
      })
      : Object.keys(inject)

    for (let i = 0; i < keys.length; i++) {
      const key = keys[i]
      const provideKey = inject[key].from
      let source = vm
      while (source) {
        if (source._provided && hasOwn(source._provided, provideKey)) {
          result[key] = source._provided[provideKey]
          break
        }
        source = source.$parent
      }
      if (!source) {
        if ('default' in inject[key]) {
          const provideDefault = inject[key].default
          result[key] = typeof provideDefault === 'function'
            ? provideDefault.call(vm)
            : provideDefault
        } else if (process.env.NODE_ENV !== 'production') {
          warn(`Injection "${key}" not found`, vm)
        }
      }
    }
    return result
  }
}
```