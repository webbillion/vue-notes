# core/instance/render-helpers/index.js
从其它文件引入方法，给`Vue`的原型添加以`_`开头的一些方法，它们主要在编译模板时使用。这里简单介绍以下在本文件夹下的，其它文件夹的请到对应文件查看，它们分别是：
- `_o` = [markOnce](./render-static.md)，用于`v-once`指令，标记对应的`vnode`。
- `_l` = [render-list](./render-list.md)，用于`v-for`指令，渲染一个列表。
- `_t` = [renderSlot](./render-slot.md)，用来渲染`slot`。
- `_m` = [renderStatic](./render-static.md)，渲染一个静态树，就是用于渲染没有绑定属性的元素。
- `_f` = [resolveFilter](./resolve-filter.md)， 解析`filter`。
- `_k` = [checkKeyCodes](./check-keycodes.md)，检查事件修饰符`keycode`是否存在。
- `_b` = [bindObjectProps](./bind-object-props.md)，将`v-bind`指令里绑定的对象放到`vnode`的`data`里。
- `_u` = [resolveScopedSlots](./resolve-slots.md)，解析那些使用了`slot-scope=""`属性的`slot`。
- `_g` = [bindObjectListeners](./bind-object-listeners.md)，将`v-on`指令里绑定的函数放到`vnode`的`data.on`中。
