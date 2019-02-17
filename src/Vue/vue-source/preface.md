# Vue 2.X 源码简析

## 前言

不久前，Vue的作者尤雨溪发布了Vue 3.0的开发线路，据介绍，新版本会对顶层的API进行重大调整，包括使用`Proxy`替代`Object.defineProperty`来实现对象的监听、渲染函数`render`中的`vdom`变动等。之前没能进行逐行地阅读源码，趁此希望在Vue 3.0版本发布之前，记录下Vue 2.X的阅读心得，以便将来甄别两者之间的差异，快速适应新版本。

## 项目结构

首先我们需要查看Vue的项目结构，了解各个文件目录的功能点以及之间的关联：

![](https://user-gold-cdn.xitu.io/2019/2/8/168ccdc02c74778b?w=320&h=637&f=png&s=18459)

其中主要关注`scripts`和`src`两个目录，`scripts`里包含了各个打包方式的配置文件，而`src`中：

* `src/compiler`：包含Vue的编译器，负责将字符串`template`解析成`AST`语法树，并根据不同的平台（如Weex、Web等）生成不同的代码。
* `src/core`：包含Vue的核心代码，负责Vue一系列静态与实例的属性、方法的添加等。
* `src/platforms`：包含各个平台下打包所需入口文件及相关配置。
* `src/server`：包含SSR下的相关文件。
* `src/sfc`：即Single File Component，最常用的单文件组件，主要负责单文件里的`template`、`script`和`style`的解析。
* `src/shared`：一些共用的常量和方法。

## Vue
了解了项目结构以后，我们需要从入口文件开始，沿着文件引用关系阅读源码。以我们常用的`module`引用文件`dist/vue.runtime.esm.js`为例，通过`scripts/config`，我们找到入口文件`platforms/web/entry-runtime.js`。根据文件依赖关系，在`src/core/instance/index.js`找到Vue的构造函数：

**src/core/instance/index.js**

```
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'

function Vue (options) {
  // ...
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)

export default Vue
```
可以看到，构造函数只是调用了`this._init(options)`方法，并在`export`之前调用5个`Mixin`。进一步查看这五个方法：

* `initMixin`方法是在Vue的原型上挂载了`_init`方法
```
export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    ...
  }
}
```

* `stateMixin`方法是对`$data`和`$props`做了代理，将其设为只读，并在Vue的原型上依次挂载了`$set`、`$del`和`$watch`方法

```
export function stateMixin (Vue: Class<Component>) {
  const dataDef = {}
  dataDef.get = function () { return this._data }
  const propsDef = {}
  propsDef.get = function () { return this._props }
  if (process.env.NODE_ENV !== 'production') {
    dataDef.set = function () {
      warn(
        'Avoid replacing instance root $data. ' +
        'Use nested data properties instead.',
        this
      )
    }
    propsDef.set = function () {
      warn(`$props is readonly.`, this)
    }
  }
  Object.defineProperty(Vue.prototype, '$data', dataDef)
  Object.defineProperty(Vue.prototype, '$props', propsDef)

  Vue.prototype.$set = set
  Vue.prototype.$delete = del
  Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: any,
    options?: Object
  ): Function {
    ...
  }
}
```

* `eventsMixin`方法是为Vue的原型挂载`$on`、`$once`、`off`和`emit`四个方法

```
export function eventsMixin (Vue: Class<Component>) {
  ...
  Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {
    ...
  }
  Vue.prototype.$once = function (event: string, fn: Function): Component {
    ...
  }
  Vue.prototype.$off = function (event?: string | Array<string>, fn?: Function): Component {
    ...
  }
  Vue.prototype.$emit = function (event: string): Component {
    ...
  }
}
```

* `lifecycleMixin`方法是为Vue的原型挂载`_update`、`$forceUpdate`和`$destroy`方法

```
export function lifecycleMixin (Vue: Class<Component>) {
  Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    ...
  }
  Vue.prototype.$forceUpdate = function () {
    ...
  }
  Vue.prototype.$destroy = function () {
    ...
  }
}
```

* `renderMixin`方法是为Vue的原型挂载`$nextTick`和`_render`方法，而`installRenderHelpers`是添加一系列其他方法。

```
export function renderMixin (Vue: Class<Component>) {
  // install runtime convenience helpers
  installRenderHelpers(Vue.prototype)
  Vue.prototype.$nextTick = function (fn: Function) {
    ...
  }
  Vue.prototype._render = function (): VNode {
    ...
  }
}
```

了解了这5个`Mixin`后，接下来我们将对Vue原型对象上这一系列的方法逐个进行解析，为方便阅读，将去掉源码中的**flow**类型检查。
