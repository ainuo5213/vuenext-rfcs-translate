# 摘要

- `h`现在使用全局导入，而不是在位一个参数传递给渲染函数。
- 渲染函数参数改变，并且有状态组件和函数式组件保持一致。
- VNodes现在有一个扁平的props结构。

# 例子

``` js
// globally imported `h`
import { h } from 'vue'

export default {
  render() {
    return h(
      'div',
      // flat data structure
      {
        id: 'app',
        onClick() {
          console.log('hello')
        }
      },
      [
        h('span', 'child')
      ]
    )
  }
}
```

# 改动

在2.x版本，VNodes是上下文特定的，这意味着创建的每一个VNode都绑定了创建它的组件实例（“context”）。这是因为我们需要支持如下的例子（`createElement`俗称`h`）：

``` js
// 通过字符串ID查找组件
h('some-component')

h('div', {
  directives: [
    {
      name: 'foo', // 通过字符串ID查找指令
      // ...
    }
  ]
})
```

为了查找已注册的组件和指令，我们需要知道“拥有”这个VNode的组件实例上下文。这就是为什么在2.x版本中`h`是通过参数传递，因为传递给渲染函数的`h`是预先绑定到上下文实例的。（就像`this.$createElement`）。

这会造成诸多不便，例如，当视图分离渲染逻辑到单独的函数时，`h`需要这样传递：

``` js
function renderSomething(h) {
  return h('div')
}

export default {
  render(h) {
    return renderSomething(h)
  }
}
```

当时用JSX时，这特别麻烦，因为`h`是隐式使用的，不需要用户编码。为了缓解这种清空，我们的JSX插件就必须自动执行`h`函数，逻辑复杂且脆弱。

在3.0版本，我们找到了无关上下文创建VNodes的方法。现在他可以在任何地方通过全局导入`h`函数，所以再任何文件它仅需要导入一次。

---

2.x版本的渲染函数API另一个问题是嵌套的Vnode数据结构：

``` js
h('div', {
  class: ['foo', 'bar'],
  style: { }
  attrs: { id: 'foo' },
  domProps: { innerHTML: '' },
  on: { click: foo }
})
```

这个结构继承自Snabbdom——2.x版本的Vue实现的最初的虚拟DOM。这样设计的原因是，不同的逻辑可以模块化：一个独立的模块（例如`class`模块）其实例是不同的。每个绑定的处理也更加明确：

但是，随着时间推移，我们注意到与扁平化结构相比，嵌套结构有许多缺点：

- 写起来更加冗长
- `class`和`style`在特殊情况下表现得不一致
- 更多的内存消耗（更多的对象内存需要分配）
- diff算法执行得更慢（每一个嵌套建构需要自身反复递归）
- 更加复杂，克隆成本更高，更加难以合并和展开
- 运行在JSX时，需要更多特殊的规则或作出明确的声明。

在3.x版本，我们正在向扁平化的VNode数据结构转变，以解决这些问题。

# 详细设计

## 全局导入`h`函数

``` js
import { h } from 'vue'

export default {
  render() {
    return h('div')
  }
}
```

## 改变渲染函数签名

由于不需要`h`作为参数，`render`函数现在不需要再接收其他参数。实际上，在3.0版本中，`render`选项将主要作为模板编译器生成渲染函数的出口。对于手动配置的渲染函数，建议它从`setup`函数进行返回：

``` js
import { h, reactive } from 'vue'

export default {
  setup(props, { slots, attrs, emit }) {
    const state = reactive({
      count: 0
    })

    function increment() {
      state.count++
    }

    // return the render function
    return () => {
      return h('div', {
        onClick: increment
      }, state.count)
    }
  }
}
```

从`setup()`返回的渲染函数天然可以访问响应式数据和作用域内声明的函数，以及传递给setup的参数：

