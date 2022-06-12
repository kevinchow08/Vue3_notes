## 说说computed的实现原理:

+ 首先, computed 可以传递一个函数或者对象，实现计算属性的读取和修改.

+ 与此同时,计算属性的读取并不是每一次都要执行getter函数,它会缓存之前依赖的响应式数据.
    + 当依赖的响应式数据没有发生变化时,获取computed的value时,是不需要再次执行getter(不需要再次计算)
    + 只有当计算属性依赖的响应式数据变化时,value才会重新获取.这一点也是实现computed的关键点.

由上我们可以得知:
+ computed的内部实现, 会用一个标志位(dirty)来判断当前计算属性依赖的响应式数据是否有变化.若dirty为false,则直接获取缓存值.
+ 首次读取到computed的value时,会收集该依赖. 同时,会执行getter方法,收集对应的数据依赖.将getter执行结果缓存起来,以便下一次直接使用.并且把dirty置为false,下一次不再执行getter,而是直接获取缓存值.
+ 只有当getter中依赖的数据变化时,则会触发更新,重新执行computed计算属性更改数据.

代码如下:

```js
export function computed(getterOrOptions) {
  // getterOrOptions可以是函数，也可以是一个对象，支持get和set
  // 还记得清单应用里的全选checkbox就是一个对象配置的computed
  let getter, setter
  if (typeof getterOrOptions === 'function') {
    getter = getterOrOptions
    setter = () => {
      console.warn('计算属性不能修改')
    }
  } else {
    getter = getterOrOptions.get
    setter = getterOrOptions.set
  }
  return new ComputedRefImpl(getter, setter)
}
class ComputedRefImpl {
  constructor(getter, setter) {
    this._setter = setter
    this._val = undefined
    this._dirty = true
    // computed就是一个特殊的effect，设置lazy和执行时机
    this.effect = effect(getter, {
      lazy: true,
      scheduler: () => {
        if (!this._dirty) {
          this._dirty = true
          trigger(this, 'value')
        }
      },
    })
  }
  get value() {
    track(this, 'value')
    if (this._dirty) {
      this._dirty = false
      this._val = this.effect()
    }
    return this._val
  }
  set value(val) {
    this._setter(val)
  }
}
```

## ref的内部实现

+ ref 函数实现就变得非常简单了。ref 的执行逻辑要比 reactive 要简单一些，不需要使用 Proxy 代理语法.
+ 直接使用class类的getter 和 setter 配置，监听 value 属性即可.
+ ref的本质其实也就是**将基本数据类型包装成响应式对象**, 但由于基本数据类型,不存在添加属性,删除属性.所以只需要getter和setter监听即可.
+ 在 ref 函数返回的对象中，对象的 get value 方法，使用 track 函数去收集依赖，set value 方法中使用 trigger 函数去触发函数的执行。

代码如下:
```js
export function ref(val) {
  if (isRef(val)) {
    return val
  }
  return new RefImpl(val)
}
export function isRef(val) {
  return !!(val && val.__isRef)
}

// ref就是利用面向对象的getter和setters进行track和trigget
class RefImpl {
  constructor(val) {
    this.__isRef = true
    this._val = convert(val)
  }
  get value() {
    track(this, 'value')
    return this._val
  }

  set value(val) {
    if (val !== this._val) {
      this._val = convert(val)
      trigger(this, 'value')
    }
  }
}

// ref也可以支持复杂数据结构
function convert(val) {
  return isObject(val) ? reactive(val) : val
}
```

值得一提的是，ref 也可以包裹复杂的数据结构，内部会直接调用 reactive 来实现