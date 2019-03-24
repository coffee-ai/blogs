# 利用Dectorator分模块存储Vuex状态(下)

## 1、引言

在上篇中我们介绍了利用`Decorator`分模块来存储数据，在`weex`页面实例创建时读取数据到`vuex`，但是存在无效数据存储冗余的问题，并且不支持`registerModule`来动态存取数据，接下来主要介绍下利用`Decorator`来实现数据存储的黑白名单和模块动态注册时数据存取的实现方式。

## 2、 设置`state`黑白名单

在设置`state`的黑白名单时，我们需要对各个模块单独设置，即允许在不同`module`中设置黑或白名单，同时需要在`unregisterModule`移除模块时同时移除黑白名单中的引用，在这里我们使用`WeakMap`和`WeakSet`来存储相关信息，避免`module`注销时无法释放相应的`prop`。

### 2.1、`WeakMap`和`WeakSet`

`WeakMap`是ES6中新增的数据类型，其键属于弱引用，可以很好的避免在没有其他引用时，因`Map`存在引用无法释放而导致的内存泄漏。首先，需要创建黑白名单的修饰器：

```
const WHITE_TAG = 1; // 白名单tag
const BLACK_TAG = 2; // 黑名单tag
// 存储state的信息
const filterMap = new WeakMap();
// 存储state[prop]的getter，用于在setState时过滤无效的prop
const descriptorSet = new WeakSet();
// 接受黑白名单的tag，返回对应的修饰器
const createDescriptorDecorator = tag => (target, prop) => {
    if (!filterMap.has(tag)) {
        // 设置当前state的黑白名单信息
        filterMap.set(target, tag);
    } else if (filterMap.get(target) ^ tag) {
        // 在同一state中，不能同时存在黑名单和白名单
        throw new Error('...');
    }
    let value = target[prop];
    return {
        enumerable: true,
        configurable: true,
        get: function() {
            const {get: getter} = Object.getOwnPropertyDescriptor(target, prop) || {};
            if (getter && !descriptorSet.has(getter)) {
                // 收集依赖到WeakSet
                descriptorSet.add(getter);
            }
            return value;
        },
        set: function(newVal) {
            value = newVal;
        }
    };
};

const shouldWrite = createDescriptorDecorator(WHITE_TAG);
const forbidWrite = createDescriptorDecorator(BLACK_TAG);
```

这样就完成了修饰器的编写，使用时对需要的`state`的`property`修饰即可：

```
const module = {
    state: () => ({
        @shouldWrite
        property1: {},
        property2: [],
    })
};
```

此时当读取`state.property1`时就会将`property1`的`getter`存入到`descriptorSet`中，供过滤时使用。

### 2.2、解析`module`的`state`

在`getter`触发依赖收集后，我们将`state`的`property`存入了`descriptorSet`，现在我们需要根据`module`的`state`，利用`descriptorSet`将无效的数据过滤掉，返回`pureState`:

```
/**
 * [根据黑白名单，过滤state中无效的数据，返回新state]
 * @param  {[type]} module [state对应的module]
 * @param  {[type]} state  [待过滤的state对象]
 * @return {[type]}        [新的state]
 */
const parseModuleState = (module, state) => {
    const {_children, state: moduleState} = module;
    const childrenKeys = Object.keys(_children);
    // 获取state中各个property的getter
    const descriptors = Object.getOwnPropertyDescriptors(moduleState);
    // 判断当前state启用黑名单还是白名单，默认黑名单
    const tag = filterMap.get(moduleState) || BLACK_TAG;
    const isWhiteTag = tag & WHITE_TAG;
    // Object.fromEntries可以用object.fromentries来polyfill
    const pureState = Object.fromEntries(Object.entries(state).filter(([stateKey]) => {
        const {get: getter} = descriptors[stateKey] || {};
        // 过滤子模块的数据，只保留白名单或不在黑名单里的数据
        return !childrenKeys.some(childKey => childKey === stateKey)
                && !((isWhiteTag ^ descriptorSet.has(getter)));
    }));
    return pureState;
}
```
这样剔除了冗余的数据，只保留有效数据存入`storage`。

## 3、`registerModule`

`vuex`支持`registerModule`和`unregisterModule`来动态的注册模块和注销模块，