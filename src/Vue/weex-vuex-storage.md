利用Dectorator分模块存储Vuex状态

1、引言
在H5的Vue项目中，最为常见的当为单页应用(SPA)，利用Vue-Router控制组件的挂载与复用，这时使用Vuex可以方便的维护数据状态而不必关心组件间的数据通信。但在Weex中，不同的页面之间使用不同的执行环境，无法共享数据，此时多为通过BroadcastChannel或storage模块来实现数据通信，本文主要使用修饰器(Decorator)来扩展Vuex的功能，实现分模块存储数据，并降低与业务代码的耦合度。

2、Decorator
设计模式中有一种装饰器模式，可以在运行时扩展对象的功能，而无需创建多个继承对象。类似的，Decorator可以在编译时扩展一个对象的功能，降低代码耦合度的同时实现多继承一样的效果。

2.1、Decorator安装
目前Decorator还只是一个提案，在生产环境中无法直接使用，可以用babel-plugin-transform-decorators-legacy来实现。使用npm管理依赖包的可以执行以下命令：

npm install babel-plugin-transform-decorators-legacy -D

然后在 .babelrc 中配置

{
    "plugins": [
        "transform-decorators-legacy"
    ]
}

或者在webpack.config.js中配置

{
    test: /\.js$/,
    loader: "babel-loader",
    options: [
        plugins: [
            require("babel-plugin-transform-decorators-legacy").default
        ]
    ]
}

这时可以在代码里编写Decorator函数了。

2.2、Decorator的编写
在本文中，Decorator主要是对方法进行修饰，主要代码如下：

decorator.js

const actionDecorator = (target, name, descriptor) => {
    const fn = descriptor.value;
    descriptor.value = function(...args) {
        console.log('调用了修饰器的方法');
        return fn.apply(this, args);
    };
    return descriptor;
};

store.js

const module = {
    state: () => ({}),
    actions: {
        @actionDecorator
        someAction() {/** 业务代码 **/ },
    },
};

可以看到，actionDecorator修饰器的三个入参和Object.defineProperty一样，通过对module.actions.someAction函数的修饰，实现在编译时重写someAction方法，在调用方法时，会先执行console.log('调用了修饰器的方法');，而后再调用方法里的业务代码。对于多个功能的实现，比如存储数据，发送广播，打印日志和数据埋点，增加多个Decorator即可。

3、Vuex
Vuex本身可以用subscribe和subscribeAction订阅相应的mutation和action，但只支持同步执行，而Weex的storage存储是异步操作，因此需要对Vuex的现有方法进行扩展，以满足相应的需求。

3.1、修饰action
在Vuex里，可以通过commit mutation或者dispatch action来更改state，而action本质是调用commit mutation。因为storage包含异步操作，在不破坏Vuex代码规范的前提下，我们选择修饰action来扩展功能。

storage使用回调函数来读写item，首先我们将其封装成Promise结构：

storage.js

const storage = weex.requireModule('storage');
const handler = {
  get: function(target, prop) {
    const fn = target[prop];
    // 这里只需要用到这两个方法
    if ([
      'getItem',
      'setItem'
    ].some(method => method === prop)) {
      return function(...args) {
        // 去掉回调函数，返回promise
        const [callback] = args.slice(-1);
        const innerArgs = typeof callback === 'function' ? args.slice(0, -1) : args;
        return new Promise((resolve, reject) => {
          fn.call(target, ...innerArgs, ({result, data}) => {
            if (result === 'success') {
              return resolve(data);
            }
            // 防止module无保存state而出现报错
            return resolve(result);
          })
        })
      }
    }
    return fn;
  },
};
export default new Proxy(storage, handler);

通过Proxy，将setItem和getItem封装为promise对象，后续使用时可以避免过多的回调结构。

现在我们把storage的setItem方法写入到修饰器：

decorator.js

import storage from './storage';
// 存放commit和module键值对的WeakMap对象
import {moduleWeakMap} from './plugin'; 
const setState = (target, name, descriptor) => {
    const fn = descriptor.value;
    descriptor.value = function(...args) {
        const [{state, commit}] = args;
        // action为异步操作，返回promise，
        // 且需在状态修改为fulfilled时再将state存储到storage
        return fn.apply(this, args).then(async data => {
            const {module, moduleKey} = moduleWeakMap.get(commit) || {};
            if (module) {
                const {_children} = module;
                const childrenKeys = Object.keys(_children);
                // 只获取当前module的state，childModule的state交由其存储，按module存储数据，避免存储数据过大
                // Object.fromEntries可使用object.fromentries来polyfill，或可用reduce替代
                const pureState = Object.fromEntries(Object.entries(state).filter(([stateKey]) => {
                    return !childrenKeys.some(childKey => childKey === stateKey);
                }));
                await storage.setItem(moduleKey, JSON.stringify(pureState));
            }
            // 将data沿着promise链向后传递
            return data;
        });
    };
    return descriptor;
};
export default setState;

