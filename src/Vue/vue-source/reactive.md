作为MVVM框架的一种，Vue最为人津津乐道的当是数据与视图的绑定，将直接操作DOM节点变为修改`data`数据，利用`Virtual Dom`来`Diff`对比新旧视图，从而实现更新。不仅如此，还可以通过`Vue.prototype.$watch`来监听`data`的变化并执行回调函数，实现自定义的逻辑。虽然日常的编码运用已经驾轻就熟，但未曾去深究技术背后的实现原理。作为一个好学的程序员，知其然更要知其所以然，本文将从源码的角度来对Vue响应式数据中的观察者模式进行简析。

## 初始化`Vue`实例

在阅读源码时，因为文件繁多，引用复杂往往使我们不容易抓住重点，这里我们需要找到一个入口文件，从`Vue`构造函数开始，抛开其他无关因素，一步步理解响应式数据的实现原理。首先我们找到`Vue`构造函数：

``` javascript
// src/core/instance/index.js
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}
````

``` javascript
// src/core/instance/init.js
Vue.prototype._init = function (options) {
    ...
    // a flag to avoid this being observed
    vm._isVue = true
    // merge options
    // 初始化vm实例的$options
    if (options && options._isComponent) {
        initInternalComponent(vm, options)
    } else {
        vm.$options = mergeOptions(
            resolveConstructorOptions(vm.constructor),
            options || {},
            vm
        )
    }
    ...
    initLifecycle(vm) // 梳理实例的parent、root、children和refs，并初始化一些与生命周期相关的实例属性
    initEvents(vm) // 初始化实例的listeners
    initRender(vm) // 初始化插槽，绑定createElement函数的vm实例
    callHook(vm, 'beforeCreate')
    initInjections(vm) // resolve injections before data/props
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')
    
    if (vm.$options.el) {
        vm.$mount(vm.$options.el)  // 挂载组件到节点
    }
}
```

为了方便阅读，我们去除了`flow`类型检查和部分无关代码。可以看到，在实例化Vue组件时，会调用`Vue.prototype._init`，而在方法内部，数据的初始化操作主要在`initState`（这里的`initInjections`和`initProvide`与`initProps`类似，在理解了`initState`原理后自然明白），因此我们重点来关注`initState`。

``` javascript
// src/core/instance/state.js
export function initState (vm) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

首先初始化了一个`_watchers`数组，用来存放`watcher`，之后根据实例的`vm.$options`，相继调用`initProps`、`initMethods`、`initData`、`initComputed`和`initWatch`方法。

##  `initProps`

``` javascript
function initProps (vm, propsOptions) {
  const propsData = vm.$options.propsData || {}
  const props = vm._props = {}
  // cache prop keys so that future props updates can iterate using Array
  // instead of dynamic object key enumeration.
  const keys = vm.$options._propKeys = []
  const isRoot = !vm.$parent
  // root instance props should be converted
  if (!isRoot) {
    toggleObserving(false)
  }
  for (const key in propsOptions) {
    keys.push(key)
    const value = validateProp(key, propsOptions, propsData, vm)
    ...
    defineReactive(props, key, value)
    if (!(key in vm)) {
      proxy(vm, '_props', key)
    }
  }
  toggleObserving(true)
}
```

在这里，`vm.$options.propsData`是通过父组件传给子组件实例的数据对象，如`<my-element :item="false"></my-element>`中的`{item: false}`，然后初始化`vm._props`和`vm.$options._propKeys`分别用来保存实例的`props`数据和`keys`，因为子组件中使用的是通过`proxy`引用的`_props`里的数据，而不是父组件传递的`propsData`，所以这里缓存了`_propKeys`，用来`updateChildComponent`时能更新`vm._props`。接着根据`isRoot`是否是根组件来判断是否需要调用`toggleObserving(false)`，这是一个全局的开关，来控制是否需要给对象添加`__ob__`属性。这个相信大家都不陌生，一般的组件的`data`等数据都包含这个属性，这里先不深究，等之后和`defineReactive`时一起讲解。因为`props`是通过父传给子的数据，在父元素`initState`时已经把`__ob__`添加上了，所以在不是实例化根组件时关闭了这个全局开关，待调用结束前在通过`toggleObserving(true)`开启。

