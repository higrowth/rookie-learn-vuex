**Vuex 的初始化核心**

```js
	// init root module.
    // this also recursively registers all sub-modules
    // and collects all module getters inside this._wrappedGetters
    installModule(this, state, [], this._modules.root)

    // initialize the store vm, which is responsible for the reactivity
    // (also registers _wrappedGetters as computed properties)
    resetStoreVM(this, state)

    // apply plugins
    plugins.forEach(plugin => plugin(this))

    if (Vue.config.devtools) {
      devtoolPlugin(this)
    }
```

这段代码是 Vuex 的初始化的核心，其中，installModule 方法是把我们通过 options 传入的各种属性模块注册和安装；resetStoreVM 方法是初始化 store._vm，观测 state 和 getters 的变化；最后是应用传入的插件。如果开启了开发工具，也会把实例对象传给开发工具。

再看一下installModule做了什么事情：

```
function installModule (store, rootState, path, module, hot) {
  const isRoot = !path.length
  const namespace = store._modules.getNamespace(path)

  // register in namespace map
  if (module.namespaced) {
    store._modulesNamespaceMap[namespace] = module
  }

  // set state
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1))
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
      Vue.set(parentState, moduleName, module.state)
    })
  }

  const local = module.context = makeLocalContext(store, namespace, path)

  module.forEachMutation((mutation, key) => {
    const namespacedType = namespace + key
    registerMutation(store, namespacedType, mutation, local)
  })

  module.forEachAction((action, key) => {
    const type = action.root ? key : namespace + key
    const handler = action.handler || action
    registerAction(store, type, handler, local)
  })

  module.forEachGetter((getter, key) => {
    const namespacedType = namespace + key
    registerGetter(store, namespacedType, getter, local)
  })

  module.forEachChild((child, key) => {
    installModule(store, rootState, path.concat(key), child, hot)
  })
}
```

installModule 函数可接收5个参数，store、rootState、path、module、hot，store 表示当前 Store 实例，rootState 表示根 state，path 表示当前嵌套模块的路径数组，module 表示当前安装的模块，hot 当动态改变 modules 或者热更新的时候为 true。

module一般长这样：

```
modules: {
    foo: {
      namespaced: true,

      actions: {
        someAction: {
          root: true,
          handler (namespacedContext, payload) { ... } // -> 'someAction'
        }
      }
    }
  }
```

代码首先通过 path 数组的长度判断是否为根。我们在构造函数调用的时候是 `installModule(this, state, [], options)`，所以这里 isRoot 为 true。

如果希望你的模块具有更高的封装度和复用性，你可以在子模块中通过添加 `namespaced: true` 的方式使其成为带命名空间的模块。这时，会将这部分模块放到对应的命名空间中去。

接着看：

```
 // set state
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1))
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
      Vue.set(parentState, moduleName, module.state)
    })
  }
```

这里判断当不为根且非热更新的情况，然后设置级联状态，这里乍一看不好理解，我们先放一放，稍后来回顾。

```js
 const local = module.context = makeLocalContext(store, namespace, path)

  module.forEachMutation((mutation, key) => {
    const namespacedType = namespace + key
    registerMutation(store, namespacedType, mutation, local)
  })

  module.forEachAction((action, key) => {
    const type = action.root ? key : namespace + key
    const handler = action.handler || action
    registerAction(store, type, handler, local)
  })

  module.forEachGetter((getter, key) => {
    const namespacedType = namespace + key
    registerGetter(store, namespacedType, getter, local)
  })

  module.forEachChild((child, key) => {
    installModule(store, rootState, path.concat(key), child, hot)
  })
```

这段代码中，会调用makeLocalContext()函数，会根据是否开启命名空间来获取上下文，具体实现稍后再讲，

接下来会分别处理module中的mutation、action、getter，分别调用它们的注册方法进行注册。

