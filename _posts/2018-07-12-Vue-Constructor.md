---
layout: post
title: Vue源码学习笔记之Vue构造函数及其实例化
tags: [Vue, source, 源码]
comments: true
excerpt: 早听说，Vue 的源码极其优雅和精辟，再者自己使用 Vue 框架有段时间了，我想只停留在使用层面会大大局限我对它的认识，所以决定拜读源码尝试理解其中的原理。
cover: 
---





**学习资源**：vue@2.5.17

**引言**： 早听说，Vue 的源码极其优雅和精辟，再者自己使用 Vue 框架有段时间了，我想只停留在使用层面会大大局限我对它的认识，所以决定拜读源码尝试理解其中的原理。



### **入口文件**

首先从 package.json 入手，打开 package.json 找到： 

```json
"main": "dist/vue.runtime.common.js"
```

```json
"scripts": {
  "dev": "rollup -w -c scripts/config.js --environment TARGET:web-full-dev",
  "dev:cjs": "rollup -w -c scripts/config.js --environment TARGET:web-runtime-cjs",
  "dev:esm": "rollup -w -c scripts/config.js --environment TARGET:web-runtime-esm",
  ...
}
```

从 scripts 字段可以看到，项目使用 rollup 工具进行打包，我们需要找到找到编译打包成 dist/vue.runtime.common.js 的入口文件，从 rollup -w -c scripts/config.js 可以知道 scripts/config.js 应该是输出最终的配置，打开'scripts/config.js'：

```javascript
const builds = {
  // Runtime only (CommonJS). Used by bundlers e.g. Webpack & Browserify
  'web-runtime-cjs': {
    entry: resolve('web/entry-runtime.js'),
    dest: resolve('dist/vue.runtime.common.js'),
    format: 'cjs',
    banner
  },
  // Runtime+compiler CommonJS build (CommonJS)
  'web-full-cjs': {
    entry: resolve('web/entry-runtime-with-compiler.js'),
    dest: resolve('dist/vue.common.js'),
    format: 'cjs',
    alias: { he: './entity-decoder' },
    banner
  },
  ...
}
   
function genConfig (name) {
  const opts = builds[name]
  const config = {
    input: opts.entry,
    external: opts.external,
    plugins: [
      replace({
        __WEEX__: !!opts.weex,
        __WEEX_VERSION__: weexVersion,
        __VERSION__: version
      }),
      flow(),
      buble(),
      alias(Object.assign({}, aliases, opts.alias))
    ].concat(opts.plugins || []),
    output: {
      file: opts.dest,
      format: opts.format,
      banner: opts.banner,
      name: opts.moduleName || 'Vue'
    }
  }

  if (opts.env) {
    config.plugins.push(replace({
      'process.env.NODE_ENV': JSON.stringify(opts.env)
    }))
  }

  Object.defineProperty(config, '_name', {
    enumerable: false,
    value: name
  })

  return config
}
if (process.env.TARGET) {
  module.exports = genConfig(process.env.TARGET)
} else {
  exports.getBuild = genConfig
  exports.getAllBuilds = () => Object.keys(builds).map(genConfig)
}
```

可以看出 'dist/vue.runtime.common.js' 的入口文件是 'web/entry-runtime.js'。



### **Vue构造函数的创建**

**1⃣打开 'web/entry-runtime.js'：**

```javascript
import Vue from './runtime/index'

export default Vue
```

引入 './runtime/index' 的 Vue 并导出，Vue 就是最终暴露给用户的 Vue 构造函数。



**2⃣打开 './runtime/index'：**

```javascript
/* @flow */

import Vue from 'core/index'
import config from 'core/config'
import { extend, noop } from 'shared/util'
import { mountComponent } from 'core/instance/lifecycle'
import { devtools, inBrowser, isChrome } from 'core/util/index'

import {
  query,
  mustUseProp,
  isReservedTag,
  isReservedAttr,
  getTagNamespace,
  isUnknownElement
} from 'web/util/index'

import { patch } from './patch'
import platformDirectives from './directives/index'
import platformComponents from './components/index'

// install platform specific utils
Vue.config.mustUseProp = mustUseProp
Vue.config.isReservedTag = isReservedTag
Vue.config.isReservedAttr = isReservedAttr
Vue.config.getTagNamespace = getTagNamespace
Vue.config.isUnknownElement = isUnknownElement

// install platform runtime directives & components
extend(Vue.options.directives, platformDirectives)
extend(Vue.options.components, platformComponents)

// install platform patch function
Vue.prototype.__patch__ = inBrowser ? patch : noop

// public mount method
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}

...

export default Vue
```

