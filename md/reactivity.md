# Vue3的响应式核心实现

响应式机制的主要功能就是：可以把普通的 JavaScript 对象封装成为响应式对象，拦截数据的获取和修改操作，实现依赖数据的自动化更新。（主要思想和vue2基本一致，只不过实现方式不同而已）

## 简要说明：Proxy和defineProperty的区别

+ Object.definePrototype是对对象上的属性进行新增或者修改, 有2种写法：
```js
const obj = {
    name: 'chrome'
}
// 数据描述符
Object.defineProperty(obj, 'age', {
    configurable: true, // 这个定义是否可以被delete
    enumerable: true, // 这个值是否可以被for in 枚举,或者Object.keys获取到
    writable: true, // 定义是否可以被修改
    value: '100'
})
// 访问器描述符
Object.defineProperty(obj, 'child', {
    configurable: true,
    enumerable: true,
    set(value) {
        console.log(value)
    },
    get() {
        console.log(this.value)
    }
})
```

+ Object.defineProperty对对象自身（属性）做修改, 而Proxy只是在Object基础上加一层拦截，不修改原对象

```js
var needProxyObj = {name: 'chrome', age:'800'}
var proxyObj = new Proxy(needProxyObj, {
    set(target, key, value, receiver) {
        consnole.log('proxy修改了', key, value)
    }
})
proxyObj.name = 'safari'; // proxy修改了 name safari
```
对比：

1. 监听不了数组的变化
2. 监听手段比较单一，只能监听 set 和 get , 而 Proxy 有10几种监听
3. 必须得把所有的属性全部添加defineProperty, Proxy对整个对象都会进行拦截
4. Proxy不用遍历每一个属性，它的代理（拦截）是对象级别的。
5. Proxy还可以监听数组的变化，比如push操作，会触发两次proxy的set方法（长度的变化和元素的新增）

## 聊聊reactive

其实，除了 reactive 还有很多别的函数需要实现，比如只读的响应式数据、浅层代理(对象的第一层属性注册响应式)的响应式数据等。本文主要说说reactive和ref的实现。

**提问**：reactive对数据的响应式代理是在何时进行的？？
答：在执行setup函数时进行的，也就是挂载组件，初始化有状态组件的时候进行的。

+ 使用：
```js
import { effect } from '../effect'
import { reactive } from '../reactive'

describe('测试响应式', () => {
  test('reactive基本使用', () => {
    const ret = reactive({ num: 0 })
    let val
    effect(() => {
      val = ret.num
    })
    expect(val).toBe(0)
    ret.num++
    expect(val).toBe(1)
    ret.num = 10
    expect(val).toBe(10)
  })
})
```

上例表示：ret被注册为响应式数据，effect函数中的fn参数，理解为对ret的依赖。当响应式数据修改时，对应的依赖会更新（即：effect函数中的fn会被执行，val会被修改）

+ reactive的内部实现如下：

```js
function reactive(target: object) {
    return createReactiveObject(target)
}

// map和weakMap的区别
const reactiveMap = new WeakMap(); // weakmap 弱引用   key必须是对象，如果key没有被引用可以被自动销毁。利于垃圾回收

function createReactiveObject(target: object) { 
    // 先默认认为这个target已经是代理过的属性了
    if ((target as any)[ReactiveFlags.IS_REACTIVE]) {
        return target
    }

    // reactiveApi 只针对对象才可以 
    if (!isObject(target)) {
        return target
    }

    // 如果缓存中有 直接使用上次代理的结果
    const exisitingProxy = reactiveMap.get(target); 
    if (exisitingProxy) {
        return exisitingProxy

    }
    // 对target做代理拦截。当用户获取属性 或者更改属性的时候，可以劫持到
    const proxy = new Proxy(target, mutableHandlers);

    // 将原对象和生成的代理对象 做一个映射表，做缓存
    reactiveMap.set(target, proxy); 

    return proxy; // 返回代理
}
```

+ 再说说mutableHandlers的实现：它要做的事就是配置 Proxy 的拦截函数，这里我们只拦截 get 和 set 操作。
    1. get 中直接返回读取的数据，这里的 Reflect.get 和 target[key]实现的结果是一致的；并且返回值是对象的话，还会嵌套执行 reactive。之后再进行调用 track 函数，收集依赖。
    2. set 中调用 trigger 函数，执行 track 收集的依赖。