那么到这，如果 Vuex 没有 module ，这个 installModule 方法可以说已经做完了。但是 Vuex 巧妙了设计了 module 这个概念，因为 Vuex 本身是单一状态树，应用的所有状态都包含在一个大对象内，随着我们应用规模的不断增长，这个 Store 变得非常臃肿。为了解决这个问题，Vuex 允许我们把 store 分 module（模块）。每一个模块包含各自的 state、mutations、actions 和 getters，甚至是嵌套模块。所以，才有了接下来还有一行代码，

```
module.forEachChild((child, key) => {
    installModule(store, rootState, path.concat(key), child, hot)
  })
```

这里通过遍历 module中的子模块，递归调用 installModule 去安装子模块。这里传入了 store、rootState、path.concat(key)、和child，和刚才不同的是，path 不为空，module 对应为子模块，那么我们回到刚才那段代码：

```js
 // set state
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1))
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
      Vue.set(parentState, moduleName, module.state)
    })
  }
```

当递归初始化子模块的时候，isRoot 为 false，注意这里有个方法`getNestedState(rootState,  path.slice(0, -1))`，来看一下 getNestedState 函数的定义：

```js
function getNestedState (state, path) {
  return path.length
    ? path.reduce((state, key) => state[key], state)
    : state
}
```

这个方法很简单，就是根据 path 查找 state 上的嵌套 state。在这里就是传入 rootState 和 path，计算出当前模块的父模块的 state，由于模块的 path 是根据模块的名称 concat 连接的，所以 path 的最后一个元素就是当前模块的模块名，最后调用

```
store._withCommit(() => {
      Vue.set(parentState, moduleName, module.state)
    })
```

把当前模块的 state 添加到 parentState 中，其中key就是模块的名称，值就是子模块的state。

这里注意一下我们用了 store._withCommit 方法，来看一下这个方法的定义：

```
_withCommit (fn) {
    const committing = this._committing
    this._committing = true
    fn()
    this._committing = committing
  }
```

由于我们是在修改 state，Vuex 中所有对 state 的修改都会用 `_withCommit`函数包装，保证在**同步**修改 state 的过程中 `this._committing` 的值始终为true。这样当我们观测 state 的变化时，如果 this._committing 的值不为 true，则能检查到这个状态修改是有问题的。

所以，我们打印一下

```
new Vuex.Store({
    actions,
    getters,
    state,
    mutations,
    modules: {dres},
    strict: debug,
    plugins: []
  })
```

这个实例对象，如下：

![](/Users/lichao/GithubCode/rookie-learn-vuex/assets/vuex_store_instance.png)

这里我们也看到，在`_actions`下存放了我们定义的actions，其中有个`_committing`也说明了在action中commit调用mutation时，时同步操作。此外，在`_mutations`中保存了我们定义的mutations。而在state中，除了rootStore保存的state，还包括了一个子模块dres对应的state。

我们回过头来看一下 installModule 过程中的其它 3 个重要方法：registerMutation、registerAction 和 registerGetter。顾名思义，这 3 个方法分别处理 mutations、actions 和 getters。我们先来看一下 registerMutation 的定义：

```js
function registerMutation (store, type, handler, local) {
  const entry = store._mutations[type] || (store._mutations[type] = [])
  entry.push(function wrappedMutationHandler (payload) {
    handler.call(store, local.state, payload)
  })
}
```

registerMutation 是对 store 的 mutation 的初始化，它接受 4 个参数，store为当前 Store 实例，type为 模块 的 名称，handler 为 mutation 执行的回调函数，local 为 `makeLocalContext(store, namespace, path)`生成的当前模块的路径。mutation 的作用就是同步修改当前模块的 state ，函数首先通过 type 拿到对应的 mutation 对象数组， 然后把一个 mutation 的包装函数 push 到这个数组中，这个函数接收一个参数 payload，这个就是我们在定义 mutation 的时候接收的额外参数。这个函数执行的时候会调用 mutation 的回调函数，并将实例对象store，当前模块的 state，和 playload 一起作为回调函数的参数。举个例子：

```
// ...
mutations: {
  increment (state, n) {
    state.count += n
  }
}
```