之后是一个`for`循环，根据组件中定义的`propsOptions`对象来设置`vm._props`，这里的`propsOptions`就是我们常写的

``` javascript
export default {
    ...
    props: {
        item: {
            type: Object,
            default: () => ({})
        }
    }
}
```

循环体内，首先

``` javascript
const value = validateProp(key, propsOptions, propsData, vm)
```

`validateProp`方法主要是校验数据是否符合我们定义的`type`，以及在`propsData`里未找到`key`时，获取默认值并在对象上定义`__ob__`，最后返回相应的值，在这里不做展开。

这里我们先跳过`defineReactive`，看最后

``` javascript
if (!(key in vm)) {
  proxy(vm, '_props', key)
}
```

其中`proxy`方法:

``` javascript
function proxy (target, sourceKey, key) {
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

在`vm`不存在`key`属性时，通过`Object.defineProperty`使得我们能通过`vm[key]`访问到`vm._props[key]`。

## `defineReactive`

在`initProps`中，我们了解到其首先根据用户定义的`vm.$options.props`对象，通过对父组件设置的传值对象`vm.$options.propsData`进行数据校验，返回有效值并保存到`vm._props`，同时保存相应的`key`到`vm.$options._propKeys`以便进行子组件的`props`数据更新，最后利用`getter/setter`存取器属性，将`vm[key]`指向对`vm._props[key]`的操作。但其中跳过了最重要的`defineReactive`，现在我们将通过阅读`defineReactive`源码，了解响应式数据背后的实现原理。

``` javascript
// src/core/observer/index.js
export function defineReactive (
  obj,
  key,
  val,
  customSetter,
  shallow
) {
  const dep = new Dep()

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  let childOb = !shallow && observe(val)
  ...
}
```

首先`const dep = new Dep()`实例化了一个`dep`，在这里利用闭包来定义一个依赖项，用以与特定的`key`相对应。因为其通过`Object.defineProperty`重写`target[key]`的`getter/setter`来实现数据的响应式，因此需要先判断对象`key`的`configurable`属性。接着

``` javascript
if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
}
```

`arguments.length === 2`意味着调用`defineReactive`时未传递`val`值，此时`val`为`undefined`，而`!getter || setter`判断条件则表示如果在`property`存在`getter`且不存在`setter`的情况下，不会获取`key`的数据对象，此时`val`为`undefined`，之后调用`observe`时将不对其进行深度观察。正如之后的`setter`访问器中的：

``` javascript
if (getter && !setter) return
```

此时数据将是只读状态，既然是只读状态，则不存在数据修改问题，继而无须深度观察数据以便在数据变化时调用观察者注册的方法。

### `Observe`

在`defineReactive`里，我们先获取了`target[key]`的`descriptor`，并缓存了对应的`getter`和`setter`，之后根据判断选择是否获取`target[key]`对应的`val`，接着是

``` javascript
let childOb = !shallow && observe(val)
```

根据`shallow`标志来确定是否调用`observe`，我们来看下`observe`函数：

``` javascript
// src/core/observer/index.js
export function observe (value, asRootData) {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

首先判断需要观察的数据是否为对象以便通过`Object.defineProperty`定义`__ob__`属性，同时需要`value`不属于`VNode`的实例(`VNode`实例通过`Diff`补丁算法来实现实例对比并更新)。接着判断`value`是否已有`__ob__`，如果没有则进行后续判断：

* `shouldObserve`：全局开关标志，通过`toggleObserving`来修改。
* `!isServerRendering()`：判断是否服务端渲染。
* `(Array.isArray(value) || isPlainObject(value))`：数组和纯对象时才允许添加`__ob__`进行观察。
* `Object.isExtensible(value)`：判断`value`是否可扩展。
* `!value._isVue`：避免`Vue`实例被观察。

满足以上五个条件时，才会调用`ob = new Observer(value)`，接下来我们要看下`Observer`类里做了哪些工作

``` javascript
// src/core/observer/index.js
export class Observer {
  constructor (value) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      if (hasProto) {
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }

  /**
   * Walk through all properties and convert them into
   * getter/setters. This method should only be called when
   * value type is Object.
   */
  walk (obj) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  /**
   * Observe a list of Array items.
   */
  observeArray (items) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```

构造函数里初始化了`value`、`dep`和`vmCount`三个属性，为`this.value`添加`__ob__`对象并指向自己，即`value.__ob__.value === value`，这样就可以通过`value`或`__ob__`对象取到`dep`和`value`。`vmCount`的作用主要是用来区分是否为`Vue`实例的根`data`，`dep`的作用这里先不介绍，待与`getter/setter`里的`dep`一起解释。

接着根据`value`是数组还是纯对象来分别调用相应的方法，对`value`进行递归操作。当`value`为纯对象时，调用`walk`方法，递归调用`defineReactive`。当`value`是数组类型时，首先判断是否有`__proto__`，有就使用`__proto__`实现原型链继承，否则用`Object.defineProperty`实现拷贝继承。其中继承的基类`arrayMethods`来自`src/core/observer/array.js`：

``` javascript
// src/core/observer/array.js
const arrayProto = Array.prototype
export const arrayMethods = Object.create(arrayProto)

const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]

methodsToPatch.forEach(function (method) {
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

这里为什么要对数组的实例方法进行重写呢？代码里的`methodsToPatch`这些方法并不会返回新的数组，导致无法触发`setter`，因而不会调用观察者的方法。所以重写了这些变异方法，使得在调用的时候，利用`observeArray`对新插入的数组元素添加`__ob__`，并能够通过`ob.dep.notify`手动通知对应的被观察者执行注册的方法，实现数组元素的响应式。

``` javascript
if (asRootData && ob) {
    ob.vmCount++
}
```

最后添加这个`if`判断，在`Vue`实例的根`data`对象上，执行`ob.vmCount++`，这里主要为了后面根据`ob.vmCount`来区分是否为根数据，从而在其上执行`Vue.set`和`Vue.delete`。

### `getter/setter`

在对`val`进行递归操作后(假如需要的话)，将`obj[key]`的数据对象封装成了一个被观察者，使得能够被观察者观察，并在需要的时候调用观察者的方法。这里通过`Object.defineProperty`重写了`obj[key]`的访问器属性，对`getter/setter`操作做了拦截处理，`defineReactive`剩余的代码具体如下：

``` javascript
...
Object.defineProperty(obj, key, {
  enumerable: true,
  configurable: true,
  get: function reactiveGetter () {
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
  set: function reactiveSetter (newVal) {
    ...
    childOb = !shallow && observe(newVal)
    dep.notify()
  }
})
```

首先在`getter`调用时，判断`Dep.target`是否存在，若存在则调用`dep.depend`。我们先不深究`Dep.target`，只当它是一个观察者，比如我们常用的某个计算属性，调用`dep.depend`会将`dep`当做计算属性的依赖项存入其依赖列表，并把这个计算属性注册到这个`dep`。这里为什么需要互相引用呢？这是因为一个`target[key]`可以充当多个观察者的依赖项，同时一个观察者可以有多个依赖项，他们之间属于多对多的关系。这样当某个依赖项改变时，我们可以根据`dep`里维护的观察者，调用他们的注册方法。现在我们回过头来看`Dep`：

``` javascript
// src/core/observer/dep.js
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    ...
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}
```

构造函数里，首先添加一个自增的`uid`用以做`dep`实例的唯一性标志，接着初始化一个观察者列表`subs`，并定义了添加观察者方法`addSub`和移除观察者方法`removeSub`。可以看到其在`getter`中调用的`depend`会将当前这个`dep`实例添加到观察者的依赖项，在`setter`里调用的`notify`会执行各个观察者注册的`update`方法，`Dep.target.addDep`这个方法将在之后的`Watcher`里进行解释。简单来说就是会在`key`的`getter`触发时进行`dep`依赖收集到`watcher`并将`Dep.target`添加到当前`dep`的观察者列表，这样在`key`的`setter`触发时，能够通过观察者列表，执行观察者的`update`方法。

当然，在`getter`中还有如下几行代码：

``` javascript
if (childOb) {
    childOb.dep.depend()
    if (Array.isArray(value)) {
        dependArray(value)
    }
}
```

这里可能会有疑惑，既然已经调用了`dep.depend`，为什么还要调用`childOb.dep.depend`？两个`dep`之间又有什么关系呢？

其实这两个`dep`的分工是不同的。对于数据的增、删，利用`childOb.dep.notify`来调用观察者方法，而对于数据的修改，则使用的`dep.notify`，这是因为`setter`访问器无法监听到对象数据的添加和删除。举个例子：

``` javascript
const data = {
    arr: [{
        value: 1
    }],
}

data.a = 1; // 无法触发setter
data.arr[1] = {value: 2}; // 无法触发setter
data.arr.push({value: 3}); // 无法触发setter
data.arr = [{value: 4}]; // 可以触发setter
```

还记得`Observer`构造函数里针对数组类型`value`的响应式转换吗？通过重写`value`原型链，使得对于新插入的数据：

``` javascript
if (inserted) ob.observeArray(inserted)
// notify change
ob.dep.notify()
```

将其转换为响应式数据，并通过`ob.dep.notify`来调用观察者的方法，而这里的观察者列表就是通过上述的`childOb.dep.depend`来收集的。同样的，为了实现对象新增数据的响应式，我们需要提供相应的`hack`方法，而这就是我们常用的`Vue.set/Vue.delete`。


``` javascript
// src/core/observer/index.js
export function set (target: Array<any> | Object, key: any, val: any): any {
  ...
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.length = Math.max(target.length, key)
    target.splice(key, 1, val)
    return val
  }
  if (key in target && !(key in Object.prototype)) {
    target[key] = val
    return val
  }
  const ob = (target: any).__ob__
  if (target._isVue || (ob && ob.vmCount)) {
    process.env.NODE_ENV !== 'production' && warn(
      'Avoid adding reactive properties to a Vue instance or its root $data ' +
      'at runtime - declare it upfront in the data option.'
    )
    return val
  }
  if (!ob) {
    target[key] = val
    return val
  }
  defineReactive(ob.value, key, val)
  ob.dep.notify()
  return val
}
```

* 判断`value`是否为数组，如果是，直接调用已经`hack`过的`splice`即可。
* 是否已存在`key`，有的话说明已经是响应式了，直接修改即可。
* 接着判断`target.__ob__`是否存在，如果没有说明该对象无须深度观察，设置返回当前的值。
* 最后，通过`defineReactive`来设置新增的`key`，并调用`ob.dep.notify`通知到观察者。

现在我们了解了`childOb.dep.depend()`是为了将当前`watcher`收集到`childOb.dep`，以便在增、删数据时能通知到`watcher`。而在`childOb.dep.depend()`之后还有：

``` javascript
if (Array.isArray(value)) {
    dependArray(value)
}
```

``` javascript
/**
 * Collect dependencies on array elements when the array is touched, since
 * we cannot intercept array element access like property getters.
 */