1、引入 'core/index' 的 Vue

2、给 Vue 的 config 属性绑定了 mustUseProp、isReservedTag、isReservedAttr、getTagNamespace、isUnknownElement 属性

3、合并平台指令和组件

4、给 Vue.prototype 绑定 \__patch__ 方法

5、给 Vue.prototype 绑定 $mounted 方法

6、导出 Vue



**3⃣打开 'core/index'：**

```javascript
import Vue from './instance/index'
import { initGlobalAPI } from './global-api/index'
import { isServerRendering } from 'core/util/env'
import { FunctionalRenderContext } from 'core/vdom/create-functional-component'

initGlobalAPI(Vue)

Object.defineProperty(Vue.prototype, '$isServer', {
  get: isServerRendering
})

Object.defineProperty(Vue.prototype, '$ssrContext', {
  get () {
    /* istanbul ignore next */
    return this.$vnode && this.$vnode.ssrContext
  }
})

// expose FunctionalRenderContext for ssr runtime helper installation
Object.defineProperty(Vue, 'FunctionalRenderContext', {
  value: FunctionalRenderContext
})

Vue.version = '__VERSION__'

export default Vue
```

1、引入 './instance/index' 的 Vue

2、初始化全局 API，展开 initGlobalAPI 函数：

```javascript
export function initGlobalAPI (Vue: GlobalAPI) {
  // config
  const configDef = {}
  configDef.get = () => config
  if (process.env.NODE_ENV !== 'production') {
    configDef.set = () => {
      warn(
        'Do not replace the Vue.config object, set individual fields instead.'
      )
    }
  }
  Object.defineProperty(Vue, 'config', configDef)

  // exposed util methods.
  // NOTE: these are not considered part of the public API - avoid relying on
  // them unless you are aware of the risk.
  Vue.util = {
    warn,
    extend,
    mergeOptions,
    defineReactive
  }

  Vue.set = set
  Vue.delete = del
  Vue.nextTick = nextTick

  Vue.options = Object.create(null)
  ASSET_TYPES.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
  })

  // this is used to identify the "base" constructor to extend all plain-object
  // components with in Weex's multi-instance scenarios.
  Vue.options._base = Vue

  extend(Vue.options.components, builtInComponents)

  initUse(Vue)
  initMixin(Vue)
  initExtend(Vue)
  initAssetRegisters(Vue)
}
```

​	1、给 Vue 绑定 config 属性

​	2、给 Vue 绑定 util 属性

​	3、给 Vue 绑定 set 方法

​	4、给 Vue 绑定 delete 方法

​	5、给 Vue 绑定 nextTick 方法

​	6、给 Vue 绑定 options 属性并将其初始化为空对象

​	7、给 Vue 的 options 属性绑定 component、directive、filter 属性并将其初始化为空对象

​	8、给 Vue 的 options 属性绑定 _base 属性并赋值为 Vue

​	9、将 keep-alive 虚拟组件合并到 Vue.options.components

​	10、给 Vue 绑定 use 方法

​	11、给 Vue 绑定 mixin 方法

​	12、给 Vue 绑定 extend 方法

​	13、给 Vue 绑定 component、directive、filter 方法

3、给 Vue.prototype 绑定 $isServe 属性 

4、给 Vue.prototype 绑定 $ssrContext 属性

5、给 Vue 绑定 FunctionalRenderContext 属性

6、给 Vue 绑定 version 属性

7、导出 Vue



**4⃣打开 './instance/index'：**

```javascript
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'

function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)

export default Vue
```

这里主要做的是声明 Vue 构造函数，并设计 Vue 构造函数，通过五个 *mixin(Vue)，

initMixin(Vue) 

​	给 Vue.prototype 绑定 _init 方法

stateMixin(Vue) 

​	给 Vue.prototype 绑定 $data 属性并赋值为 this._data

​	给 Vue.prototype 绑定 $props 属性并赋值为 this._props

​	给 Vue.prototype 绑定 $set 方法

​	给 Vue.prototype 绑定 $delete 方法

​	给 Vue.prototype 绑定 $watch 方法

eventsMixin(Vue)

​	给 Vue.prototype 绑定 $on 方法