这里我们定义了一个 mutation，通过刚才的 registerMutation 方法，我们注册了这个 mutation，这里的 state 对应的就是当前模块的 state，n 就是额外参数 payload，接下来我们会从源码分析的角度来介绍这个 mutation 的回调是何时被调用的，参数是如何传递的。

我们有必要知道 mutation 的回调函数的调用时机，在 Vuex 中，mutation 的调用是通过 store 实例的 API 接口 commit 来调用的，来看一下 commit 函数的定义：

```js
commit (_type, _payload, _options) {
    // check object-style commit
    const {
      type,
      payload,
      options
    } = unifyObjectStyle(_type, _payload, _options)

    const mutation = { type, payload }
    const entry = this._mutations[type]
    if (!entry) {
      if (process.env.NODE_ENV !== 'production') {
        console.error(`[vuex] unknown mutation type: ${type}`)
      }
      return
    }
    this._withCommit(() => {
      entry.forEach(function commitIterator (handler) {
        handler(payload)
      })
    })
    this._subscribers.forEach(sub => sub(mutation, this.state))

    if (
      process.env.NODE_ENV !== 'production' &&
      options && options.silent
    ) {
      console.warn(
        `[vuex] mutation type: ${type}. Silent option has been removed. ` +
        'Use the filter functionality in the vue-devtools'
      )
    }
  }
```

commit 支持 3 个参数，`_type` 表示 mutation 的类型，`_payload` 表示额外的参数，`_options` 表示一些配置，比如 silent 等，稍后会用到。

commit 函数首先调用了 unifyObjectStyle()函数，

```js
function unifyObjectStyle (type, payload, options) {
  if (isObject(type) && type.type) {
    options = payload
    payload = type
    type = type.type
  }

  if (process.env.NODE_ENV !== 'production') {
    assert(typeof type === 'string', `Expects string as the type, but found ${typeof type}.`)
  }

  return { type, payload, options }
}
```

对 type 的类型做了判断，处理了 type 为 object 的情况，接着根据 type 去查找对应的 mutation，如果找不到，则输出一条错误信息，否则遍历这个 type 对应的 mutation 对象数组，执行 handler(payload) 方法，这个方法就是之前定义的 wrappedMutationHandler(handler)，执行它就相当于执行了 registerMutation 注册的回调函数，并把当前模块的 state 和 额外参数 payload 作为参数传入。注意这里我们依然使用了 `this._withCommit` 的方法提交 mutation。commit 函数的最后，判断如果不是静默模式，则遍历 `this._subscribers`，调用回调函数，并把 mutation 和当前的根 state 作为参数传入。那么这个 `this._subscribers` 是什么呢？

原来 Vuex 的 Store 实例提供了 subscribe API 接口，它的作用是订阅（注册监听） store 的 mutation。先来看一下它的实现：

```js
  subscribe (fn) {
    return genericSubscribe(fn, this._subscribers)
  }
  
  function genericSubscribe (fn, subs) {
      if (subs.indexOf(fn) < 0) {
        subs.push(fn)
      }
      return () => {
        const i = subs.indexOf(fn)
        if (i > -1) {
          subs.splice(i, 1)
        }
      }
  }
```

subscribe实际上调用的是genericSubscribe() 方法，genericSubscribe()方法也很很简单，他接受的参数是一个回调函数，会把这个回调函数保存到 `this._subscribers` 上，并返回一个函数，当我们调用这个返回的函数，就可以解除当前函数对 store 的 mutation 的监听。其实，Vuex 的内置 logger 插件就是基于 subscribe 接口实现对 store 的 muation的监听，稍后我们会详细介绍这个插件。

#### registerAction

在了解完 registerMutation，我们再来看一下 registerAction 的定义：

```js
function registerAction (store, type, handler, local) {
  const entry = store._actions[type] || (store._actions[type] = [])
  entry.push(function wrappedActionHandler (payload, cb) {
    let res = handler.call(store, {
      dispatch: local.dispatch,
      commit: local.commit,
      getters: local.getters,
      state: local.state,
      rootGetters: store.getters,
      rootState: store.state
    }, payload, cb)
    if (!isPromise(res)) {
      res = Promise.resolve(res)
    }
    if (store._devtoolHook) {
      return res.catch(err => {
        store._devtoolHook.emit('vuex:error', err)
        throw err
      })
    } else {
      return res
    }
  })
}
```

