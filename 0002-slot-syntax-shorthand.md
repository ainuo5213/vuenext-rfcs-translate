# 摘要

添加`v-slot`的简写语法在[rfc-0001](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0001-new-slot-syntax.md)已提议。请务必先阅读它来获得该提案适当的背景

# 例子

``` html
<foo>
  <template #header="{ msg }">
    Message from header: {{ msg }}
  </template>

   <template #footer>
    A static footer
  </template>
</foo>
```

# 改动

人如其名，简写的目的旨在提供更简洁的语法。

在Vue中，我们目前仅对两个指令提供了简写：`v-bind`和`v-on`：

``` html
<div v-bind:id="id"></div>
<div :id="id"></div>

<button v-on:click="onClick"></button>
<button @click="onClick"></button>
```

`v-bind` 和 `v-on`经常在一个元素中使用，当信息每个元素之间的不同信息是**指令参数**而非指令本身时，会变得冗余。**因此，通过简写省去重复部分(v-xxx)和高亮不同的部分（参数），能够提高阅读和书写效率**（原文是**signal/noise ratio**，意味信噪比）

新的`v-slot`语法在多个插槽中也会遇到相同的问题：

``` html
<TestComponent>
  <template v-slot:one="{ name }">Hello {{ name }}</template>
  <template v-slot:two="{ name }">Hello {{ name }}</template>
  <template v-slot:three="{ name }">Hello {{name }}</template>
</TestComponent>
```

这里的`v-slot`被重复了多次，实际上他们不同点仅只有插槽名字（插槽参数）。

简写使得插槽名字更容易阅读和记住

``` html
<TestComponent>
  <template #one="{ name }">Hello {{ name }}</template>
  <template #two="{ name }">Hello {{ name }}</template>
  <template #three="{ name }">Hello {{name }}</template>
</TestComponent>
```

# 详细设计

`v-slot`的简写遵循`v-bind`和`v-on`的简写规则：使用简写符号`#`替换指令名称和冒号

``` html
<!-- full syntax -->
<foo>
  <template v-slot:header="{ msg }">
    Message from header: {{ msg }}
  </template>

   <template v-slot:footer>
    A static footer
  </template>
</foo>

<!-- shorthand -->
<foo>
  <template #header="{ msg }">
    Message from header: {{ msg }}
  </template>

   <template #footer>
    A static footer
  </template>
</foo>
```

**和`v-bind`及`v-on`相似，简写仅只有有参数时可用。这意味着`v-slot`在没有参数时不能够简写成`#=`**。对于默认插槽，要么用全写语法`v-slot`，要么使用显示名字（`#default`）。

``` html
<foo v-slot="{ msg }">
  {{ msg }}
</foo>

<foo #default="{ msg }">
  {{ msg }}
</foo>
```

`#`的选择是基于之前的RFC反馈收集选择的。它与CSS中id选择器的相似，并且在概念上很好可以很好地表示插槽名字。

在实际依赖于作用域插槽的类库中（[vue-promised](https://github.com/posva/vue-promised)），也有些使用使用案例：

 ``` html
<Promised :promise="usersPromise">
  <template #pending>
    <p>Loading...</p>
  </template>

   <template #default="users">
    <ul>
      <li v-for="user in users">{{ user.name }}</li>
    </ul>
  </template>

   <template #rejected="error">
    <p>Error: {{ error.message }}</p>
  </template>
</Promised>
 ```

# 缺点

- 也有些人说插槽并不那么常用，因此不需要简写，而且这会给初学者带来额外的学习成本。对此作出回应：
  1. 我认为作用域插槽是构建高度可定制的和可组合的第三方组件所必需的重要机制。在未来，我想我们会看到更多的组件库依赖于定制化和组合化。对于使用组件库的用户来说，简写就会变得非常有价值（如例所示）。
  2. 简写的翻译规则言简意赅，并且和现有简写一致。如果用户了解基础语法怎么运作的，那么理解`v-slot`的简写就是额外的一小步。

# 备选方案

在之前讨论的RFC中也出现了一些备替代符号。唯一有相似的就是`&`：

``` html
<foo>
  <template &header="{ msg }">
    Message from header: {{ msg }}
  </template>

   <template &footer>
    A static footer
  </template>
</foo>
```

# 采用策略

这应该是新语法`v-slot`之上的一种扩展。理想情况下，我们想要同事引入基础语法和简写，以便让用户学习。过后再引入简写会冒有一些风险，有部分用户仅知道`v-slot`，而对其他人代码中的`#`感到困惑。

# Unresolved questions

N/A