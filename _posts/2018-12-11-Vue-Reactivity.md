---
layout: post
title: Vue源码学习笔记之响应式原理
tags: [Vue, 响应式, 源码]
comments: true
excerpt: 众所周知，Vue的数据响应式的实现结合了Object.defineProperty和发布订阅模式。本文旨在结合源码梳理响应式的过程。
cover: /images/reactivity.png
---


众所周知，Vue的数据响应式的实现结合了 `Object.defineProperty` 和发布订阅模式。本文旨在结合源码梳理响应式的过程。



#### Observer,Dep,Watcher

![数据响应式](/images/reactivity.png)

以上是我简单梳理的原理图，图中涉及到三个对象： Observer、Dep 和 Watcher。简单介绍一下它们： 

Observer：是发布订阅模式中的发布者，具体指的是Vue 中的数据对象（数组、对象），如 data、props。在Vue实例化的过程中这些数据对象会被递归地绑定一个 `__ob__` 属性，这个属性就是 Observer 的实例。

Dep：是发布订阅模式中的主题，Observer 和 Watcher 中间的纽带，每一个 Observer 都享有一个 Dep，用来存储订阅者 Watcher。

Watcher：是发布订阅模式中的订阅者，具体指的是 computed、watchs 以及模板编译过程中的指令和数据绑定。

以下，我们从源码一步一步分析。



#### 源码解析

在执行`new Vue()`时，首先会调用`this._init(options)`，我们来看一下源码（仅贴出概要部分）：

```javascript
// ...
initLifecycle(vm)
initEvents(vm)
initRender(vm)
callHook(vm, 'beforeCreate')
initInjections(vm) // resolve injections before data/props
initState(vm)
initProvide(vm) // resolve provide after data/props
callHook(vm, 'created')
// ...
if (vm.$options.el) {
  vm.$mount(vm.$options.el)
}
```

可以看出，这主要是执行 Vue 的生命周期函数，依次是 beforeCreate、created、beforeMount、mounted，在执行 created 生命周期函数之前执行了 `initState(vm)`，它主要是对 props、data、methods、computed、watch 进行处理，包含将数据对象转换成 Observer。



以 data 为例：

```javascript
function initData (vm: Component) {
  let data = vm.$options.data
  data = vm._data = typeof data === 'function' // 将vm._data指向vm.$options.data
    ? getData(data, vm)
    : data || {}
  if (!isPlainObject(data)) {
    data = {}
  }
  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) { // 判断键名是否被methods或props占用
    const key = keys[i]
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) {
      proxy(vm, `_data`, key) // 通过getter、setter将vm[key]代理到vm._data[key] 
    }
  }
  // observe data
  observe(data, true /* asRootData */)
}
```

这段代码比较好理解，我们来看最后以及最关键的一步 `observe(data, true /* asRootData */)` ，展开 `observe` 函数：