registerAction 是对 store 的 action 的初始化，它和 registerMutation 的参数一致，和 mutation 不同一点，mutation 是同步修改当前模块的 state，而 action 是可以**异步**去修改 state，这里不要误会，在 action 的回调中并不会直接修改 state ，仍然是通过提交一个 mutation 去修改 state（在 Vuex 中，mutation 是修改 state 的唯一途径）。那我们就来看看 action 是如何做到这一点的。

函数首先也是通过 type 拿到对应 action 的对象数组，然后把一个 action 的包装函数 push 到这个数组中，这个函数接收 2 个参数，payload 表示额外参数 ，cb 表示回调函数（实际上我们并没有使用它）。这个函数执行的时候会调用 action 的回调函数，传入一个 context 对象，这个对象包括了 store 的 commit 和 dispatch 方法、getter、当前模块的 state 和 rootState 等等。接着对这个函数的返回值做判断，如果不是一个 Promise 对象，则调用 `Promise.resolve（res）` 给res 包装成了一个 Promise 对象。这里也就解释了为何 Vuex 的源码依赖 Promise，这里对 Promise 的判断也和简单，参考代码 src/util.js，对 isPromise 的判断如下：

```
export function isPromise (val) {
  return val && typeof val.then === 'function'
}
```

其实就是简单的检查对象的 then 方法，如果包含说明就是一个 Promise 对象。

接着判断 `store._devtoolHook`，这个只有当用到 Vuex  devtools  开启的时候，我们才能捕获 promise 的过程中的 。 action 的包装函数最后返回 res ，它就是一个地地道道的 Promise 对象。来看个例子：

```
actions: {
  checkout ({ commit, state }, payload) {
    // 把当前购物车的商品备份起来
    const savedCartItems = [...state.cart.added]
    // 发送结帐请求，并愉快地清空购物车
    commit(types.CHECKOUT_REQUEST)
    // 购物 API 接收一个成功回调和一个失败回调
    shop.buyProducts(
      products,
      // 成功操作
      () => commit(types.CHECKOUT_SUCCESS),
      // 失败操作
      () => commit(types.CHECKOUT_FAILURE, savedCartItems)
    )
  }
}
```

这里我们定义了一个 action，通过刚才的 registerAction 方法，我们注册了这个 action，这里的 commit 就是 store 的 API 接口，可以通过它在 action 里提交一个 mutation。state 对应的就是当前模块的 state，我们在这个 action 里即可以同步提交 mutation，也可以异步提交。接下来我们会从源码分析的角度来介绍这个 action 的回调是何时被调用的，参数是如何传递的。

我们有必要知道 action 的回调函数的调用时机，在 Vuex 中，action 的调用是通过 store 实例的 API 接口 dispatch 来调用的，来看一下 dispatch 函数的定义：

```
  dispatch (_type, _payload) {
    // check object-style dispatch
    const {
      type,
      payload
    } = unifyObjectStyle(_type, _payload)

    const action = { type, payload }
    const entry = this._actions[type]
    if (!entry) {
      if (process.env.NODE_ENV !== 'production') {
        console.error(`[vuex] unknown action type: ${type}`)
      }
      return
    }

    this._actionSubscribers.forEach(sub => sub(action, this.state))

    return entry.length > 1
      ? Promise.all(entry.map(handler => handler(payload)))
      : entry[0](payload)
  }
```

