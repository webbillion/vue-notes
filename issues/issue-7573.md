[issue](https://github.com/vuejs/vue/issues/7573)

[commit](https://github.com/vuejs/vue/commit/318f29fcdf3372ff57a09be6d1dc595d14c92e70)

[笔记引用](/src/core/instance/state.md)
## 问题描述
在获取`data`的过程中，如果用到了`props`，会导致`props`第一次更新时触发两次组件的`update`生命周期。
```javascript
let Child = {
	name: 'child',
  template: '<div><span>{{ localMsg }}</span><button @click="change">click</button></div>',
  data: function() {
  	return {
    	localMsg: this.msg
    }
  },
  props: {
  	msg: String
  },
  methods: {
  	change() {
    	this.$emit('update:msg', 'world')
    }
  }
}

new Vue({
	el: '#app',
  template: '<child :msg.sync="msg"></child>',
  beforeUpdate() {
  	alert('update twice')
  },
  data() {
  	return {
    	msg: 'hello'
    }
  },
  components: {
  	Child
  }
})
```
第一次点击按钮时会触发两次`beforeUpdate`，实际上我们期待是只触发一次。
修复之后不仅在获取`data`时禁用了依赖收集，同时在所有生命周期钩子内都禁用了。
## 修改之前
### 分析
```javascript
  try {
    // 这个时候，以上文为例，在初始化Child组件时，Dep.target因为没有被置空，还是父组件的update watcher
    // 表达式为function () { vm._update(vm._render(), hydrating);
    // getter 为 updateComponent
    // 因为在data函数内访问了`this.props`，会将这个`watcher`放入对应的deps中。
    // 所以设置props时会触发两次beforeUpdate
    // 可以打印出这个watcher.deps，发现里面由有两个依赖
    // 但是为什么会只有第一次设置的时候会触发两次呢？
    // 是因为触发更新时会重新收集依赖并清除不再使用的依赖，
    // 详情请看相关源码
    return data.call(vm, vm)
  } catch (e) {
    handleError(e, vm, `data()`)
    return {}
  }
```
## 修改之后
有趣一点是这个问题在`2018-02-03`才被修复，可能是因为用到`beforeUpdate`的时候太少了吧。
### 分析
```javascript
// 改版之后pushTarget可以不传入参数，将Dep.target置为空
 pushTarget()
  try {
    return data.call(vm, vm)
  } catch (e) {
    handleError(e, vm, `data()`)
    return {}
  } finally {
    popTarget()
  }
```