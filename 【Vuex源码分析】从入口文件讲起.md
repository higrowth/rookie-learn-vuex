之前我们大致浏览了vuex的源码目录结构。现在，我们从入口文件`/src/index.js`看起：

```js
import { Store, install } from './store'
import { mapState, mapMutations, mapGetters, mapActions, createNamespacedHelpers } from './helpers'

export default {
  Store,
  install,
  version: '__VERSION__',
  mapState,
  mapMutations,
  mapGetters,
  mapActions,
  createNamespacedHelpers
}
```

从export可以看出vuex对外暴露出的API。这里稍微说一下，Store是Vuex提供的状态存储类，我们使用vuex就是通过`new Store()`来获得实例对象，稍后再详细讲解。



在Vue中我们知道，Vue有个全局性的API，就是`Vue.use(plugin)` 这个方法。我们看一下官方文档的解释：

```
Vue.use( plugin )

参数：
{Object | Function} plugin

用法：
安装 Vue.js 插件。如果插件是一个对象，必须提供 install 方法。如果插件是一个函数，它会被作为 install 方法。install 方法调用时，会将 Vue 作为参数传入。
当 install 方法被同一个插件多次调用，插件将只会被安装一次。
```

这里明确说明，当在Vue中使用插件时，必须提供intsall方法。所以当我们在vue项目中使用vuex时，我们一般会这样操作：

```
import Vue from 'vue'
import Vuex from 'vuex'
...
Vue.use(Vuex)
```

由于vuex也可以看做是一个插件，当调用`Vue.use(Vuex)` 时，实际上就是调用了 install 的方法并传入 Vue 的引用。

这里先看一下install方法。

```js
let Vue // bind on install

//...

export function install (_Vue) {
  if (Vue && _Vue === Vue) {
    if (process.env.NODE_ENV !== 'production') {
      console.error(
        '[vuex] already installed. Vue.use(Vuex) should be called only once.'
      )
    }
    return
  }
  Vue = _Vue
  applyMixin(Vue)
}
```

对 Vue 的判断主要是保证 install 方法只执行一次，这里把 install 方法的参数 _Vue 对象赋值给 Vue 变量，这样我们就可以在 index.js 文件的其它地方使用 Vue 这个变量了。install 方法的最后调用了 applyMixin 方法，我们顺便来看一下这个方法的实现，在 src/mixin.js 文件里定义：

```js
export default function (Vue) {
  const version = Number(Vue.version.split('.')[0])

  if (version >= 2) {
    Vue.mixin({ beforeCreate: vuexInit })
  } else {
    // override init and inject vuex init procedure
    // for 1.x backwards compatibility.
    const _init = Vue.prototype._init
    Vue.prototype._init = function (options = {}) {
      options.init = options.init
        ? [vuexInit].concat(options.init)
        : vuexInit
      _init.call(this, options)
    }
  }

  /**
   * Vuex init hook, injected into each instances init hooks list.
   */

  function vuexInit () {
    const options = this.$options
    // store injection
    if (options.store) {
      this.$store = typeof options.store === 'function'
        ? options.store()
        : options.store
    } else if (options.parent && options.parent.$store) {
      this.$store = options.parent.$store
    }
  }
}
```

这段代码的作用就是在 Vue 的生命周期中的初始化（1.0 版本是 init，2.0 版本是 beforeCreated）钩子前插入一段 Vuex 初始化代码。这里做的事情很简单——给 Vue 的实例注入一个 `$store` 的属性，这也就是为什么我们在 Vue 的组件中可以通过 `this.$store.xxx` 访问到 Vuex 的各种数据和状态。