dispatch 支持2个参数，type 表示 action 的类型，payload 表示额外的参数。前面几行代码和 commit 接口非常类似，都是找到对应 type 下的 action 对象数组，唯一和 commit 不同的地方是最后部分，它对 action 的对象数组长度做判断，如果长度为 1 则直接调用 `entry[0](payload)`， 这个方法就是之前定义的 wrappedActionHandler(payload, cb)，执行它就相当于执行了 registerAction 注册的回调函数，并把当前模块的 context 和 额外参数 payload 作为参数传入。如果长度大于1，调用Promise.all()方法，异步的调用所有的回调函数。所以我们在 action 的回调函数里，可以拿到当前模块的上下文包括 store 的 commit 和 dispatch 方法、getter、当前模块的 state 和 rootState，可见 action 是非常灵活的。

#### registerGetter

了解完 registerAction 后，我们来看看 registerGetter的定义：

```
function registerGetter (store, type, rawGetter, local) {
  if (store._wrappedGetters[type]) {
    if (process.env.NODE_ENV !== 'production') {
      console.error(`[vuex] duplicate getter key: ${type}`)
    }
    return
  }
  store._wrappedGetters[type] = function wrappedGetter (store) {
    return rawGetter(
      local.state, // local state
      local.getters, // local getters
      store.state, // root state
      store.getters // root getters
    )
  }
}
```

registerGetter 是对 store 的 getters 初始化，它接受 4个 参数， store 表示当前 Store 实例，type 表示当前的命名空间的key，rawGetter表示当前模块下的单个getter, local 对应当前的模块上下文。而且会使用`store._wrappedGetters[type]`来判断key是否重复。

这个函数做的事情就是把每一个 getter 包装成一个方法，添加到 store._wrappedGetters 对象中，注意 getter 的 key 是不允许重复的。在这个包装的方法里，会执行 getter 的回调函数，并把当前模块的 state，store 的 getters 和 store 的 rootState ， store 的 getters作为它参数。来看一个例子：

```
export const cartProducts = state => {
  return state.cart.added.map(({ id, quantity }) => {
    const product = state.products.all.find(p => p.id === id)
    return {
      title: product.title,
      price: product.price,
      quantity
    }
  })
}
```

这里我们定义了一个 getter，通过刚才的 wrapGetters 方法，我们把这个 getter 添加到 `store._wrappedGetters` 对象里，这和回调函数的参数 state 对应的就是当前模块的 state，接下来我们从源码的角度分析这个函数是如何被调用，参数是如何传递的。

我们有必要知道 getter 的回调函数的调用时机，在 Vuex 中，我们知道当我们在组件中通过 `this.$store.getters.xxxgetters` 可以访问到对应的 getter 的回调函数，那么我们需要把对应 getter 的包装函数的执行结果绑定到 ``this.$store` 上。这部分的逻辑就在 resetStoreVM 函数里。我们在 Store 的构造函数中，在执行完 installModule 方法后，就会执行 resetStoreVM 方法。

#### resetStoreVM

```
function resetStoreVM (store, state, hot) {
  const oldVm = store._vm

  // bind store public getters
  store.getters = {}
  const wrappedGetters = store._wrappedGetters
  const computed = {}
  forEachValue(wrappedGetters, (fn, key) => {
    // use computed to leverage its lazy-caching mechanism
    computed[key] = () => fn(store)
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true // for local getters
    })
  })

  // use a Vue instance to store the state tree
  // suppress warnings just in case the user has added
  // some funky global mixins
  const silent = Vue.config.silent
  Vue.config.silent = true
  store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  })
  Vue.config.silent = silent

  // enable strict mode for new vm
  if (store.strict) {
    enableStrictMode(store)
  }

  if (oldVm) {
    if (hot) {
      // dispatch changes in all subscribed watchers
      // to force getter re-evaluation for hot reloading.
      store._withCommit(() => {
        oldVm._data.$$state = null
      })
    }
    Vue.nextTick(() => oldVm.$destroy())
  }
}
```

这个方法主要是重置一个私有的 _vm 对象，它是一个 Vue 的实例。这个 _vm 对象会保留我们的 state 树，以及用计算属性的方式存储了 store 的 getters。来具体看看它的实现过程。我们把这个函数拆成几个部分来分析：

```
 const oldVm = store._vm

  // bind store public getters
  store.getters = {}
  const wrappedGetters = store._wrappedGetters
  const computed = {}
  Object.keys(wrappedGetters).forEach(key => {
    const fn = wrappedGetters[key]
    // use computed to leverage its lazy-caching mechanism
    computed[key] = () => fn(store)
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key]
    })
  })