```js
const get = createGetter();
const set = createSetter();

function createGetter(shallow = false) {
    // receiver是代理对象的本身
  return function get(target, key, receiver) {
    // target[key]
    const res = Reflect.get(target, key, receiver)
    track(target, "get", key)
    if (isObject(res)) {
      // 值也是对象的话，需要嵌套调用reactive
      // res就是target[key]
      // 浅层代理，不需要嵌套
      return shallow ? res : reactive(res)
    }
    return res
  }
}

function createSetter() {
  return function set(target, key, value, receiver) {
    let oldValue = target[key]

    const result = Reflect.set(target, key, value, receiver)
    if(oldValue !== value) {
        // 在触发 set 的时候进行触发依赖更新
        trigger(target, "set", key)
    }
    return result
  }
}
export const mutableHandles = {
  get,
  set,
};
```

+ track 函数是怎么完成依赖收集的: 

在 track 函数中，我们可以使用一个巨大的 tragetMap 去存储依赖关系。map 的 key 是我们要代理的 target 对象，值还是一个 depsMap，存储这每一个 key(target属性) 依赖的函数，每一个 key(target属性) 都可以依赖多个 effect。

依赖地图的格式，用代码描述如下：
```js
targetMap = {
 target： {
   key1: [回调函数1，回调函数2],
   key2: [回调函数3，回调函数4],
 }  ,
  target1： {
   key3: [回调函数5]
 }  
}
```

track函数的实现：由于 target 是对象，所以必须得用 map 才可以把 target 作为 key 来管理数据，每次操作之前需要做非空的判断。最终把 activeEffect 存储在集合之中。

```js
const targetMap = new WeakMap()

export function track(target, type, key) {
    // 首先明确入参：target要被代理的对象，type：get，key被读取的对象属性

  // console.log(`触发 track -> target: ${target} type:${type} key:${key}`)

  // 1. 先基于 target 找到对应的 dep
  // 如果是第一次的话，那么就需要初始化
  // {
  //   target1: {//depsmap
  //     key:[effect1,effect2]
  //   }
  // }
  let depsMap = targetMap.get(target)
  if (!depsMap) {
    // 初始化 depsMap 的逻辑
    // depsMap = new Map()
    // targetMap.set(target, depsMap)
    // 上面两行可以简写成下面的
    targetMap.set(target, (depsMap = new Map()))
  }
  let deps = depsMap.get(key)
  if (!deps) {
    deps = new Set()
  }
    // activeEffect是一个全局变量，在effect中的fn执行的时候，读取响应式数据的时候，就能在get配置里读取到这个activeEffect。
  if (!deps.has(activeEffect) && activeEffect) {
    // 防止重复注册
    deps.add(activeEffect)
  }
  depsMap.set(key, deps)
}
```

+ effect的实现：

我们把传递进来的 fn 函数通过 effectFn 函数包裹执行，在 effectFn 函数内部，把函数赋值给全局变量 activeEffect；然后执行 fn() 的时候，就会触发响应式对象的 get 函数，get 函数内部就会把 activeEffect 存储到依赖地图中，完成依赖的收集。

首次渲染的时候，effect会被自动执行一次，以完成响应式数据的读取，依赖收集。

**提问**：数据的读取具体在哪里进行的？？答：在首次执行effect.run时，内部执行组件更新函数时，调用组件的render函数生成虚拟dom时完成对数据的读取，从而进行依赖收集。

同时，可以配置lazy 和 scheduler 来控制函数执行的时机。稍后详细讲讲scheduler的使用。

```js
export function effect(fn, options = {}) {
  // effect嵌套，通过队列管理
  const effectFn = () => {
    try {
      activeEffect = effectFn
      //fn执行的时候，内部读取响应式数据的时候，就能在get配置里读取到activeEffect
      return fn()
    } finally {
      activeEffect = null
    }
  }
  if (!options.lazy) {
    //没有配置lazy 直接执行
    effectFn()
  }
  effectFn.scheduler = options.scheduler // 调度时机 watchEffect回用到
  return effectFn
}
```

+ trigger的实现

有了上面 targetMap 的实现机制，trigger 函数实现的思路就是从 targetMap 中，根据 target 和 key 找到对应的依赖函数集合 deps，然后遍历 deps 执行依赖函数effect。