​	给 Vue.prototype 绑定 $once 方法

​	给 Vue.prototype 绑定 $off 方法

​	给 Vue.prototype 绑定 $emit 方法

lifecycleMixin(Vue)

​	给 Vue.prototype 绑定 _update 方法

​	给 Vue.prototype 绑定 $forceUpdate 方法

​	给 Vue.prototype 绑定 $destroy 方法

​	给 Vue.prototype 绑定 $emit 方法	

renderMixin(Vue)

​	执行 installRenderHelpers(Vue.prototype)

​	给 Vue.prototype 绑定 $nextTick 方法

​	给 Vue.prototype 绑定 _render 方法



**至此，我们知道以上过程创建了 Vue 构造函数对其加以设计，且为其绑定了一些全局方法和属性。**



### **实例化**

现在结合实际使用案例从正面看看实例化一个vm到底发生了什么。

以 vue-cli 初始化的项目的 main.js 为例：

```javascript
import Vue from 'vue'
import App from './App'

Vue.config.productionTip = false

/* eslint-disable no-new */
new Vue({
  el: '#app',
  components: { App },
  template: '<App/>'
})
```

当 new  Vue() 时会执行 this._init(options) 并生成一个vm实例。options 是我们给 Vue 传的参数，即：

```javascript
{
  el: '#app',
  components: { App },
  template: '<App/>'
}
```

\_init() 是在上面提到的 './instance/index' 中绑定在 Vue.prototype 上的方法，当调用 this._init() 时会执行该函数，我们来看一下做了什么，

```javascript
Vue.prototype._init = function (options?: Object) {
  const vm: Component = this
  // a uid
  vm._uid = uid++

  let startTag, endTag
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    startTag = `vue-perf-start:${vm._uid}`
    endTag = `vue-perf-end:${vm._uid}`
    mark(startTag)
  }

  // a flag to avoid this being observed
  vm._isVue = true
  // merge options
  if (options && options._isComponent) {
    // optimize internal component instantiation
    // since dynamic options merging is pretty slow, and none of the
    // internal component options needs special treatment.
    initInternalComponent(vm, options)
  } else {
    vm.$options = mergeOptions(
      resolveConstructorOptions(vm.constructor),
      options || {},
      vm
    )
  }
  /* istanbul ignore else */
  if (process.env.NODE_ENV !== 'production') {
    initProxy(vm)
  } else {
    vm._renderProxy = vm
  }
  // expose real self
  vm._self = vm
    
  initLifecycle(vm) // 在vm上挂载了一些生命周期及组件层级的属性
  initEvents(vm) // 在vm上挂载了一些事件相关的属性并作更新事件列表操作   
  initRender(vm) // 在vm上挂载了一些node相关的属性和方法 
  callHook(vm, 'beforeCreate') // 执行beforeCreate绑定的函数
  initInjections(vm) // resolve injections before data/props
  initState(vm)
  initProvide(vm) // resolve provide after data/props
  callHook(vm, 'created')

  // 如果el存在，挂载；如果不存在，则需要手动挂载
  if (vm.$options.el) {
    vm.$mount(vm.$options.el)
  }
}
```

_init 方法在 vm 上挂载了一些属性和方法，这里主要说一下 vm.\$options，vm.\$options 被赋值为 mergeOptions() 的返回值，先看看传入的参数都是什么，

```javascript
mergeOptions(
  resolveConstructorOptions(vm.constructor),
  options || {},
  vm
)
```

**argument1: ** resolveConstructorOptions(vm.constructor)，vm.constructor 其实就是 Vue 构造函数。	

接着看 resolveConstructorOptions 函数

```javascript
export function resolveConstructorOptions (Ctor: Class<Component>) {
  let options = Ctor.options
  if (Ctor.super) {
    ...
  }
  return options
}
```

这里简化了代码，Ctor.super 此时为 undefined， 所以 resolveConstructorOptions(Vue) 返回值为 Vue.options，还记得 Vue.options 是什么样吗？

```javascript
Vue.options = {
    components: {
        KeepAlive
    },
    directives: {},
    filters: {},
    _base: vm
}
```

**argument2: ** options 是实例化 vm 时传入构造函数Vue的参数，

```javascript
{
  el: '#app',
  components: { App },
  template: '<App/>'
}
```

**argument3：** vm 是当前实例

所以

```javascript
mergeOptions(
  resolveConstructorOptions(vm.constructor),
  options || {},
  vm
)
```

