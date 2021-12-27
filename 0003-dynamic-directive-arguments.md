# 摘要

指令参数支持动态值

# 例子

``` html
<div v-bind:[key]="value"></div>
<div v-on:[event]="handler"></div>
```

# 改动

由于指令参数是静态的，目前，用户不得不依赖无参构造函数的对象字面量的动态键：

``` html
<div v-bind="{ [key]: value }"></div>
<div v-on="{ [event]: handler }"></div>
```

但是这里会有几个问题:

- 这是一种鲜为人知的技术，它需要使用者掌握`v-for`/`v-on`的基于对象的语法和计算属性键存在的原理知识

- 低效率代码：临时分配到的一个对象，如果该元素有其他静态绑定，则必须动态迭代，然后将数据混入到现有数据中。代码大致如下：

  ``` js
  return h('div', {
    on: Object.assign({
      click: onClick
    }, {
      [event]: handler
    })
  })
  ```

  我们可以按照动态参数那样，直接生成：

  ``` js
  return h('div', {
    on: {
      click: onClick,
      [event]: handler
    }
  })
  ```

除此之外，`v-slot`没有对应的对象语法，因为它的值被用来声明插槽作用域变量。所以，如果没有动态参数，`v-slot`就不支持动态插槽名。尽管这是一个非常罕见的用例，但仅因这一个限制就需要重写整个模板为渲染函数是非常痛苦的。

# 详细设计

``` html
<!-- v-bind with dynamic key -->
<div v-bind:[key]="value"></div>

<!-- v-bind shorthand with dynamic key -->
<div :[key]="value"></div>

<!-- v-on with dynamic event -->
<div v-on:[event]="handler"></div>

<!-- v-on shorthand with dynamic event -->
<div @[event]="handler"></div>

<!-- v-slot with dynamic name -->
<foo>
  <template v-slot:[name]>
    Hello
  </template>
</foo>

<!-- v-slot shorthand with dynamic name -->
<!-- pending #3 -->
<foo>
  <template #[name]>
    Default slot
  </template>
</foo>
```

### 将`null`作为特殊值处理

动态参数的值应该是一个字符串。但是，如果我们允许`null`作为一个特殊值来明确表示应该解除绑定，使用时会很方便。任何其他非字符串都可能错误且触发警告。

`null`作为一个特殊值仅适用于`v-bind`和`v-on`，而不适合`v-slot`。这是因为`v-slot`不是绑定且不能被移除。自定义指令可以自己是否需要处理非字符串参数，但在使用时应遵守约定。

# 缺点 / 注意事项

### 表达式约束

理论上来讲，对任意复杂程度的JavasScript表达式使用指令参数复杂，但是html属性名不能包含空格和引号，所以在某些清空，用户可能会犯如下的错：

``` html
<div :[key + 'foo']="value"></div>
```

这并不会像预期那样运行。一种解决方法是：

``` html
<div :[`${key}foo`]="value"></div>
```

也就是说，复杂的动态键绑定应该通过计算属性在JavaScript中提前转换。

**更新:**: 可以检测到这种用法并在解析器中提供适当的警告（通过检查缺少右括号的参数）。

### 自定义指令

允许所有指令的动态参数意味着自定义指令除了考虑潜在的值变更之外还需要考虑潜在的参数变更。

这还添加`binding.oldArgs`到自定义指令的上下文中去。

# 备选方案

N/A

# 采用策略

这是不间断的，应该才去适当的文档更新直接引入。

# 未解决的问题

N/A