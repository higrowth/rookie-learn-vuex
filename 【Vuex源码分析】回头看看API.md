对比着[官方的API文档](https://vuex.vuejs.org/zh-cn/api.html)，我们再来梳理一下源码和API上面的联系:

### Vuex.Store

```
class Store {
    constructor (){
        //...
    }
    
    //...
}
```

就是一个构造函数。

### Vuex.Store 构造器选项

**state**  **mutations**  **actions**  **getters**  **modules**  **plugins**  **strict**

在Store构造函数的constructor统一以options传入。

### Vuex.Store 实例属性

**state**

```
get state () {
    return this._vm._data.$$state
  }

set state (v) {
    if (process.env.NODE_ENV !== 'production') {
      assert(false, `Use store.replaceState() to explicit replace store state.`)
    }
}
```

 Store构造函数中的set函数不会执行任何操作。只能get，不能set。

**getters**

```
Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true // for local getters
    })
```

在registerGetter时，会保存getters。当调用resetStoreVM时，会将其转化为一个只能get的对象。

### Vuex.Store 实例方法

- **commit(type: string, payload?: any, options?: Object)**

```
 commit (_type, _payload, _options) {
     //...
     
     const entry = this._mutations[type]
     
     //...
     
     this._withCommit(() => {
      entry.forEach(function commitIterator (handler) {
        handler(payload)
      })
    })
    
    //...
 }
```

其中会根据type在mutations数组中找到对应的回调函数，依次调用。调用的时候会使用`_withCommit`函数包裹，保证只能同步在mutations中修改state。

- **dispatch(type: string, payload?: any, options?: Object)**

```
 dispatch (_type, _payload) {
     //...
     
     const entry = this._actions[type]
     
     //...
     
     return entry.length > 1
      ? Promise.all(entry.map(handler => handler(payload)))
      : entry[0](payload)
     
 }
```

根据type找到actions，然后使用Promise异步的调用回调函数。这也是为什么dispatch可以异步执行的原因。

- **replaceState(state: Object)**

```
replaceState (state) {
    this._withCommit(() => {
      this._vm._data.$$state = state
    })
  }
```

使用`_withCommit`包裹，修改store的根状态。之所以提供这个 API 是由于在我们是不能在 muations 的回调函数外部去改变 state。

- **watch(getter: Function, cb: Function, options?: Object)**

```
 watch (getter, cb, options) {
    if (process.env.NODE_ENV !== 'production') {
      assert(typeof getter === 'function', `store.watch only accepts a function.`)
    }
    return this._watcherVM.$watch(() => getter(this.state, this.getters), cb, options)
  }
  //注意：this._watcherVM = new Vue()
```

函数首先断言 watch 的 getter 必须是一个方法，接着利用了内部一个 Vue 的实例对象 ``this._watcherVM` 的 $watch 方法，观测 getter 方法返回值的变化，如果有变化则调用 cb 函数，回调函数的参数为新值和旧值。watch 方法返回的是一个方法，调用它则取消观测。

- **subscribe(handler: Function)**

```
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

```
 commit (_type, _payload, _options) {
     //...
     
     this._subscribers.forEach(sub => sub(mutation, this.state))
     
     //...
 }

```

subscribe方法会调用genericSubscribe方法将其放到`_subscribers`中，每次通过commit触发mutations时，会触发`_subscribers`上保存的回调函数。

- **subscribeAction(handler: Function)**

```
subscribeAction (fn) {
    return genericSubscribe(fn, this._actionSubscribers)
  }
```

```
dispatch (_type, _payload) {
	//...
	
    this._actionSubscribers.forEach(sub => sub(action, this.state))
    
    //...
}
```

同subscribe一样，调用subscribeAction时会将回调函数保存到`_actionSubscribers`中，每次通过dispatch触发action时，会根据`_actionSubscribers`保存的回调函数依次触发。

- **registerModule(path: string | Array<string>, module: Module, options?: Object)**

```
registerModule (path, rawModule, options = {}) {
    if (typeof path === 'string') path = [path]

    if (process.env.NODE_ENV !== 'production') {
      assert(Array.isArray(path), `module path must be a string or an Array.`)
      assert(path.length > 0, 'cannot register the root module by using registerModule.')
    }

    this._modules.register(path, rawModule)
    installModule(this, this.state, path, this._modules.get(path), options.preserveState)
    // reset store to update getters...
    resetStoreVM(this, this.state)
  }
```

registerModule 的作用是注册一个动态模块，有的时候当我们异步加载一些业务的时候，可以通过这个 API 接口去动态注册模块。

函数首先对 path 判断，如果 path 是一个 string 则把 path 转换成一个 Array。接着把这个module注册到`_modules`中。`this._modules.register`方法如下：

```
 register (path, rawModule, runtime = true) {
    if (process.env.NODE_ENV !== 'production') {
      assertRawModule(path, rawModule)
    }

    const newModule = new Module(rawModule, runtime)
    if (path.length === 0) {
      this.root = newModule
    } else {
      const parent = this.get(path.slice(0, -1))
      parent.addChild(path[path.length - 1], newModule)
    }

    // register nested modules
    if (rawModule.modules) {
      forEachValue(rawModule.modules, (rawChildModule, key) => {
        this.register(path.concat(key), rawChildModule, runtime)
      })
    }
  }
```

接着和初始化 Store 的逻辑一样，调用  installModule 和 resetStoreVm 方法安装一遍动态注入的 module。

- **unregisterModule(path: string | Array<string>)**

```
unregisterModule (path) {
    if (typeof path === 'string') path = [path]

    if (process.env.NODE_ENV !== 'production') {
      assert(Array.isArray(path), `module path must be a string or an Array.`)
    }

    this._modules.unregister(path)
    this._withCommit(() => {
      const parentState = getNestedState(this.state, path.slice(0, -1))
      Vue.delete(parentState, path[path.length - 1])
    })
    resetStore(this)
  }
```

和 registerModule 方法相对的就是 unregisterModule 方法，它的作用是注销一个动态模块。函数首先还是对 path 的类型做了判断，这部分逻辑和注册是一样的。接着从  `_modules`里删掉模块。` this._modules.unregister(path)`方法如下：

```
unregister (path) {
    const parent = this.get(path.slice(0, -1))
    const key = path[path.length - 1]
    if (!parent.getChild(key).runtime) return

    parent.removeChild(key)
  }
```

接着通过 `this._withCommit` 方法把当前模块的 state 对象从父 state 上删除。这里用到了resetStore()，我们来看一下：

```
function resetStore (store, hot) {
  store._actions = Object.create(null)
  store._mutations = Object.create(null)
  store._wrappedGetters = Object.create(null)
  store._modulesNamespaceMap = Object.create(null)
  const state = store.state
  // init all modules
  installModule(store, state, [], store._modules.root, true)
  // reset vm
  resetStoreVM(store, state, hot)
}
```

这个方法作用就是重置 store 对象，重置 store 的 `_actions、_mutations、_wrappedGetters` 等等属性。然后再次调用 installModules 去重新安装一遍 Module 对应的这些属性，注意这里我们的最后一个参数 hot 为true，表示它是一次热更新。这样在 installModule 这个方法体类，如下这段逻辑就不会执行：

```
function installModule (store, rootState, path, module, hot) {
  ... 
  // set state
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1))
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
      Vue.set(parentState, moduleName, state || {})
    })
  }
  ...
}
```

由于 hot 始终为 true，这里我们就不会重新对状态树做设置，我们的 state 保持不变。因为我们已经明确的删除了对应 path 下的 state 了，要做的事情只不过就是重新注册一遍 muations、actions 以及 getters。

- **hotUpdate(newOptions: Object)**

```
 hotUpdate (newOptions) {
    this._modules.update(newOptions)
    resetStore(this, true)
  }
```

函数首先调用 update 方法去更新状态，其中要更新的 newOptions 会作为参数。来看一下这个函数的实现：

```
update (rawRootModule) {
    update([], this.root, rawRootModule)
  }
```



```
function update (path, targetModule, newModule) {
  if (process.env.NODE_ENV !== 'production') {
    assertRawModule(path, newModule)
  }

  // update target module
  targetModule.update(newModule)

  // update nested modules
  if (newModule.modules) {
    for (const key in newModule.modules) {
      if (!targetModule.getChild(key)) {
        if (process.env.NODE_ENV !== 'production') {
          console.warn(
            `[vuex] trying to add a new module '${key}' on hot reloading, ` +
            'manual reload is needed'
          )
        }
        return
      }
      update(
        path.concat(key),
        targetModule.getChild(key),
        newModule.modules[key]
      )
    }
  }
}
```

```
function assertRawModule (path, rawModule) {
  Object.keys(assertTypes).forEach(key => {
    if (!rawModule[key]) return

    const assertOptions = assertTypes[key]

    forEachValue(rawModule[key], (value, type) => {
      assert(
        assertOptions.assert(value),
        makeAssertionMessage(path, key, type, value, assertOptions.expected)
      )
    })
  })
}
```

首先我们调用assertRawModule方法module做了判断，并且更新 targetModule。最后判断如果 newOptions 包含 modules 这个 key，则遍历这个 modules 对象，如果 modules 对应的 key 不在之前的 modules 中，则报一条警告，因为这是添加一个新的 module ，需要手动重新加载。如果 key 在之前的 modules，则递归调用 updateModule，热更新子模块。调用完 update 后，回到 hotUpdate 函数，接着调用 resetStore 方法重新设置 store，刚刚我们已经介绍过了。



至此，我们从index.js入口文件触发，根据API将所有的源码大致看了一遍。后面，我们会对辅助函数进行了解，也就是`/src/helpers.js`文件。