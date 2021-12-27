# 摘要

通过使用具名导出尽可能多的API，来使得Vue运行时可树摇。

# 例子

```js
import { nextTick, observable } from 'vue'

nextTick(() => {})

const obj = observable({})
```

# 改动

随着Vue的API增长，我们不断地尝试平衡功能和包大小之间权衡。我们想保持Vue的文件大小最小，但我们也不希望因为大小而限制其功能。

借助ES模块静态分析的友好设计，现在捆绑器和压缩器相结合，能够排除ES模块导出中未被其他任何使用使用到的API。我们可以采用这一点重构Vue的全局和局部的API，这样用户就仅会使用他们实际使用到的功能。

此外，对于使用的用户来说，可选功能并不会增加包的大小，在核心中，我们就会有更多的空间包含可选功能。

# 详细设计

在目前的2.x版本中，所有全局API都暴露给了单Vue实例对象:

```js
import Vue from 'vue'

Vue.nextTick(() => {})

const obj = Vue.observable({})
```

在3.x版本，他们**仅能**通过具名导入方式获得

```js
import Vue, { nextTick, observable } from 'vue'

Vue.nextTick // undefined

nextTick(() => {})

const obj = observable({})
```

通过不挂载所有的API到`Vue`默认导出，任何未使用的API都能被最终的支持树摇的打包器删除

## 受到影响的2.x版本的API

- `Vue.nextTick`
- `Vue.observable`
- `Vue.version`
- `Vue.compile` (仅完整版)
- `Vue.set` (仅兼容版)
- `Vue.delete` (仅兼容版)

## 局部帮手

除了公共API，许多局部组件或帮手也可以作为具名导出方式导出。这允许编译器输出仅使用时导入的功能。例如如下的模板：

```html
<transition>
  <div v-show="ok">hello</div>
</transition>
```

会编译成如下代码（为了解释，而非精确输出）：

``` js
import { h, Transition, applyDirectives, vShow } from 'vue'

export function render() {
  return h(Transition, [
    applyDirectives(h('div', 'hello'), this, [vShow, this.ok])
  ])
}
```

这意味着`Transition`组件仅当程序实际使用它时，才会被导入。

**注意，以上仅适用于具有树摇功能的ES模块捆绑器-UMD仍然包含所有的功能且暴露所有API到`Vue`全局变量（并且编译器将产生适当的输出来使用全局的API，而非导入）**。

# 缺点

用户不能再导入单个`Vue`变量来使用其API。然而，对于最小包的大小，这是一个值得权衡的问题。

## 插件中使用全局API

一些插件可能依赖最初暴露给`Vue`的全局API：

```js
const plugin = {
  install: Vue => {
    Vue.nextTick(() => {
      // ...
    })
  }
}
```

在3.0，他们需要明确地导入这些：

```js
import { nextTick } from 'vue'

const plugin = {
  install: app => {
    nextTick(() => {
      // ...
    })
  }
}
```

这会产生一些额外开销，因为这需要类库的作者在他们的构建过程中适当地配置Vue：

- Vue不应被打包到类库;
- 对于模块构建，导入的模块应该单独存在并由用户捆绑器处理；
- 对于UMD/浏览器构建，它应该首先尝试全局的`Vue.h`并退回到`requiure`调用。

这是React类库常见的做法，并且在webpack和Rollup中都是可能的。很多Vue类库也早就这样做了，我们仅需要提供适当的文档和工具支持。

# 备选方案

N/A

# 采用策略

作为迁移工具的一部分，它应该能够为此提供代码模组。