- `props` 和`attrs` 等同于 `this.$props` and `this.$attrs` - 见 [Optional Props Declaration](https://github.com/vuejs/rfcs/pull/25) 和 [Attribute Fallthrough](https://github.com/vuejs/rfcs/pull/92).

- `slots` 等同于 `this.$slots` - 见 [Slots Unification](https://github.com/vuejs/rfcs/pull/20).

- `emit` 等同于 `this.$emit`.

`props`, `slots` 和`attrs` 对象都是代理对象，所以在渲染函数使用他们时，他们总是指向的最新的值。

`setup()`工作的细节，参考[Composition API RFC](https://vue-composition-api-rfc.netlify.com/api.html#setup).

## 函数式组件的签名

请注意，函数式组件的渲染函数也会有相同的签名，有状态组件和函数式组件保持一致：

``` js
const FunctionalComp = (props, { slots, attrs, emit }) => {
  // ...
}
```

新的参数列表会提供完全替代当前函数组件渲染上下文的能力：

- `props` 和`slots` 有等价项;

- `data` 和`children` 不再必需 (就和 `props` and `slots`一样);

- `listeners` 会被包含进 `attrs`;

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

## 扁平化的VNode参数格式

``` js
// before
{
  class: ['foo', 'bar'],
  style: { color: 'red' },
  attrs: { id: 'foo' },
  domProps: { innerHTML: '' },
  on: { click: foo },
  key: 'foo'
}

// after
{
  class: ['foo', 'bar'],
  style: { color: 'red' },
  id: 'foo',
  innerHTML: '',
  onClick: foo,
  key: 'foo'
}
```

在扁平化的结构下，VNode的参数可以被如下规则进行处理：

- `key` 和`ref` 都被保留下来
- `class` 和`style` 和2.x保持一致
- 以`on`开头的属性被当做`v-on`绑定一样处理，`on`之后的所有东西都会被转换为全小写的的事件名（更多如下：）
- 包括但不仅：
  - 如果该DOM节点有key，则将其设置为DOM的属性(property)
  - 其他情况则设置成一个特性(attribute)。

### 特殊“保留”的参数

有两个全局保留的参数：

- `key`
- `ref`

初次之外你可以使用被保留有前缀`onVnodexxx`的钩子来勾入组件的声明周期：

``` js
h('div', {
  onVnodeMounted(vnode) {
    /* ... */
  },
  onVnodeUpdated(vnode, prevVnode) {
    /* ... */
  }
})
```

这些钩子也是定制自定义组件的基础。因为它们以`on`开头，所以他们也可以在模板中用`v-on`来声明：

``` html
<div @vnodeMounted="() => { ... }">
```

---

由于扁平的结构，组件内的`this.$attrs`包含了组件未显示声明的原始参数，包括`class`、`style`、`onXXX`监听和`vnodeXXX`钩子。这使组件封装的编写更加容易。只需要传递通过`v-bind="$attrs"`来向下传递`this.$attr`。

## 上下文无关的VNodes

由于vnode是上下文无关的，我们不会再使用字符串ID（例如`h('some-component')`）来隐式查找全局注册过的主键。查找指令也一样。相反我们需要使用导入的API：

``` js
import { h, resolveComponent, resolveDirective, withDirectives } from 'vue'

export default {
  render() {
    const comp = resolveComponent('some-global-comp')
    const fooDir = resolveDirective('foo')
    const barDir = resolveDirective('bar')

    // <some-global-comp v-foo="x" v-bar="y" />
    return withDirectives(
      h(comp),
      [fooDir, this.x],
      [barDir, this.y]
    )
  }
}
```

这将主要用于编译器生成的输出，因为手工编写的渲染函数通常直接导入组件和指令，并且按质使用它们。

# 缺点

## 依赖的Vue核心

`h`全局导入意味着任何包含Vue的类库都将会包含`import { h } from 'vue'`（这也是隐式包含到模板编译后的渲染函数中的）。这会造成一定的开销，因为它要求类库作者在构建setup是正确配置Vue：

- Vue不应该被打包进类库（这里指整体导入的Vue）；
- 对于模块构建，导入应该单独进行，并交由最终打包用户；
- 对于UMD或浏览器构建，它应该尝试全局的`Vue.h`，然后再退回去用`require`调用。

这是React类库的常见操作，在webpack和Rollup也时常出现。大多数Vue类库也早已做到这一点。我们只需要提供正确的文档和工具支持。

# 备选方案

N/A

# 采取策略

- 对于使用模板的用户，不会有影响。
- 对于使用JSX的用户，影响也很小，我们确实需要重写我们的JSX插件。
- 对于使用`h`手动编写渲染函数的用户有主要的迁移成本。但这在我们的用户中应该只占有很少的比例，但我们确实需要提供一个像样的迁移途径。

  - 我们可能提供一个兼容插件来对渲染函数进行处理，并使其暴露一个兼容2.x版本的参数，并且可以在每一个组件迁移过程中关闭。
  - 也有可能提供一个codemod，它将自动转换`h`调用结果为使用新的VNode的数据格式，因为h函数的映射很机械。
- 使用上下文功能的函数组件可能必须手动迁移，但可以提供一个相似的适配器。

# 没有解决的问题

## 显示绑定的Escape Hatches

对于扁平化的VNode数据结构，怎么在内部处理属性变得有点不太好搞。这会产生一些问题——例如，怎么显示地设置不存在的DOM的属性，或者在自定义元素上的capscase事件。

我们可能希望通过前缀支持显示绑定的类型：

``` js
h('div', {
  'attr:id': 'foo',
  'prop:__someCustomProperty__': { /*... */ },
  'on:SomeEvent': e => { /* ... */ }
})
```

它相当于2.x版本中`attrs`，`domProps`和`on`的嵌套。但是，这要求我们对每个修补的属性都要执行额外的检查，这会导致对一个非常小的用例需要付出持续的性能成本。我们应该会找到一个更好的方法解决它。