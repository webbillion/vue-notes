# 前置
## hooks
= `['create', 'activate', 'update', 'remove', 'destroy']`，在更新或移除`vnode`时调用的钩子名称列表。
## samveVnode
### 作用
判断两个节点是否是同类型的。如果是，则可以复用。
### 参数
- `a` 旧节点
- `b` 新节点
### 过程
满足以下要求即返回`true`，否则`false`。
1. `key`值相同。
2. 以下两个条件之一。
   1. 普通情况下，标签相同 && 是否是注释节点的值相同 && 两个节点的`data`值是否定义相同 && `sameInputType(a, b)`的值为真，详情见下文。
   2. 是异步组件时， 旧节点是异步组件占位节点 && 两个节点的工厂函数相同 && 新节点没有加载失败。
## sameInputType
### 作用 && 过程
在节点标签是`input`时增加对它的`type`的判断。`type`相同或者都是文本类`input`则返回`true`。
## createKeyToOldIdx
### 作用 && 过程
如果子节点有`key`值，如：
```javascript
[
  {
    key: 'first'
  },
  {
    key: 'second'
  }
]
```
创建一个`key`到`index`索引的映射，即：
```javascript
{
  'first': 0,
  'second': 1
}
```
## 代码
```javascript
const hooks = ['create', 'activate', 'update', 'remove', 'destroy']

function sameVnode (a, b) {
  return (
    a.key === b.key && (
      (
        a.tag === b.tag &&
        a.isComment === b.isComment &&
        isDef(a.data) === isDef(b.data) &&
        sameInputType(a, b)
      ) || (
        isTrue(a.isAsyncPlaceholder) &&
        a.asyncFactory === b.asyncFactory &&
        isUndef(b.asyncFactory.error)
      )
    )
  )
}

function sameInputType (a, b) {
  if (a.tag !== 'input') return true
  let i
  const typeA = isDef(i = a.data) && isDef(i = i.attrs) && i.type
  const typeB = isDef(i = b.data) && isDef(i = i.attrs) && i.type
  return typeA === typeB || isTextInputType(typeA) && isTextInputType(typeB)
}

function createKeyToOldIdx (children, beginIdx, endIdx) {
  let i, key
  const map = {}
  for (i = beginIdx; i <= endIdx; ++i) {
    key = children[i].key
    if (isDef(key)) map[key] = i
  }
  return map
}
```