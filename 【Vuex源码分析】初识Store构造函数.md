当我们使用vuex时，我们一般会这么做：

```
//import ...

new Vuex.Store({
    actions,
    getters,
    state,
    mutations,
    modules: {},
    strict: debug,
    plugins: [myPlugin]
})
```

也就是说，我们是实例化了Vuex中的Store函数，然后传入一个对象，包含了已经定义好的actions、getters、state、mutations、plugins等，当我们还拥有子模块的时候，还会传入modules对象。

现在我们要思考，我们将这些传给Store构造函数后，他在内部做了什么事情？接下来，我们就来看一下Store构造函数。

```js
export class Store {
    constructor (options = {}) {
    // Auto install if it is not done yet and `window` has `Vue`.
    // To allow users to avoid auto-installation in some cases,
    // this code should be placed here. See #731
    if (!Vue && typeof window !== 'undefined' && window.Vue) {
      install(window.Vue)
    }

    if (process.env.NODE_ENV !== 'production') {
      assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`)
      assert(typeof Promise !== 'undefined', `vuex requires a Promise polyfill in this browser.`)
      assert(this instanceof Store, `Store must be called with the new operator.`)
    }

    const {
      plugins = [],
      strict = false
    } = options

    let {
      state = {}
    } = options
    if (typeof state === 'function') {
      state = state() || {}
    }

    // store internal state
    this._committing = false
    this._actions = Object.create(null)
    this._actionSubscribers = []
    this._mutations = Object.create(null)
    this._wrappedGetters = Object.create(null)
    this._modules = new ModuleCollection(options)
    this._modulesNamespaceMap = Object.create(null)
    this._subscribers = []
    this._watcherVM = new Vue()

    // bind commit and dispatch to self
    const store = this
    const { dispatch, commit } = this
    this.dispatch = function boundDispatch (type, payload) {
      return dispatch.call(store, type, payload)
    }
    this.commit = function boundCommit (type, payload, options) {
      return commit.call(store, type, payload, options)
    }

    // strict mode
    this.strict = strict

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
  }
  
  //...
  
}
```

先看一下构造函数constructor()部分源码：

首先会使用断言函数`assert()`来对一些条件进行校验。这里扩展一下`assert()`的源码，这个源码放在`/src/util.js`中,断言就是当不满足某些条件时，会抛出错误。

```js
export function assert (condition, msg) {
  if (!condition) throw new Error([vuex] ${msg})
}
```

言归正传，看一下着几个断言：

```js
assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`)
assert(typeof Promise !== 'undefined', `vuex requires a Promise polyfill in this browser.`)
assert(this instanceof Store, `Store must be called with the new operator.`)
```

第一个是确保创建实例前已经在Vue中注册了Vuex，也就是`Vue.use(Vuex)`,前面也说过，当我们在调用Vue.use()函数时，会调用Vuex的install()函数，来给 Vue 的实例注入初始化函数，这是初始化函数会在vue实例化过程中执行，然后给Vue实例注入一个 `$store` 的属性，保证我们可以通过通过 `this.$store.xxx` 访问到 Vuex 的各种数据和状态。所以如果在new Vue（）之后再去Vue.use(Vuex)的话，初始化函数并没有挂在到Vue上， `$store` 属性就无法添加到Vue的实例化对象上。

第二个是为了确保 Promsie 可以使用的，因为 Vuex 的源码是依赖 Promise 的。Promise 是 es6 提供新的 API，由于现在的浏览器并不是都支持 es6 语法的，所以通常我们会用 babel 编译我们的代码，如果想使用 Promise 这个 特性，我们需要在 package.json 中添加对 babel-polyfill 的依赖并在代码的入口加上 `import 'babel-polyfill'` 这段代码。

第三个是验证调用方是否是通过new出来的，也就是说是否是Store的原型，防止通过直接调用Stroe()这样，把Store当成普通函数来调用。

接下来看一下

```
const {
  plugins = [],
  strict = false
} = options

let {
      state = {}
    } = options
    
if (typeof state === 'function') {
   state = state() || {}
 }
```

利用ES6的解构赋值拿到的options中的plugins和strict。plugins 表示应用的插件、strict 表示是否开启严格模式。拿state时，会先解构拿到state，如果state是函数的话，会调用state()方法,拿到state。在下面关于state的官方API文档中也能看到state的定义。

```
state
类型: Object | Function
Vuex store 实例的根 state 对象。
如果你传入返回一个对象的函数，其返回的对象会被用作根 state。这在你想要重用 state 对象，尤其是对于重用 module 来说非常有用。详细介绍
```

接着源码往下看：

```js
	// store internal state
    this._committing = false
    this._actions = Object.create(null)
    this._actionSubscribers = []
    this._mutations = Object.create(null)
    this._wrappedGetters = Object.create(null)
    this._modules = new ModuleCollection(options)
    this._modulesNamespaceMap = Object.create(null)
    this._subscribers = []
    this._watcherVM = new Vue()
```

这里主要是创建一些内部的属性：
`this._committing` 标志一个提交状态，作用是保证对  Vuex 中 state 的修改只能在 mutation 的回调函数中，而不能在外部随意修改 state。

`this._actions` 用来存储用户定义的所有的 actions。

`this._actionSubscribers` 用来存储所有的 actions的订阅者。

`this._mutations` 用来存储用户定义所有的 mutatins。

`this._wrappedGetters` 用来存储用户定义的所有 getters 。

`this._modules` 用来存储用户定义的module。

`this._modulesNamespaceMap` 用来存储用户定义的modules的命名空间。

`this._subscribers` 用来存储所有对 mutation 变化的订阅者。

`this._watcherVM`  是一个 Vue 对象的实例，主要是利用 Vue 实例方法 $watch 来观测变化的

再往下看：

```js
// bind commit and dispatch to self
const store = this
const { dispatch, commit } = this
this.dispatch = function boundDispatch (type, payload) {
  return dispatch.call(store, type, payload)
}
this.commit = function boundCommit (type, payload, options) {
  return commit.call(store, type, payload, options)
}

// strict mode
this.strict = strict
```
这里的代码也不难理解，把 Store 类的 dispatch 和 commit 的方法的 this 指针指向当前 store 的实例上，dispatch 和 commit 的实现我们稍后会分析。this.strict 表示是否开启严格模式，在严格模式下会观测所有的 state 的变化，建议在开发环境时开启严格模式，线上环境要关闭严格模式，否则会有一定的性能开销。
