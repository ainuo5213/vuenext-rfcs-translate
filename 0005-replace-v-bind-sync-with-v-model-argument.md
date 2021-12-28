## 摘要

移除`v-bind`的`.sync`修饰符，用带参数的`v-model`作为替代

## 例子

替代:

```vue
<MyComponent v-bind:title.sync="title" />
```

语法应该是:

```vue
<MyComponent v-model:title="title" />
```

## 改动

我已经看到`v-bind.sync`在Vue2中引起了很多的混乱，因为用户希望使用像v-bind这样的表达式（而不管我们向文档中添加任何东西）。最好的解释如下：

> 将`v-bind:title.sync="title"`视为一种具有额外行为的普通绑定确实思考错了方向，因为两种绑定根本不同。`.sync`修饰符同v-model运行原理类似，是Vue创建双向绑定的另一种语法糖。主要区别在于它扩展了一种稍微不同的模式，允许你在一个组件中有多个双向绑定，而非仅限一个。

这让我想到了这个问题：如果告诉用户不要像`v-bind`那样看待`v-bind.sync`，而是要像理解`v-model`那样理解`v-bind.sync`，那么它是否应该成为`v-model`API的一部分呢？

## 详细设计

> **注意**：虽然这不再提案里，但是`v-model`的实现细节可能在Vue3里发生改变，来使得像隐式包装组件那样的公共模式更易于实现。当你看到`modelValue`属性和`update:modelValue`事件时，知道它是实现表单元素中`v-model`特殊行为的一个占位符，并不包含在这项提议中。

### 在元素节点上

```vue
<input v-model="xxx">

<!-- would be shorthand for: -->

<input
  :model-value="xxx"
  @update:model-value="newValue => { xxx = newValue }"
>
```

```vue
<input v-model:aaa="xxx">

<!-- INVALID: should throw a compile time error -->
```

请注意，`v-bind:aaa.sync="xxx"`目前不会抛出编译错误，尽管它应该抛出。

### 在组件上

```vue
<MyComponent v-model="xxx" />

<!-- would be shorthand for: -->

<MyComponent
  :model-value="xxx"
  @update:model-value="newValue => { xxx = newValue }"
/>
```

```vue
<MyComponent v-model:aaa="xxx"/>

<!-- would be shorthand for: -->

<MyComponent
  :aaa="xxx"
  @update:aaa="newValue => { xxx = newValue }"
/>
```

### `v-bind.sync="xxx"`重复的展开行为

`v-bind`和`v-on`是带有参数的指令。这两者都可使用他们无参版本来展开一个对象，但是`v-model`是一个没有参数的指令，其已是`v-demol:model-value="xxx"`的简写。我们能看到些许不同点：

1. **改变`v-model="xxx"`的行为来展开一个对象，强制用户编写`v-model:model-value="xxx"`是一种过时的行为**。这使得v-model`的行为和`v-bind`、`v-on`更相似，而且也创造了另一种重大变化并使得最常见的用例都变得更加冗余和复杂。
2. **向`v-model`添加新的修饰符（例如`.spread`）。**虽然这会最大幅度减少破坏性变更，但与其他带参数的指令的对象展开行为不一致，潜在地造成混乱，并使得整个框架整体感觉更加复杂。
3. **检测和改变原始对象值的行为（例如`v-model="{...xxx}"`）。**尽管这会减少破坏性变更，但是这与其他带参数的指令行为更一致，因为`v-bind={...xxx}`也有同样的效果。所以我觉得这会引起分歧，尽管有些人可能认为它非常直观，但是其他人会觉得使用`xxx`创建了一个相对于`{...xxx}`完全不同行为的东西会很困惑。
4. **简单来看，不允许`v-model`使用对象展开**。这能避免上述两个提议的问题，但是会有一个缺点，它会使得一些人更难以迁移到Vue3（虽然这可能是少部分人）。模板或JSX因为这一个功能，最好的情况就是变得更加难以书写和维护，最坏的情况就是无法使用（使用`createElement/h`函数强制重构渲染函数）

这些都不是很好的选择，但是我可能最中意第二种。我也想听听其他我错过了的解决方案。

## 缺点

除了重大变更引起的无法避免的痛点，我认为这种语法的痛点局部在于`.sync`修饰符没有大规模使用，并且对于正在迁移至Vue3的用户是一种很简单的方式（参阅下面的采用策略）

## 采用策略

作为一项重大变更，这只能在主要版本（v3）中引入。但是，我认为我们做一些事情让迁移更加简单：

- 检测到在`v-bind`上使用`.sync`修饰符时抛出一个异常，链接到迁移指南中此项改变的迁移入口。
- 使用新的迁移工具，我们要能够100%检测和自动修复`v-bind`和`.sync`一起用的情况

结合起来，即使使用了大量`.sync`的大型代码库，迁移起来也仅需要几分钟。