```javascript
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) { // 如果data不是对象或数组则返回
    return
  }
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__ // 如果data包含__ob__属性，则ob = value.__ob__
  } else if (
    observerState.shouldConvert &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    // 如果不包含，则新建一个Observer对象
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

 哇，看到了 Observer，没错就是它，图中的三大巨头之一。这个函数返回了一个 Observer 对象。展开 Observer 的构造函数：

```javascript
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that has this object as root $data

  constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      const augment = hasProto
        ? protoAugment
        : copyAugment
      augment(value, arrayMethods, arrayKeys)
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }

  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i], obj[keys[i]])
    }
  }

  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```

Observer 对象比较简洁，包含 `value`、`dep`、`vmCount` 三个属性，当其被实例化时，做了以下几件事情：

- 创建一个 Dep 对象
- 将自身绑定到目标数据对象的 `__ob__` 属性上
- 如果目标数据对象是数组，执行 `this.observeArray(value)` ，否则执行 `this.walk(value)`

先从对象说起吧，`this.walk(value)` 遍历了目标对象并执行 `defineReactive(obj, keys[i], obj[keys[i]])`，此方法是实现数据响应式的**核心**！

```javascript
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep() // 实例化一个Dep

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) { // 如果是不可配置的属性则返回
    return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get // 获取原本的getter
  const setter = property && property.set // 获取原本的setter

  let childOb = !shallow && observe(val) // 递归转换Observer
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () { // 扩展getter
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) { // 扩展setter
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      dep.notify()
    }
  })
}
```

`defineReactive` 就是通过 Object.defineProperty 改写对象属性的 get 和 set

get 做了以下几件事情：

- 获取属性值
- 如果 Dep.target 存在，执行 `dep.depend()`，也就是执行 `Dep.target.addDep(dep)`，先提一下 Dep.target 是 Watcher，所以实际执行的是 `dep.addSub(Dep.target)`，也就是 `dep.subs.push(Dep.target)`，意即将订阅者添加到 dep.subs 列表中，这个过程叫做**依赖收集**
  - 如果 childOb 存在，childOb 的 dep 也进行收集
    - 如果 value 是数组，则数组元素的 \__ob__.dep 也进行收集
- 返回属性值

set 做了以下几件事情：

- 对比新设置的值和旧的值，如果相等则返回
- 执行原本的 setter
- 对新设置的值进行 Observer 转换
- 执行 `dep.notify()` 通知订阅者更新

细心的你可能注意到，`defineReactive` 实例化了一个私有的 dep，这与 Observer 的 dep 属性有什么区别呢？

前者是开发者看不到的，而后者则是绑定在 Observer 上的，开发者可以看到，aha，这不是重点 : ) 。我猜想是因为 Array 没有 get 和 set 属性，Observer 的 dep 属性主要是为了 Array 服务的吧。

然后绕回来说说数组，`this.observeArray(value)` 就是遍历数组元素，然后将元素转换成 Observer，上面说了数组没有 get 和 set 属性，那它是怎么进行依赖收集和消息通知的呢。这就要从  `augment(value, arrayMethods, arrayKeys)` 说起了，代码片段如下：

```javascript
const augment = hasProto
  ? protoAugment
  : copyAugment
augment(value, arrayMethods, arrayKeys)
this.observeArray(value)
```

`protoAugment` 和 `copyAugment` 是两种给目标数组绑定变异方法的函数，用哪种取决于目标数组是否拥有 `__proto__` 属性。`arrayMethods` 是关联了 `Array.prototype` 原型的对象。

```javascript
const arrayProto = Array.prototype
export const arrayMethods = Object.create(arrayProto)

/**
 * Intercept mutating methods and emit events
 */