function dependArray (value: Array<any>) {
  for (let e, i = 0, l = value.length; i < l; i++) {
    e = value[i]
    e && e.__ob__ && e.__ob__.dep.depend()
    if (Array.isArray(e)) {
      dependArray(e)
    }
  }
}
```

在触发`target[key]`的`getter`时，如果`value`的类型为数组，则递归将其每个元素都调用`__ob__.dep.depend`，这是因为无法拦截数组元素的`getter`，所以将当前`watcher`收集到数组下的所有`__ob__.dep`，这样当其中一个元素触发增、删操作时能通知到观察者。比如：

``` javascript
const data = {
    list: [[{value: 0}]],
};
data.list[0].push({value: 1});
```

这样在`data.list[0].__ob__.notify`时，才能通知到`watcher`。

`target[key]`的`getter`主要作用：

* 将`Dep.target`收集到闭包中`dep`的观察者列表，以便在`target[key]`的`setter`修改数据时通知观察者
* 根据情况对数据进行遍历添加`__ob__`，将`Dep.target`收集到`childOb.dep`的观察者列表，以便在增加/删除数据时能通知到观察者
* 通过`dependArray`将数组型的`value`递归进行观察者收集，在数组元素发生增、删、改时能通知到观察者

`target[key]`的`setter`主要作用是对新数据进行观察，并通过闭包保存到`childOb`变量供`getter`使用，同时调用`dep.notify`通知观察者，在此就不再展开。

## `Watcher`

在前面的篇幅中，我们主要介绍了`defineReactive`来定义响应式数据：通过闭包保存`dep`和`childOb`，在`getter`时来进行观察者的收集，使得在数据修改时能触发`dep.notify`或`childOb.dep.notify`来调用观察者的方法进行更新。但具体是如何进行`watcher`收集的却未做过多解释，现在我们将通过阅读`Watcher`来了解观察者背后的逻辑。

``` javascript
function initComputed (vm: Component, computed: Object) {
  const watchers = vm._computedWatchers = Object.create(null)
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
    ...
  }
}
```

这是`Vue`计算属性的初始化操作，去掉了一部分不影响的代码。首先初始化对象`vm._computedWatchers`用以存储所有的计算属性，`isSSR`用以判断是否为服务端渲染。再根据我们编写的`computed`键值对循环遍历，如果不是服务端渲染，则为每个计算属性实例化一个`Watcher`，并以键值对的形式保存到`vm._computedWatchers`对象，接下来我们主要看下`Watcher`这个类。

### `Watcher`的构造函数

构造函数接受5个参数，其中当前`Vue`实例`vm`、求值表达式`expOrFn`(支持`Function`或者`String`，计算属性中一般为`Function`)，回调函数`cb`这三个为必传参数。设置`this.vm = vm`用以后续绑定`this.getter`的执行环境，并将`this`推入`vm._watchers`(`vm._watchers`用以维护实例`vm`中所有的观察者)，另外根据是否为渲染观察者来赋值`vm._watcher = this`(常用的`render`即为渲染观察者)。接着根据`options`进行一系列的初始化操作。其中有几个属性：

* `this.lazy`：设置是否懒求值，这样能保证有多个被观察者发生变化时，能只调用求值一次。
* `this.dirty`：配合`this.lazy`，用以标记当前观察者是否需要重新求值。
* `this.deps`、`this.newDeps`、`this.depIds`、`this.newDepIds`：用以维护被观察对象的列表。
* `this.getter`：求值函数。
* `this.value`：求值函数返回的值，即为计算属性中的值。

### `Watcher`的求值

因为计算属性是惰性求值，所以我们继续看`initComputed`循环体：

``` javascript
if (!(key in vm)) {
  defineComputed(vm, key, userDef)
}
```

`defineComputed`主要将`userDef`转化为`getter/setter`访问器，并通过`Object.defineProperty`将`key`设置到`vm`上，使得我们能通过`this[key]`直接访问到计算属性。接下来我们主要看下`userDef`转为`getter`中的`createComputedGetter`函数：

``` javascript
function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```

利用闭包保存计算属性的`key`，在`getter`触发时，首先通过`this._computedWatchers[key]`获取到之前保存的`watcher`，如果`watcher.dirty`为`true`时调用`watcher.evaluate`(执行`this.get()`求值操作，并将当前`watcher`的`dirty`标记为`false`)，我们主要看下`get`操作：

``` javascript
get () {
  pushTarget(this)
  let value
  const vm = this.vm
  try {
    value = this.getter.call(vm, vm)
  } catch (e) {
    ...
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
```

可以看到，求值时先执行`pushTarget(this)`，通过查阅`src/core/observer/dep.js`，我们可以看到：

``` javascript
Dep.target = null
const targetStack = []

export function pushTarget (target: ?Watcher) {
  targetStack.push(target)
  Dep.target = target
}

export function popTarget () {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}
```

`pushTarget`主要是把`watcher`实例进栈，并赋值给`Dep.target`，而`popTarget`则相反，把`watcher`实例出栈，并将栈顶赋值给`Dep.target`。`Dep.target`这个我们之前在`getter`里见到过，其实就是当前正在求值的观察者。这里在求值前将`Dep.target`设置为`watcher`，使得在求值过程中获取数据时触发`getter`访问器，从而调用`dep.depend`，继而执行`watcher`的`addDep`操作：

``` javascript
addDep (dep: Dep) {
  const id = dep.id
  if (!this.newDepIds.has(id)) {
    this.newDepIds.add(id)
    this.newDeps.push(dep)
    if (!this.depIds.has(id)) {
      dep.addSub(this)
    }
  }
}
```

先判断`newDepIds`是否包含`dep.id`，没有则说明尚未添加过这个`dep`，此时将`dep`和`dep.id`分别加到`newDepIds`和`newDeps`。如果`depIds`不包含`dep.id`，则说明之前未添加过此`dep`，因为是双向添加的(将`dep`添加到`watcher`的同时也需要将`watcher`收集到`dep`)，所以需要调用`dep.addSub`，将当前`watcher`添加到新的`dep`的观察者队列。

``` javascript
if (this.deep) {
  traverse(value)
}
```

再接着根据`this.deep`来调用`traverse`。`traverse`的作用主要是递归遍历触发`value`的`getter`，调用所有元素的`dep.depend()`并过滤重复收集的`dep`。最后调用`popTarget()`将当前`watcher`移出栈，并执行`cleanupDeps`：

``` javascript
cleanupDeps () {
  let i = this.deps.length
  while (i--) {
    const dep = this.deps[i]
    if (!this.newDepIds.has(dep.id)) {
      dep.removeSub(this)
    }
  }
  ...
}
```

遍历`this.deps`，如果在`newDepIds`中不存在`dep.id`，则说明新的依赖里不包含当前`dep`，需要到`dep`的观察者列表里去移除当前这个`watcher`，之后便是`depIds`和`newDepIds`、`deps`和`newDeps`的值交换，并清空`newDepIds`和`newDeps`。到此完成了对`watcher`的求值操作，同时更新了新的依赖，最后返回`value`即可。

回到`createComputedGetter`接着看：

``` javascript
if (Dep.target) {
  watcher.depend()
}
```

当执行计算属性的`getter`时，有可能表达式中还有别的计算属性依赖，此时我们需要执行`watcher.depend`将当前`watcher`的`deps`添加到`Dep.target`即可。最后返回求得的`watcher.value`即可。

总的来说我们从`this[key]`触发`watcher`的`get`函数，将当前`watcher`入栈，通过求值表达式将所需要的依赖`dep`收集到`newDepIds`和`newDeps`，并将`watcher`添加到对应`dep`的观察者列表，最后清除无效`dep`并返回求值结果，这样就完成了依赖关系的收集。

### `Watcher`的更新

以上我们了解了`watcher`的依赖收集和`dep`的观察者收集的基本原理，接下来我们了解下`dep`的数据更新时如何通知`watcher`进行`update`操作。

``` javascript
notify () {
  // stabilize the subscriber list first
  const subs = this.subs.slice()
  for (let i = 0, l = subs.length; i < l; i++) {
    subs[i].update()
  }
}
```

首先在`dep.notify`时，我们将`this.subs`拷贝出来，防止在`watcher`的`get`时候`subs`发生更新，之后调用`update`方法：

``` javascript
update () {
  /* istanbul ignore else */
  if (this.lazy) {
    this.dirty = true
  } else if (this.sync) {
    this.run()
  } else {
    queueWatcher(this)
  }
}
```

* 如果是`lazy`，则将其标记为`this.dirty = true`，使得在`this[key]`的`getter`触发时进行`watcher.evaluate`调用计算。
* 如果是`sync`同步操作，则执行`this.run`，调用`this.get`求值和执行回调函数`cb`。
* 否则执行`queueWatcher`，选择合适的位置，将`watcher`加入到队列去执行即可，因为和响应式数据无关，故不再展开。

## 小结

因为篇幅有限，只对数据绑定的基本原理做了基本的介绍，在这画了一张简单的流程图来帮助理解`Vue`的响应式数据，其中省略了一些`VNode`等不影响理解的逻辑及边界条件，尽可能简化地让流程更加直观：

![](https://user-gold-cdn.xitu.io/2019/5/6/16a8d66f419d837a?w=1052&h=973&f=png&s=123892)

最后，本着学习的心态，在写作的过程中也零零碎碎的查阅了很多资料，其中难免出现纰漏以及未覆盖到的知识点，如有错误，还请不吝指教。