```js
export function trigger(target, type, key) {
  // console.log(`触发 trigger -> target:  type:${type} key:${key}`)
  // 从targetMap中找到触发的函数，执行他
  const depsMap = targetMap.get(target)
  if (!depsMap) {
    // 没找到依赖
    return
  }
  const deps = depsMap.get(key)
  if (!deps) {
    return
  }
  // 遍历当前key对应的deps，如果effect没有配置的话。执行其中的effectFn，
  // 如果有配置scheduler，则执行scheduler()
  deps.forEach((effectFn) => {

    if (effectFn.scheduler) {
      effectFn.scheduler()
    } else {
      effectFn()
    }
  })
  
}
```

以上基本就是reactive的核心实现原理。

接下来再说说，**数据更新后的渲染**的大概流程：
+ 数据更新后，会触发数据代理的set函数，并对该数据的key对应的deps中的依赖（effect）进行依次调用执行。
+ effect.run的执行会触发componentUpdateFn（组件更新函数的执行）这其中由于已经是首次渲染过了所以，会执行更新的逻辑：获取新的响应式数据对应的虚拟dom，并调用patch函数（内部传入老vnode，新vnode，container）进行重新渲染，之后就是diff的流程了。

代码如下：

```js
const setupRenderEffect = (initialVNode, instance, container) => {

    const componentUpdateFn = () => {
        let { proxy } = instance; //  render中的参数
        if (!instance.isMounted) {
            // 组件初始化的流程
            // 调用组件的render方法 （渲染页面的时候会进行取值操作，那么取值的时候会进行依赖收集 ， 收集对应的effect，稍后属性变化了会重新执行当前方法）
            const subTree = instance.subTree = instance.render.call(proxy, proxy); // 渲染的时候会调用h方法
            // 真正渲染组件 其实渲染的应该是subTree
            patch(null, subTree, container); // 稍后渲染完subTree 会生成真实节点之后挂载到subTree
            initialVNode.el = subTree.el
            instance.isMounted = true;
        } else {
            // 组件更新的流程 。。。
            // 我可以做 diff算法   比较前后的两颗树 
            const prevTree = instance.subTree;
            const nextTree = instance.render.call(proxy, proxy);
            patch(prevTree, nextTree, container); // 比较两棵树
        }
    }
    const effect = new ReactiveEffect(componentUpdateFn);
    // 默认调用update方法 就会执行componentUpdateFn
    const update = effect.run.bind(effect);
    update();
}
```

### 再说说effect函数的执行时机的配置lazy和schedule

+ 首先effect的内部实现中，配置了lazy默认为false，意为：首次就会执行effect中入参的fn。
+ 重点说说schedule，再数据更新触发set函数中的trigger时。有个逻辑是：配置了scheduler的话，则执行effectFn.schedule().没有配置的话则执行effectFn。
+ 实际上，Vue3中对数据更新，渲染的过程，并不是同步执行的。源码中有配置schedule，来控制渲染过程。代码如下：
```js
instance.update = effect(componentEffect, {
  scheduler: () => {
    queueJob(instance.update)
  },
})
```
+ 说说这个queueJob：实际上就是在批量数据更新时，让渲染性能最大化。而不是数据一更新，就执行渲染过程。
+ 大致实现逻辑为：放置一个队列用来接收渲染任务，每当数据更新时，就将对应的要渲染的任务入队，如果是同一个数据的反复更新，则不进行多余的入队。设置一个标志位：`isFlushing` 用来控制渲染任务的执行.当同步代码都执行完毕后,再依次遍历执行异步微任务队列中的渲染任务.执行完成后,将`isFlushing`归位.

```js
// 任务的调度 queue收集任务，

const queue = []
let isFlushing = false
const resolvedPromise = Promise.resolve()
let currentPromise = null

export function nextTick(fn) {
  const p = currentPromise || resolvedPromise
  return fn ? p.then(fn) : p
}
// 任务入列
export function queueJob(job) {
  if (!queue.includes(job)) {
    queue.push(job)
    queueFlush()
  }
}

function queueFlush() {
  if (!isFlushing) {
    isFlushing = true
    currentPromise = resolvedPromise.then(flushJobs)
  }
}
// 执行所有任务
function flushJobs() {
  try {
    for (let i = 0 ;i < queue.length; i++) {
      const job = queue[i]
      job()
    }
  } finally {
    isFlushing = false
    queue.length = 0
    currentPromise = null
  }
}
```