```

这部分留了现有的 `store._vm` 对象，接着遍历 `store._wrappedGetters`  对象，在遍历过程中，依次拿到每个 getter 的包装函数，并把这个包装函数执行的结果用 computed 临时变量保存。接着用 es5 的 Object.defineProperty 方法为 store.getters 定义了 get 方法，也就是当我们在组件中调用`this.$store.getters.xxxgetters` 这个方法的时候，会访问 `store._vm[xxxgetters]`。我们接着往下看：

```
// use a Vue instance to store the state tree
// suppress warnings just in case the user has added
 // some funky global mixins
 const silent = Vue.config.silent
 Vue.config.silent = true
 store._vm = new Vue({
   data: { state },
   computed
 })
 Vue.config.silent = silent

 // enable strict mode for new vm
 if (store.strict) {
   enableStrictMode(store)
 }
```

这部分的代码首先先拿全局 Vue.config.silent 的配置，然后临时把这个配置设成 true，接着实例化一个 Vue 的实例，把 store 的状态树 state 作为 data 传入，把我们刚才的临时变量 computed 作为计算属性传入。然后再把之前的 silent 配置重置。设置 silent 为 true 的目的是为了取消这个 `_vm` 的所有日志和警告。把 computed 对象作为 `_vm `的 computed 属性，这样就完成了 getters 的注册。因为当我们在组件中访问 `this.$store.getters.xxxgetters` 的时候，就相当于访问 `store._vm[xxxgetters]`，也就是在访问 computed[xxxgetters]，这样就访问到了 xxxgetters 对应的回调函数了。这段代码最后判断 strict 属性决定是否开启严格模式，我们来看看严格模式都干了什么：

```
function enableStrictMode (store) {
  store._vm.$watch(function () { return this._data.$$state }, () => {
    if (process.env.NODE_ENV !== 'production') {
      assert(store._committing, `Do not mutate vuex store state outside mutation handlers.`)
    }
  }, { deep: true, sync: true })
}
```

严格模式做的事情很简单，监测 `store._vm.state` 的变化，看看 state 的变化是否通过执行 mutation 的回调函数改变，如果是外部直接修改 state，那么 `store._committing` 的值为 false，这样就抛出一条错误。再次强调一下，Vuex 中对 state 的修改只能在 mutation 的回调函数里。

回到 resetStoreVM 函数，我们来看一下最后一部分：

```
if (oldVm) {
  // dispatch changes in all subscribed watchers
  // to force getter re-evaluation.
  store._withCommit(() => {
    oldVm.state = null
  })
  Vue.nextTick(() => oldVm.$destroy())
}
```

这里的逻辑很简单，由于这个函数每次都会创建新的 Vue 实例并赋值到 `store._vm` 上，那么旧的 _vm 对象的状态设置为 null，并调用 $destroy 方法销毁这个旧的 _vm 对象。

那么到这里，Vuex 的初始化基本告一段落了，初始化核心就是 installModule 和
resetStoreVM 函数。通过对 mutations 、actions 和 getters 的注册，我们了解到 state 的是按模块划分的，按模块的嵌套形成一颗状态树。而 actions、mutations 和 getters 的全局的，其中 actions 和 mutations 的 key 允许重复，但 getters 的 key 是不允许重复的。官方推荐我们给这些全局的对象在定义的时候加一个名称空间来避免命名冲突。



总结一下：

当我们定义好action、mutations、getters之后，使用new Vuex.Store()时，会调用构造函数Store的初始化方法，然后将传入的options进行注册，分别调用了registerXXX的方法，其实就是根据一定的规则将这些函数放到了实例对象对应的数组中。当我们调用dispatch、commit等方法时，本质上就是去找到这些回调函数，并触发。