;[
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]
.forEach(function (method) {
  // cache original method
  const original = arrayProto[method]
  def(arrayMethods, method, function mutator (...args) {
    const result = original.apply(this, args)
    const ob = this.__ob__
    let inserted
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    if (inserted) ob.observeArray(inserted)
    // notify change
    ob.dep.notify()
    return result
  })
})
```

在 `arrayMethods` 上绑定了变异后的 'push'，'pop'，shift'，'unshift'，'splice'，'sort'，'reverse' 7种方法，在执行这7个方法之一时，会执行 `ob.dep.notify()` 去通知相关的 Watcher。

所以从对象和数组实现响应式的原理中，我们可以发现，直接通过下标新增的对象属性或通过索引改变的数组元素是不会触发视图更新的，这也是为什么 Vue 官方作出如下建议：

> 由于 JavaScript 的限制，Vue 不能检测以下变动的数组：
>
> 1. 当你利用索引直接设置一个项时，例如：`vm.items[indexOfItem] = newValue`
> 2. 当你修改数组的长度时，例如：`vm.items.length = newLength`
>
> 举个例子：
>
> ```
> var vm = new Vue({
>   data: {
>     items: ['a', 'b', 'c']
>   }
> })
> vm.items[1] = 'x' // 不是响应性的
> vm.items.length = 2 // 不是响应性的
> ```
>
> 为了解决第一类问题，以下两种方式都可以实现和 `vm.items[indexOfItem] = newValue` 相同的效果，同时也将触发状态更新：
>
> ```
> // Vue.set
> Vue.set(vm.items, indexOfItem, newValue)
> // Array.prototype.splice
> vm.items.splice(indexOfItem, 1, newValue)
> ```
>
> 你也可以使用 [`vm.$set`](https://vuejs.org/v2/api/#vm-set) 实例方法，该方法是全局方法 `Vue.set` 的一个别名：
>
> ```
> vm.$set(vm.items, indexOfItem, newValue)
> ```
>
> 为了解决第二类问题，你可以使用 `splice`：
>
> ```
> vm.items.splice(newLength)
> ```



如果你还记得的话，上文提到了 Dep.target，这是一个全局标志，标志当前正在被收集的 Watcher。刚开始我对此也非常疑惑，但是从 Watcher 着手就会很清楚，以 computed 为例：

```javascript
function initComputed (vm: Component, computed: Object) {
  const watchers = vm._computedWatchers = Object.create(null)
  // computed properties are just getters during SSR
  const isSSR = isServerRendering()

  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get

    if (!isSSR) {
      // create internal watcher for the computed property.
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }

    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    } 
  }
}
```

`initComputed` 函数主要做了以下几件事情：

- 遍历 `computed` 对象，获取每个属性的 getter
- 创建 Watcher
- 绑定 computed 属性到 vm 上

展开 Wather 的构造函数：

```javascript
export default class Watcher {
  // ...
  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: Object
  ) {
    this.vm = vm
    vm._watchers.push(this)
    // options
    if (options) {
      this.deep = !!options.deep
      this.user = !!options.user
      this.lazy = !!options.lazy
      this.sync = !!options.sync
    } else {
      this.deep = this.user = this.lazy = this.sync = false
    }
    this.cb = cb
    this.id = ++uid // uid for batching
    this.active = true
    this.dirty = this.lazy // for lazy watchers
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
    this.expression = process.env.NODE_ENV !== 'production'
      ? expOrFn.toString()
      : ''
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
      if (!this.getter) {
        this.getter = function () {}
        process.env.NODE_ENV !== 'production' && warn(
          `Failed watching path: "${expOrFn}" ` +
          'Watcher only accepts simple dot-delimited paths. ' +
          'For full control, use a function instead.',
          vm
        )
      }
    }
    this.value = this.lazy
      ? undefined
      : this.get()
  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   */
  get () {
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      value = this.getter.call(vm, vm)
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      if (this.deep) {
        traverse(value)
      }
      popTarget()
      this.cleanupDeps()
    }
    return value
  }
  // ...
}
```

我们重点看 `get` 方法，`get` 是给 `this.value` 赋值的方法，赋值实际是调用了 `value = this.getter.call(vm, vm)`，`this.getter` 是定义的计算属性函数，`get` 在获取值前首先执行了 `pushTarget(this)`

```javascript
Dep.target = null
const targetStack = []

export function pushTarget (_target: Watcher) {
  if (Dep.target) targetStack.push(Dep.target)
  Dep.target = _target
}

export function popTarget () {
  Dep.target = targetStack.pop()
}
```

假设 Dep.target = null，targetStack = []，Dep.target 就会被赋值为当前正在获取值的 Watcher，然后通过 getter 取得值，完成后执行了 popTarget，Dep.target 又重新被赋值为 null。而依赖收集就是在此时触发的，假设代码如下：

```javascript
export default {
    data() {
        return {
            a: 2,
            b: 3
        }
    },
    computed: {
        sum: {
            return a+b
        }
    }
}
```

这里 `sum` 享有一个 Watcher，`a `和 `b` 分别享有一个都是 Observer，当我们通过 `this.sum `访问 `sum` 的值时，会触发 'sum'（实际不是它本身，而是它享有的 Watcher ）的 get 函数，同时也会触发 `a` 和 `b` 的 get 函数，这里就保证了 `a` 和 `b` 在进行依赖收集时 Dep.target 为 'sum'。

最后我们看看 Observer 改变时，即 set 函数被触发时，是如何通知订阅者的？还是拿上面的例子，当我们执行  `this.a = 4` 时，`a `的 set 函数被触发，则 `a` 享有的 `dep` 就会去执行 `dep.notify()`，此时 `dep` 的 subs 列表只有一个 Watcher，那就是 'sum'，'sum' 就会执行 `update()`，'sum' 会重新计算自身的值并执行自己的 `cb` 函数，本例中 `cb` 即 `function sum(){ return a+b }` 



现在，回过头看看文章开始的原理图，是不是清晰了许多？这就是我对 Vue 响应式的理解和分析，诸多细节有待考证，如有不足请在评论指出，或通过邮箱发送给我，感谢。