实际上是

```javascript
mergeOptions(
    {
        components: {KeepAlive},
        directives: {},
        filters: {},
        _base: vm
    },
    {
        el: '#app',
        components: { App },
        template: '<App/>'
    },
    vm
)
```

下面再来看 mergeOptions 函数到底做了什么

```javascript
export function mergeOptions (
  parent: Object,
  child: Object,
  vm?: Component
): Object {
  if (process.env.NODE_ENV !== 'production') {
    checkComponents(child)
  }

  // 如果child也是Vue构造函数
  if (typeof child === 'function') {
    child = child.options
  }
  // 格式化child.props
  normalizeProps(child, vm)
  // 格式化child.inject
  normalizeInject(child, vm)
  // 格式化child.directives
  normalizeDirectives(child)
    
  const extendsFrom = child.extends
  if (extendsFrom) {
    parent = mergeOptions(parent, extendsFrom, vm)
  }
  if (child.mixins) {
    for (let i = 0, l = child.mixins.length; i < l; i++) {
      parent = mergeOptions(parent, child.mixins[i], vm)
    }
  }
  const options = {}
  let key
  
  for (key in parent) {
    mergeField(key)
  }
    
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key)
    }
  }
  function mergeField (key) {
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
  }
  return options
}
```

这里需要注意的是不同的选项字段用的合并策略是不同的，

```javascript
function mergeField (key) {
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
}
```

从上面几行代码可以看出取决于 strats[key]，在 options.js 中可以找到 

```
strats.el = strats.propsData = function (parent, child, vm, key) {
  if (!vm) {
    warn(
      `option "${key}" can only be used during instance ` +
      'creation with the `new` keyword.'
    )
  }
  return defaultStrat(parent, child)
}
```

即 strats.el = strats.propsData = defaultStrat(parent, child)

```javascript
strats.data = function (
  parentVal: any,
  childVal: any,
  vm?: Component
): ?Function {
  if (!vm) {
    if (childVal && typeof childVal !== 'function') {
      process.env.NODE_ENV !== 'production' && warn(
        'The "data" option should be a function ' +
        'that returns a per-instance value in component ' +
        'definitions.',
        vm
      )
      return parentVal
    }
    return mergeDataOrFn(parentVal, childVal)
  }

  return mergeDataOrFn(parentVal, childVal, vm)
}
```

strats.data = mergeDataOrFn(parentVal, childVal, vm)

```javascript
LIFECYCLE_HOOKS.forEach(hook => {
  strats[hook] = mergeHook
})
```

生命周期选项使用的是 mergeHook 合并函数

```javascript
ASSET_TYPES.forEach(function (type) {
  strats[type + 's'] = mergeAssets
})
```

components、directives、filters 使用的是 mergeAssets 合并函数

```javascript
strats.watch = function (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): ?Object {
  // work around Firefox's Object.prototype.watch...
  if (parentVal === nativeWatch) parentVal = undefined
  if (childVal === nativeWatch) childVal = undefined
  /* istanbul ignore if */
  if (!childVal) return Object.create(parentVal || null)
  if (process.env.NODE_ENV !== 'production') {
    assertObjectType(key, childVal, vm)
  }
  if (!parentVal) return childVal
  const ret = {}
  extend(ret, parentVal)
  for (const key in childVal) {
    let parent = ret[key]
    const child = childVal[key]
    if (parent && !Array.isArray(parent)) {
      parent = [parent]
    }
    ret[key] = parent
      ? parent.concat(child)
      : Array.isArray(child) ? child : [child]
  }
  return ret
}
```

```javascript
strats.props =
strats.methods =
strats.inject =
strats.computed = function (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): ?Object {
  if (childVal && process.env.NODE_ENV !== 'production') {
    assertObjectType(key, childVal, vm)
  }
  if (!parentVal) return childVal
  const ret = Object.create(null)
  extend(ret, parentVal)
  if (childVal) extend(ret, childVal)
  return ret
}
```

strats.provide = mergeDataOrFn

其他选项字段的合并函数是

```javascript
const defaultStrat = function (parentVal: any, childVal: any): any {
  return childVal === undefined
    ? parentVal
    : childVal
}
```



### 总结

该篇文章仅记录了 Vue 构造函数设计和实例化的过程，从结构层面上对 Vue 有一个大概的了解，至于具体的方法是如何实现会后续会进行研究。
