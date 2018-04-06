> 声明：本文中提到的**项目**是基于Vue-cli生成的项目demo，vue版本为 ^2.5.2。
>
> 本文中提到的**Vue源码**是[vue-2.5.16](https://github.com/vuejs/vue/archive/v2.5.16.zip)
>
> 本文中提到的**Vuex源码**是[vuex-3.0.1](https://github.com/vuejs/vuex/archive/v3.0.1.zip)

本文参考了 掘金作者：**滴滴出行·DDFE**  的  [Vuex 2.0 源码分析](https://juejin.im/post/58352aaf880741006cfd65af)这篇文章，感谢作者和掘金。

#### 什么是vuex？

```
Vuex 是一个专为 Vue.js 应用程序开发的状态管理模式。它采用集中式存储管理应用的所有组件的状态，并以相应的规则保证状态以一种可预测的方式发生变化。Vuex 也集成到 Vue 的官方调试工具 devtools extension，提供了诸如零配置的 time-travel 调试、状态快照导入导出等高级调试功能。
```

这段来自官方的解释告诉我们，Vuex是来做状态管理。但是，为什么要用Vuex来做状态管理？

仔细回想一下，当我们使用Vue在开发SPA时，会遇到一些问题：

- 多个Vue组件之间怎样共享状态？
- Vue组件之间，包括兄弟组件之间，父子组件之间以及两个没有直接关系的组件之间怎样通讯？

一般来说，如果我们项目不是很复杂的话，我们会使用全局时间总线（global event bus）解决。

```
// 将在各处使用该事件中心
// 组件通过它来通信
var eventHub = new Vue()
```

在组件中，我们引入eventHub，然后可以使用 `$emit`, `$on`, `$off` 分别来分发、监听、取消监听事件：

```
 eventHub.$emit('delete-todo', id)
 eventHub.$on('delete-todo', this.deleteTodo)
 eventHub.$off('delete-todo', this.deleteTodo)
```

这样，我们可以实现简单的组件之间通信。关于Vue怎样通过`$emit` `$on` `$off` 实现组件通信，请看[【Vuex源码分析】Vue中EventBus实现原理](./【Vuex源码分析】Vue中EventBus实现原理.md)。

但是，随着我们项目的复杂度越来越高，如果还是使用EventBus，将很难对其进行维护。因此，我们需要引入vuex。具体怎样使用vuex，请移步官方文档。这里我们主要是分析源码。

![](https://vuex.vuejs.org/zh-cn/images/vuex.png)

Vuex 背后的基本思想，借鉴了 [Flux](https://facebook.github.io/flux/docs/overview.html)、[Redux](http://redux.js.org/)、和 [The Elm Architecture](https://guide.elm-lang.org/architecture/)。与其他模式不同的是，Vuex 是专门为 Vue.js 设计的状态管理库，以利用 Vue.js 的细粒度数据响应机制来进行高效的状态更新。

将Vuex源码[vuex-3.0.1](https://github.com/vuejs/vuex/archive/v3.0.1.zip) clone下来之后，我们可以看到其目录结构：

![](/Users/lichao/学习笔记/Vue技术栈源码分析系列/assets/vuex_constructor.png)

其源码主要放到了src文件夹下。

![](/Users/lichao/学习笔记/Vue技术栈源码分析系列/assets/vuex_src.png)

其中的只包含了几个js文件，以及模块文件和插件。

之后，我们来开始看一下源码。