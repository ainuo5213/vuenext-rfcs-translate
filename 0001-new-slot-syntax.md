# 摘要

介绍使用作用域插槽的一种新语法：

- 在单指令语法中，新的指令`v-slot`会将`slot`和`slot-scope`统一起来
- `v-slot`的简写形式也能够隐式地统一作用域插槽和普通插槽的写法

# 例子

使用`v-slot`来声明传递到作用域插槽`<foo>`的参数：

``` html
<!-- 默认插槽 -->
<foo v-slot="{ msg }">
  {{ msg }}
</foo>

<!-- 具名插槽 -->
<foo>
  <template v-slot:one="{ msg }">
    {{ msg }}
  </template>
</foo>
```

# 改动

当我们首次引入作用域插槽时，因为它总是使用`<template slot-scope>`，就显得很冗余

When we first introduced scoped slots, it was verbose because it required always using `<template slot-scope>`:

``` html
<foo>
  <template slot-scope="{ msg }">
    <div>{{ msg }}</div>
  </template>
</foo>
```

为了降低这种冗余，在2.5版本中，我们对插槽元素中加入了使用`slot-scope`的能力

``` html
<foo>
  <div slot-scope="{ msg }">
    {{ msg }}
  </div>
</foo>
```

这也意味着这种写法在组件中也能够正常工作

``` html
<foo>
  <bar slot-scope="{ msg }">
    {{ msg }}
  </bar>
</foo>
```

但是，上面的使用方法会导致一个问题：`slot-scope`的放置并不能总是清楚地反映哪一个组件实际提供这该作用域变量。例如，`slot-scope`被放置在了组件`<bar>`上，但是他实际定义的变量是由默认插槽`<foo>`提供的

随着嵌套的加上，这种情况会变得更糟

``` html
<foo>
  <bar slot-scope="foo">
    <baz slot-scope="bar">
      <div slot-scope="baz">
        {{ foo }} {{ bar }} {{ baz }}
      </div>
    </baz>
  </bar>
</foo>
```

在模板中，由于不能立即得知哪个组件提供了哪些变量

有一些人建议，我们应该在组件本身上使用`slot-scope`来表示其默认插槽的作用域

``` html
<foo slot-scope="foo">
  {{ foo }}
</foo>
```

不幸地是，当组件嵌套时，如下代码不能工作，因为这会导致歧义

``` html
<parent>
  <foo slot-scope="foo"> <!-- provided by parent or by foo? -->
    {{ foo }}
  </foo>
</parent>
```

这就是为什么我现在认为应该使用不带有模板的`slot-scope`是一个错误

### 为什么用新的指令而不是修复`slot-scope`？

如果我们能回到过去，我可能会改变`slot-scope`的语义，但是：

1. 这将会是一个突破性的改变，这意味着我们再也不能够在2.x版本中发布它。
2. 即使我们在3.x版本中改变了`slot-scope`，但是改变现有已存在的语法会给未来学习者带来很多困惑，那么学习资料就会变得过时。我们最终决定避免这样的改动。所以，我们引入了新的内容用于区别`slot-scope。
3. 在3.x版本中，我们计划统一作用域类型，那么我们就不必再区分作用域插槽和非作用域插槽（概念上）。插槽可能会，也有可能不会接收参数，但是他们都是插槽。在概念上的统一之后，我们就不再需要`slot`和`slot-scope`这两个特别的属性，在一种新的结构下统一语法也非常得好。