Vuex 的 store 接受 `plugins` 选项，这个选项暴露出每次 mutation 的钩子。Vuex 插件就是一个函数，它接收 store 作为唯一参数。插件作用通常是用来监听每次 mutation 的变化，来做一些事情。

之前我们在Store的构造函数结尾，看到这样调用插件：

```
// apply plugins
plugins.forEach(plugin => plugin(this))

if (Vue.config.devtools) {
    devtoolPlugin(this)
 }
```

我们通常实例化 store 的时候，还会调用 logger 插件，代码如下：

```
import Vue from 'vue'
import Vuex from 'vuex'
import createLogger from 'vuex/dist/logger'

Vue.use(Vuex)

const debug = process.env.NODE_ENV !== 'production'

export default new Vuex.Store({
  ...
  plugins: debug ? [createLogger()] : []
})
```

在上述 2 个例子中，我们分别调用了 devtoolPlugin 和 createLogger() 2 个插件，它们是 Vuex 内置插件，我们接下来分别看一下他们的实现。

#### devtoolPlugin

devtoolPlugin 主要功能是利用 Vue 的开发者工具和 Vuex 做配合，通过开发者工具的面板展示 Vuex 的状态。它的源码在 `src/plugins/devtool.js` 中，来看一下这个插件到底做了哪些事情。

```
const devtoolHook =
  typeof window !== 'undefined' &&
  window.__VUE_DEVTOOLS_GLOBAL_HOOK__

export default function devtoolPlugin (store) {
  if (!devtoolHook) return

  store._devtoolHook = devtoolHook

  devtoolHook.emit('vuex:init', store)

  devtoolHook.on('vuex:travel-to-state', targetState => {
    store.replaceState(targetState)
  })

  store.subscribe((mutation, state) => {
    devtoolHook.emit('vuex:mutation', mutation, state)
  })
}
```

我们直接从对外暴露的 devtoolPlugin 函数看起，函数首先判断了devtoolHook 的值，如果我们浏览器装了 Vue 开发者工具，那么在 window 上就会有一个 `__VUE_DEVTOOLS_GLOBAL_HOOK__` 的引用， 那么这个 devtoolHook 就指向这个引用。

接下来通过 `devtoolHook.emit('vuex:init', store)` 派发一个 Vuex 初始化的事件，这样开发者工具就能拿到当前这个 store 实例。

接下来通过 `devtoolHook.on('vuex:travel-to-state', targetState => {    store.replaceState(targetState)  })`监听 Vuex 的 traval-to-state 的事件，把当前的状态树替换成目标状态树，这个功能也是利用 Vue 开发者工具替换 Vuex 的状态。

最后通过 `store.subscribe((mutation, state) => {    devtoolHook.emit('vuex:mutation', mutation, state)  })` 方法订阅 store 的 state 的变化，当 store 的 mutation 提交了 state 的变化， 会触发回调函数——通过 devtoolHook 派发一个 Vuex mutation 的事件，mutation 和 rootState 作为参数，这样开发者工具就可以观测到 Vuex state 的实时变化，在面板上展示最新的状态树。

#### loggerPlugin

通常在开发环境中，我们希望实时把 mutation 的动作以及 store 的 state 的变化实时输出，那么我们可以用 loggerPlugin 帮我们做这个事情。它的源码在` src/plugins/logger.js` 中，来看一下这个插件到底做了哪些事情。

```
// Credits: borrowed code from fcomb/redux-logger

import { deepCopy } from '../util'

export default function createLogger ({
  collapsed = true,
  filter = (mutation, stateBefore, stateAfter) => true,
  transformer = state => state,
  mutationTransformer = mut => mut,
  logger = console
} = {}) {
  return store => {
    let prevState = deepCopy(store.state)

    store.subscribe((mutation, state) => {
      if (typeof logger === 'undefined') {
        return
      }
      const nextState = deepCopy(state)

      if (filter(mutation, prevState, nextState)) {
        const time = new Date()
        const formattedTime = ` @ ${pad(time.getHours(), 2)}:${pad(time.getMinutes(), 2)}:${pad(time.getSeconds(), 2)}.${pad(time.getMilliseconds(), 3)}`
        const formattedMutation = mutationTransformer(mutation)
        const message = `mutation ${mutation.type}${formattedTime}`
        const startMessage = collapsed
          ? logger.groupCollapsed
          : logger.group

        // render
        try {
          startMessage.call(logger, message)
        } catch (e) {
          console.log(message)
        }

        logger.log('%c prev state', 'color: #9E9E9E; font-weight: bold', transformer(prevState))
        logger.log('%c mutation', 'color: #03A9F4; font-weight: bold', formattedMutation)
        logger.log('%c next state', 'color: #4CAF50; font-weight: bold', transformer(nextState))

        try {
          logger.groupEnd()
        } catch (e) {
          logger.log('—— log end ——')
        }
      }

      prevState = nextState
    })
  }
}

function repeat (str, times) {
  return (new Array(times + 1)).join(str)
}

function pad (num, maxLength) {
  return repeat('0', maxLength - num.toString().length) + num
}
```

插件对外暴露的是 createLogger 方法，它实际上接受 3 个参数，它们都有默认值，通常我们用默认值就可以。createLogger 的返回的是一个函数，当我执行 logger 插件的时候，实际上执行的是这个函数，下面来看一下这个函数做了哪些事情。

函数首先执行了 `let prevState = deepCopy(store.state)` 深拷贝当前 store 的 rootState。这里为什么要深拷贝，因为如果是单纯的引用，那么 store.state 的任何变化都会影响这个引用，这样就无法记录上一个状态了。我们来了解一下 deepCopy 的实现，在 src/util.js 里定义：

```
function find (list, f) {
  return list.filter(f)[0]
}

export function deepCopy (obj, cache = []) {
  // just return if obj is immutable value
  if (obj === null || typeof obj !== 'object') {
    return obj
  }

  // if obj is hit, it is in circular structure
  const hit = find(cache, c => c.original === obj)
  if (hit) {
    return hit.copy
  }

  const copy = Array.isArray(obj) ? [] : {}
  // put the copy into cache at first
  // because we want to refer it in recursive deepCopy
  cache.push({
    original: obj,
    copy
  })

  Object.keys(obj).forEach(key => {
    copy[key] = deepCopy(obj[key], cache)
  })

  return copy
}
```

deepCopy 并不陌生，很多开源库如 loadash、jQuery 都有类似的实现，原理也不难理解，主要是构造一个新的对象，遍历原对象或者数组，递归调用 deepCopy。不过这里的实现有一个有意思的地方，在每次执行 deepCopy 的时候，会用 cache 数组缓存当前嵌套的对象，以及执行 deepCopy 返回的 copy。如果在 deepCopy 的过程中通过 `find(cache, c => c.original === obj)` 发现有循环引用的时候，直接返回 cache 中对应的 copy，这样就避免了无限循环的情况。

回到 loggerPlugin 函数，通过 deepCopy 拷贝了当前 state 的副本并用 prevState 变量保存，接下来调用 store.subscribe 方法订阅 store 的 state 的变。 在回调函数中，也是先通过 deepCopy 方法拿到当前的 state 的副本，并用 nextState 变量保存。接下来获取当前格式化时间已经格式化的 mutation 变化的字符串，然后利用 console.group 以及 console.log 分组输出 prevState、mutation以及 nextState，这里可以通过我们 createLogger 的参数 collapsed、transformer 以及 mutationTransformer 来控制我们最终 log 的显示效果。在函数的最后，我们把 nextState 赋值给 prevState，便于下一次 mutation。



至此，我们的源码分析告一段落。
