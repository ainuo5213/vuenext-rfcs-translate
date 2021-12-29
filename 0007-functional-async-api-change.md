# 摘要

- 函数式组件必须写成普通函数
  - `{ functional: true }` 选项被移除
  - `<template functional>` 将不再支持
- 异步组件现在必须通过专门的API进行创建

# 例子

``` js
import { h } from 'vue'

const FunctionalComp = props => {
  return h('div', `Hello! ${props.name}`)
}
```

``` js
import { defineAsyncComponent } from 'vue'

const AsyncComp = defineAsyncComponent(() => import('./Foo.vue'))
```

# 改动

## 简化函数式组件

在2.x版本，函数式组件必须以下列形式进行创建：

``` js
const FunctionalComp = {
  functional: true,
  render(h) {
    return h('div', `Hello! ${props.name}`)
  }
}
```

这会有以下的问题：

- 即使组件仅需要渲染函数，它仍需要使用`functional: true`这个对象。
- 某些选项支持（例如`props`和`inject`）某些选项不支持（例如`components`）。但用户常觉得所有选项都应该支持，因为它看起来就跟正常的状态组件一样（特别是使用`<template functional>`的单文件组件）。

另一方面我们注意到，有些用户使用仅出于性能原因才使用函数式组件，例如在带有`<template functional>`的单文件组件中，就要求我们支持更多带有状态的组件选项。但是，我认为我们不应该在这个事情投入更多时间。

在V3版本，状态组件和无状态组件之间的性能差异已经被显著地降低了，并且在大多数例子可以忽略不计。因此，可以不必仅为了性能而是使用函数式组件，也证明不再维护`<template functional>`是个明智的选择。v3版本的函数式组件应该主要是为了简单，而不仅是性能。

# 详细设计

在3.x版本，我们倾向使函数式组件作为一个普通：

``` js
import { h } from 'vue'

const FunctionalComp = (props, { slots, attrs, emit }) => {
  return h('div', `Hello! ${props.name}`)
}
```

- `functional`选项会被移除，对象中带有`{ functional: true }`将不再支持。
- 单文件组件将不再支持`<template functional>`。如果你需要的不仅仅是个函数，那就像组件一样使用它。
- 函数功能也发生了改变：
  - `h`将会全局导入。
  - 函数接收两个参数：`props`和上下文对象，该对象暴露`slots`、`attrs`和`emit`。他们等价于有状态组件中带有前缀`$`的等价项。

## 与旧语法作比较

新的函数参数应该可以完全替代2.x版本中函数渲染上下文：

- `props` 和`slots` 有等价项

- `data` 和`children` 不再必须只使用 `props` and `slots`的话);

- `listeners` 将会包含在 `attrs`;

- `injections` 将会被新的API `inject` 替代 ([Composition API](https://vue-composition-api-rfc.netlify.com/api.html#provide-inject)的一部分):

  ``` js
  import { inject } from 'vue'
  import { themeSymbol } from './ThemeProvider'
  
  const FunctionalComp = props => {
    const theme = inject(themeSymbol)
    return h('div', `Using theme ${theme}`)
  }
  ```

- `parent`将会被移除。在某些内部用例中是一种选择，但是用户代码中，props和injections才是首选。

## 可选的参数声明

为了更加简单易用，3.x的函数式组件不再需要声明`props`：

```js
const Foo = props => h('div', props.msg)
```

``` html
<Foo msg="hello!" />
```

没有显示地声明props，第一个参数`props`就会包含父组件传递给组件的所有数据。

## 明确的参数声明

要添加显示的props声明，请将`props`挂载到函数本身：

``` js
const FunctionalComp = props => {
  return h('div', `Hello! ${props.name}`)
}

FunctionalComp.props = {
  name: String
}
```

## Async Component Creation

新的异步组件API在 [its own dedicated RFC](https://github.com/vuejs/rfcs/pull/148)讨论。

# 成本

- 迁移成本

# 备选方案

N/A

# 采取策略

- 对于函数式组件，可以为每一次的迁移提供兼容模式。
- 使用`<template functional>`的单文件会被转换成普通的单文件组件。