完成了setState修饰器功能以后，就可以装饰action方法了，这样等action返回的promise状态修改为fulfilled后调用storage的存储功能，及时保存数据状态以便在新开Weex页面加载最新数据。

store.js

import setState from './decorator';
const module = {
    state: () => ({}),
    actions: {
        @setState
        someAction() {/** 业务代码 **/ },
    },
};

3.2、读取module数据
完成了存储数据到storage以后，我们还需要在新开的Weex页面实例能自动读取数据并初始化Vuex的状态。在这里，我们使用Vuex的plugins设置来完成这个功能。

首先我们先编写Vuex的plugin：

plugin.js

import storage from './storage';
// 加个rootKey，防止rootState的namespace为''而导致报错
// 可自行替换为其他字符串
import {rootKey} from './constant';
const parseJSON = (str) => {
    try {
        return str ? JSON.parse(str) : undefined;
    } catch(e) {}
    return undefined;
};
export const moduleWeakMap = new WeakMap();
const getState = (store) => {
    const getStateData = async function getModuleState(module, path = []) {
        const moduleKey = `${path.join('/')}/`;
        const {_children, context} = module;
        const {commit} = context || {};
        // 初始化store时将commit和module存入WeakMap，以便setState时快速查找对应module
        moduleWeakMap.set(commit, {module, moduleKey});
        // 根据path读取当前module下存储在storage里的数据
        const data = parseJSON(await storage.getItem(moduleKey)) || {};
        const children = Object.entries(_children);
        if (!children.length) {
            return data;
        }
        // 剔除childModule的数据，递归读取
        const childModules = await Promise.all(
            children.map(async ([childKey, child]) => {
              return [childKey, await getModuleState(child, path.concat(childKey))];
            })
        );
        return {
            ...data,
            ...Object.fromEntries(childModules),
        }
    };
    // 读取本地数据，merge到Vuex的state
    const init = getStateData(store._modules.root, [rootKey]).then(savedState => {
        store.replaceState(merge(store.state, savedState, {
            arrayMerge: function (store, saved) { return saved },
            clone: false,
        }));
    });
};
export default getState;

以上就完成了Vuex的数据按照module读取，但Weex的IOS/Andriod中的storage存储是异步的，为防止组件挂载以后发送请求返回的数据被本地数据覆盖，需要在本地数据读取并merge到state以后再调用new Vue，这里我们使用一个简易的interceptor来拦截：

interceptor.js

const interceptors = {};
export const registerInterceptor = (type, fn) => {
    const interceptor = interceptors[type] || (interceptors[type] = []);
    interceptor.push(fn);
};
export const runInterceptor = async (type) => {
    const task = interceptors[type] || [];
    return Promise.all(task);
};

这样plugin.js中的getState就修改为：

import {registerInterceptor} from './interceptor';
const getState = (store) => {
    /** other code **/
    const init = getStateData(store._modules.root, []).then(savedState => {
        store.replaceState(merge(store.state, savedState, {
            arrayMerge: function (store, saved) { return saved },
            clone: false,
        }));
    });
    // 将promise放入拦截器
    registerInterceptor('start', init);
};

store.js

import getState from './plugin';
import setState from './decorator';
const rootModule = {
    state: {},
    actions: {
        @setState
        someAction() {/** 业务代码 **/ },
    },
    plugins: [getState],
    modules: {
        /** children module**/
    }
};

app.js

import {runInterceptor} from './interceptor';
// 待拦截器内所有promise返回resolved后再实例化Vue根组件
// 也可以用Vue-Router的全局守卫来完成
runInterceptor('start').then(() => {
   new Vue({/** other code **/});
});

这样就实现了Weex页面实例化后，先读取storage数据到Vuex的state，再实例化各个Vue的组件，更新各自的module状态。

4、TODO
通过Decorator实现了Vuex的数据分模块存储到storage，并在Store实例化时通过plugin分模块读取数据再merge到state，提高数据存储效率的同时实现与业务逻辑代码的解耦。但还存在一些可优化的点：

1、触发action将会存储所有module中的state，只需保存必要状态，避免无用数据。
2、对于通过registerModule注册的module，需支持自动读取本地数据。
在此不再展开，将在后续版本中实现。
