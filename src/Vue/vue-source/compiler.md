# 浅析Vue编译原理

在上一篇里，我们主要聊了下[Vue数据绑定简析](https://juejin.im/post/5ca614eff265da3072620189)，明白了其观察者模式的基本原理。我们知道在观察者中有一种属于**渲染函数观察者**(`vm._watcher`)，通过对渲染函数的求值计算来触发依赖收集，进而进行响应式的数据绑定，但是对于渲染函数如何编译，我们知之甚少。 这一篇我们将从`template`编译`AST`语法树， 再`generate`转化为`render`这一过程来看看`Vue`的编译原理。

## `compilerOption`

在我们编写的`Vue`组件中，要渲染一条文本的`UI`可能会是如下写法：

```
<template>
    <p class="foo">{{text}}</p>
</template>
```

编译后变成：

```javascript
var render = function() {
    var _vm = this
    _vm._c("p", {
        staticClass: "foo"
    }, [
        _vm._v(_vm._s(_vm.text))
    ])
}
```

经过编译，将一段`template`的`html`代码变成一个`js`函数，通过对该函数的调用来收集到依赖`_vm.text`，实现数据变化时实时更新
`UI`。那么它是如何识别`class`这些`attr`的，又是如何编译成一个完整的渲染函数的呢？

为了探究`Compiler`的原理，我们找到完整版的入口文件`entry-runtime-with-compiler.js`，发现其重写了`$mount`函数：如果没有`render`函数，则转换`template`为字符串，再调用`compileToFunctions`进行编译，最终将`render`和`staticRenderFns`挂到`$options`下并调用原始的`$mount`。通过对`compileToFunctions`的层层剥离，我们可以得到最终的结果：

```javascript
const { compile, compileToFunctions } = createCompilerCreator(function baseCompile() {
  ...
})(baseOptions)
```

可以看到，其利用闭包将`baseCompile`函数和`baseOptions`对象依次传入`createCompilerCreator`函数并保存，相当于对工厂函数进行柯理化以提高模块解耦下的代码复用性，最终返回`compile`和`compileToFunctions`两个编译函数。

`baseOptions`属于编译时的配置项，与具体的目标平台有关，`web`平台下的代码如下：

```javascript
// src/platforms/web/compiler/options.js
export const baseOptions: CompilerOptions = {
  expectHTML: true,
  modules,
  directives,
  isPreTag,
  isUnaryTag,
  mustUseProp,
  canBeLeftOpenTag,
  isReservedTag,
  getTagNamespace,
  staticKeys: genStaticKeys(modules)
}
```

* `modules`：主要包含了一些编译时做的转换和前置转换的操作，包括对`class`和`style`做一些`staticAttr`和`bindingAttr`相关的处理，对`<input />`的`bindingType`做了动态匹配`type`的处理等。
* `directives`：包含了`v-html`、`v-text`及`v-model`的指令编译。
* `staticKeys`：将`modules`里的`staticKeys`拼接成字符串，这里是`staticClass,staticStyle`。
* 剩下的主要是一些对`tag`的判断函数，待用到时再来具体分析。

了解了`baseOptions`，我们再来看看`baseCompile`：

```javascript
// src/compiler/index.js
export const createCompiler = createCompilerCreator(function baseCompile (
  template: string,
  options: CompilerOptions
): CompiledResult {
  const ast = parse(template.trim(), options)
  if (options.optimize !== false) {
    optimize(ast, options)
  }
  const code = generate(ast, options)
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
})
```

* `parse`：根据字符串`template`和`options`生成`AST`语法树。
* `optimize`：用来给`root`递归添加各种`static`属性，配合`VNode`和`patch`使用，避免不必要的`re-render`，提高性能。
* `generate`：根据`ast`语法树，生成`String`类型的`render`和`Array<String>`类型的`staticRenderFns`，供后续的`new Function`构造生成最终的`render`和`staticRenderFns`。

以上只是对`compiler`有个粗略的了解，明白了其编译的主要工作流程，现在我们需要对整个`compile`过程进行代码阅读，加深对其编译原理的理解。首先我们回到`createCompilerCreator`，为了避免无关代码干扰阅读，这里去掉了一些和开发调试相关的代码。我们先来看下最终的返回结果：

```javascript
// src/compiler/create-compiler.js
export function createCompilerCreator (baseCompile: Function): Function {
  return function createCompiler (baseOptions: CompilerOptions) {
    function compile (
      template: string,
      options?: CompilerOptions
    ): CompiledResult {
      ...
    }

    return {
      compile,
      compileToFunctions: createCompileToFunctionFn(compile)
    }
  }
}
```

`compileToFunctions`是通过调用`createCompileToFunctionFn(compile)`返回，`createCompileToFunctionFn`函数只是负责将`String`类型的`render`通过调用`new Function(render)`转换成`Function`类型的`render`，并将结果存到`cache`对象，之后再编译相同的`key`时直接返回本地缓存，提高编译效率，具体代码可见`src/compiler/to-function.js`，这里就不再赘述。再来看具体的`compile`函数：

```javascript
function compile (
  template: string,
  options?: CompilerOptions
): CompiledResult {
  const finalOptions = Object.create(baseOptions)
  const errors = []
  const tips = []

  let warn = (msg, range, tip) => {
    (tip ? tips : errors).push(msg)
  }

  if (options) {
    ...
    // merge custom modules
    if (options.modules) {
      finalOptions.modules =
        (baseOptions.modules || []).concat(options.modules)
    }
    // merge custom directives
    if (options.directives) {
      finalOptions.directives = extend(
        Object.create(baseOptions.directives || null),
        options.directives
      )
    }
    // copy other options
    for (const key in options) {
      if (key !== 'modules' && key !== 'directives') {
        finalOptions[key] = options[key]
      }
    }
  }

  finalOptions.warn = warn

  const compiled = baseCompile(template.trim(), finalOptions)
  if (process.env.NODE_ENV !== 'production') {
    detectErrors(compiled.ast, warn)
  }
  compiled.errors = errors
  compiled.tips = tips
  return compiled
}
```

显而易见，这里主要通过继承、拷贝等生成`finalOptions`，并执行`baseCompile`方法。我们以`web`下的编译为例，在`$mount`方法中编译传入的`options`如下：

```javascript
// src/platforms/web/entry-runtime-with-compiler.js
{
  outputSourceRange: process.env.NODE_ENV !== 'production', // 开发模式下标记start和end，便于定位编译出错信息
  shouldDecodeNewlines, // 是否对换行符进行转码
  shouldDecodeNewlinesForHref, // 是否对href内的换行符进行转码
  delimiters: options.delimiters, // 改变纯文本插入分隔符，用来自定义<p>{{a}}</p>中的'{{}}'
  comments: options.comments // 是否保留模板HTML中的注释，默认false
}
```

这里我们以一张流程图来简要梳理下编译过程：

![](https://user-gold-cdn.xitu.io/2019/5/25/16aee5e3b67be5a0?w=1209&h=1000&f=png&s=101919)

## `parse`

了解了编译参数的生成过程后，我们再来看`parse`的工作原理。现在回到之前的`baseCompile`，其中`template`在经过`$mount`函数转换后，统一转为字符串：

```javascript
const ast = parse(template.trim(), options)
```

进入`parse`方法体，首先是：

```javascript
// src/compiler/parser/index.js
transforms = pluckModuleFunction(options.modules, 'transformNode')
preTransforms = pluckModuleFunction(options.modules, 'preTransformNode')
postTransforms = pluckModuleFunction(options.modules, 'postTransformNode')
```

`pluckModuleFunction`方法用以将`modules`数组中的各元素`target`对应的`target[key]`组成新的数组，用以后续进行遍历操作，详见`src/compiler/helper.js`，在后续调用时再做具体说明。

而后声明一个变量`root`，并调用`parseHTML`(传入`template`和`option`，包括`start`、`end`、`chars`和`comment`方法)，最终返回`root`作为`AST`语法树。

### `parseHTML`

`parseHTML`，如其命名，用于解析我们的`HTML`字符串。首先是初始化`stack`，用以维护非自闭合的`element`栈，之后利用`while(html)`对`html`进行标签正则匹配、循环解析，这里我们用一张导图来更加直观地展示其解析逻辑：

![](https://user-gold-cdn.xitu.io/2019/5/26/16af44ef4c97f6ef?w=1247&h=1058&f=png&s=148934)

通过调用`advance`来截断更新`html`字符串，供下一次循环解析，并调用`option`中的`start`、`end`、`chars`和`comment`方法将其添加到`root`对象中，最终完成`AST`语法树的输出。在`parseHTML`中，最核心的当属解析标签的开始，因此我们以`parseStartTag`为入口，来进行代码阅读。

```javascript
const start = html.match(startTagOpen)
if (start) {
  const match = {
    tagName: start[1],
    attrs: [],
    start: index
  }
  // 正则匹配标签，并截断正在解析的html字符串
  advance(start[0].length)
  let end, attr
  while (!(end = html.match(startTagClose)) && (attr = html.match(dynamicArgAttribute) || html.match(attribute))) {
    // 正则获取标签上的attrs，并存入match的attrs数组
    attr.start = index
    advance(attr[0].length)
    attr.end = index
    match.attrs.push(attr)
  }
  // 解析到'>'字符
  if (end) {
    match.unarySlash = end[1] // '/>'时为自闭合标签，否则为false
    advance(end[0].length)
    match.end = index
    return match
  }
}
```

以如下`template`为例：

```html
<my-component :foo.sync="foo" :key="index" @barTap="barTap" v-for="(item, index) in barList">
  <input v-model.number="value" />
  <template #bodySlot="{link, text}">
    <a v-if="link" :href="link">链接地址为：{{text | textFilter | linkFilter}}！</a>
    <p v-else>{{text | filter}}</p>
  </template>
</my-component>
```

可以得到如下`match`：

```javascript
{ 
  tagName: 'my-component',
  attrs: [ 
    [ ' :foo.sync="foo"',
       ':foo.sync',
       '=',
       'foo',
       undefined,
       undefined,
       index: 0,
       input: ' :foo.sync="foo" :key="index" @barTap="barTap" v-for="(item, index) in barList">'],
    [ ' :key="index"',
       ':key',
       '=',
       'index',
       undefined,
       undefined,
       index: 0,
       input: ' :key="index" @barTap="barTap" v-for="(item, index) in barList">'],
    [ ' @barTap="barTap"',
       '@barTap',
       '=',
       'barTap',
       undefined,
       undefined,
       index: 0,
       input: ' @barTap="barTap" v-for="(item, index) in barList">'],
    [ ' v-for="(item, index) in barList"',
       'v-for',
       '=',
       '(item, index) in barList',
       undefined,
       undefined,
       index: 0,
       input: ' v-for="(item, index) in barList">']
  ],
}
```

获取到`match`后，接着调用`handleStartTag`方法：

* 判断`if (expectHTML)`，主要是对于编写自闭合标签时做自动纠错的功能，确保`lastTag`的正确解析。
* `const unary = isUnaryTag(tagName) || !!unarySlash`来获取当前标签的闭合属性，其中`isUnaryTag`为当前编译的`web`平台下自闭合的标签，可见于`src/platforms/web/compiler/util.js`。
* 解析`match`到的`attrs`数组，其中需要对属性的值做反转义处理，确保值能正确运算执行。
* 判断`if (!unary)`，如果非自闭合，则将当前标签的所有信息存入`stack`栈里，同时将`lastTag`替换为当前`tagName`，确保后续`parseEndTag`的正常解析。
* 最后通过传入的`options.start`方法，调用`options.start(tagName, attrs, unary, match.start, match.end)`。

回到`parseHTML`时的`start`方法，首先创建了一个`ASTElement`：

```
let element: ASTElement = createASTElement(tag, attrs, currentParent)
```

得到`element`如下：

```javascript
// 去掉了与调试相关的start和end属性
{
  type: 1,
  tag: 'my-component',
  attrsList: [
    {name: ':foo.sync', value: 'foo'},
    {name: ':key', value: 'index'},
    {name: '@barTap', value: 'barTap'},
    {name: 'v-for', value: '(item, index) in barList'},
  ],
  attrsMap: {
    ':foo.sync': 'foo',
    ':key': 'index',
    '@barTap': 'barTap',
    'v-for': '(item, index) in barList',
  },
  rawAttrsMap: {},
  parent, // parent为其父element
  children: []
}
```

接着是

```javascript
for (let i = 0; i < preTransforms.length; i++) {
  element = preTransforms[i](element, options) || element
}
```

在`parseHTML`开头提到`preTransforms = pluckModuleFunction(options.modules, 'preTransformNode')`(用以执行`options.modules`数组中每个元素的`preTransformNode`方法)，通过查看编译时的`finalOptions`，我们得到`options.modules`为`baseOptions.modules`(详见`src/platforms/web/compiler/options.js`)。在这里主要是执行`input`标签里`v-model`指令的前置转换，具体等解析`input`标签时我们再来看。接着是一连串的判断：

* `if (!inVPre)`，用来判断其某个`parent`是否包含`v-pre`指令，我们知道包含`v-pre`指令，则跳过这个元素和它的子元素的编译过程。如果没有，则调用`processPre`来判断当前元素是否包含`v-pre`，若是则设置`inVPre`。
* `platformIsPreTag(element.tag)`判断`element`是否为`pre`标签，若是则设置`inPre`。
* 根据`inVPre`执行`process`编译，若`inVPre`为`true`，则将`el.attrsList`的各元素`value`调用`JSON.stringify`为字符串。
* 若`inVPre`为`false`，则根据`element.processed`判断当前`element`是否已编译完成，依次调用`processFor(element)`、`processIf(element)`、`processOnce(element)`。

#### `processFor`

在利用`processFor`解析`v-for`指令时：

* 首先是获取`v-for`的`attr`值，`exp = getAndRemoveAttr(el, 'v-for')`，根据上面得到的`element`和`getAndRemoveAttr`方法，我们可以得到`exp`为`'(item, index) in barList'`，同时在`element`的`attrsList`移除当前元素
* 之后调用`const res = parseFor(exp)`解析`for`语句：

```javascript
export function parseFor (exp: string): ?ForParseResult {
  const inMatch = exp.match(forAliasRE)
  // 简化后 inMatch = ['(item, index) in barList', '(item, index)', 'barList']
  if (!inMatch) return
  const res = {}
  res.for = inMatch[2].trim()
  const alias = inMatch[1].trim().replace(stripParensRE, '') // 取括号内的alias如：item, index
  const iteratorMatch = alias.match(forIteratorRE)
  if (iteratorMatch) {
    res.alias = alias.replace(forIteratorRE, '').trim()
    // iterator1和iterator2，依次取for的后两项迭代属性
    res.iterator1 = iteratorMatch[1].trim()
    if (iteratorMatch[2]) {
      res.iterator2 = iteratorMatch[2].trim()
    }
  } else {
    res.alias = alias
  }
  return res
}
```

在我们的例子中，可以得到`res`为`{for: 'barList', alias: 'item', iterator1: 'index'}`

* 得到`res`后，我们调用`extend(el, res)`将其合并到`element`里，得到此时的`element`为：

```javascript
{
  type: 1,
  tag: 'my-component',
  attrsList: [
    {name: ':foo.sync', value: 'foo'},
    {name: '@barTap', value: 'barTap'},
  ],
  attrsMap: {
    ':foo.sync': 'foo',
    '@barTap': 'barTap',
    'v-for': '(item, index) in barList',
  },
  rawAttrsMap: {},
  parent, // parent为其父element
  children: [],
  key: 'index',
  for: 'barList',
  alias: 'item',
  iterator1: 'index'
}
```

接着是`processIf`和`processOnce`，`processOnce`用来给`element`添加`once`标志，`processIf`这里未涉及到，将在解析`a`标签那里去展开。

之后便是判断是否为根节点，如果之前未解析过标签，则将当前`element`赋值给`root`，同时，由于`<my-component></my-component>`非自闭合标签，因此暂存至`currentParent`变量，同时将其推入`stack`栈，至此，我们简单的解析完了一个标签的开始部分。

#### `preTransformNode`

当前的`html`字符串变成了：

```html
  <input v-model.number="value" />
  <template #bodySlot="{link, text}">
    <a v-if="link" :href="link">链接地址为：{{text | textFilter | linkFilter}}！</a>
    <p v-else>{{text | filter}}</p>
  </template>
</my-component>
```

接着我们重复`parseHTML`的`while`方法，此时解析到的是一个`input`的自闭合标签，因此不需要进入`stack`，其他与上一个标签类似，这里主要来看下之前提到的`preTransforms`：

```javascript
// src/platforms/web/compiler/modules/model.js
function preTransformNode (el: ASTElement, options: CompilerOptions) {
  if (el.tag === 'input') {
    const map = el.attrsMap
    if (!map['v-model']) {
      return
    }
    let typeBinding
    if (map[':type'] || map['v-bind:type']) {
      typeBinding = getBindingAttr(el, 'type')
    }
    if (!map.type && !typeBinding && map['v-bind']) {
      // 取v-bind的对象中type值
      typeBinding = `(${map['v-bind']}).type`
    }
    if (typeBinding) {
      // 判断是否有v-if指令
      const ifCondition = getAndRemoveAttr(el, 'v-if', true)
      const ifConditionExtra = ifCondition ? `&&(${ifCondition})` : ``
      const hasElse = getAndRemoveAttr(el, 'v-else', true) != null
      const elseIfCondition = getAndRemoveAttr(el, 'v-else-if', true)
      // 1. checkbox
      const branch0 = cloneASTElement(el)
      // process for on the main node
      processFor(branch0)
      addRawAttr(branch0, 'type', 'checkbox')
      processElement(branch0, options)
      branch0.processed = true // prevent it from double-processed
      branch0.if = `(${typeBinding})==='checkbox'` + ifConditionExtra
      addIfCondition(branch0, {
        exp: branch0.if,
        block: branch0
      })
      // 2. add radio else-if condition
      const branch1 = cloneASTElement(el)
      getAndRemoveAttr(branch1, 'v-for', true)
      addRawAttr(branch1, 'type', 'radio')
      processElement(branch1, options)
      addIfCondition(branch0, {
        exp: `(${typeBinding})==='radio'` + ifConditionExtra,
        block: branch1
      })
      // 3. other
      const branch2 = cloneASTElement(el)
      getAndRemoveAttr(branch2, 'v-for', true)
      addRawAttr(branch2, ':type', typeBinding)
      processElement(branch2, options)
      addIfCondition(branch0, {
        exp: ifCondition,
        block: branch2
      })

      if (hasElse) {
        branch0.else = true
      } else if (elseIfCondition) {
        branch0.elseif = elseIfCondition
      }

      return branch0
    }
  }
}
```

* 判断`ASTElement`的`tag`及`attrsMap`，当包含`v-model`的`input`标签时才继续执行，并根据多种绑定方式，获取`typeBinding`。
* 生成`checkbox`情况下的`ASTElement`代码块，并添加到`branch0`。
* 生成`radio`下的`ASTElement`代码块，并添加到`branch1`。
* 其他`type`情况，保留`:type`，并添加到`branch2`。

在`preTransformNode`方法里，通过`cloneASTElement`克隆各种`type`的`branch`，用`processFor`来解析`branch0`根节点的`v-for`指令，并调用`getAndRemoveAttr`删除其余`branch`的`v-for`属性和`addRawAttr`添加`type`属性。再调用`processElement`来编译解析各个`branch0`，并添加到`ifConditions`数组，最后根据`v-if`、`v-else`和`v-else-if`来设置`branch0`对应的属性，并返回`branch0`。

`addIfCondition`中入参对象有两个属性：`exp`和`block`，为表达式和表达式成立时返回的`ASTElement`，具体在后续`generate`中执行`genIf`时再做展开。

`input`是自闭合标签，因而不需要放入`stack`，直接执行`closeElement`。在`preTransformNode`已设置`element.processed`为`true`，只需将`my-component`和`input`设置`children`和`parent`互相引用即可。最后执行`postTransforms`，完成当前`element`的编译。

接着解析`template`标签`<template #bodySlot="{link, text}">`，将对应的`attr`放入`attrsList`和`attrsMap`，过程与上述类似，在此不再赘述，可以得到：

```javascript
{
  type: 1,
  tag: 'template',
  attrsList: [
    {name: '#bodySlot', value: '{link, text}'},
  ],
  attrsMap: {
    '#bodySlot': '{link, text}',
  },
  rawAttrsMap: {},
  parent, // parent为其父my-component
  children: [],
}
```

#### `parseText`

接着是一组`a`标签和`p`标签，首先是一组`v-if`和`v-else`：

```html
<a v-if="link" :href="link">链接地址为：{{text | textFilter | linkFilter}}！</a>
<p v-else>{{text | filter}}</p>
```

可以看到，`processIf`方法里通过对`v-if`的判断来调用`addIfCondition`，而`v-else-if`没有调用，是因为在`closeElement`里会通过`processIfConditions`向前查找最近一个`v-if`表达式的`element`并对其调用`addIfCondition`方法。之后通过传入的`option.char`方法来解析字符串。在方法里，我们可以看到根据`(res = parseText(text, delimiters))`用来生成不同的`child`，下面我们来看下`parseText`是如何编译的。

首先会根据`compilerOption`里传入的`delimiters`来确定`text`的匹配规则，默认为`{{}}`，接着通过`while`循环匹配`"链接地址为：{{text | textFilter | linkFilter}}！"`字符串，并将结果依次存入`rawTokens`和`tokens`，最终返回`expression`和`tokens`。

>增加`index > lastIndex`及循环结束时的`lastIndex < text.length`，是为了将未包含在`delimiters`之内的字符串也一起解析保存。

在`while`循环体内，还有一个`parseFilter`用来解析我们的过滤器，利用`for`循环去识别各个`i`位置的`char`

```!
这里增加对 ` ' " / 等的判断，是为了避免把字符串和正则里的 | 当做过滤器操作符
```

首先是取出需要操作的表达式`expression`，之后将匹配到的过滤器依次存入`filters`数组，最后调用`wrapFilter`方法，根据过滤器有无其他参数， 返回真正的表达式。据此我们可以得到最终的`expression`为`_f("linkFilter")(_f("textFilter")(text))`，**可以看到当同一表达式存在多个过滤器时，其将按照从左往右的顺序执行**。`_f`是挂载在`Vue`原型链上的方法，用以运行时根据`vm.$options`来获取到对应的过滤器函数，后续类似的`_s, _c`等方法，可从`src/core/instance/render-helpers/index.js`进行查看，具体不做展开了。

在`parseText`方法最后，我们可以得到：

```javascript
{
  expression: '"链接地址为：" + _s(_f("linkFilter")(_f("textFilter")(text))) + "！"'
  tokens: ["链接地址为：", {@binding: '_f("linkFilter")(_f("textFilter")(text))'}, "！"]
}
```

### `closeElement`

之后解析`</template>`的闭合标签：

```html
  </template>
</my-component>
```

我们先取到之前的`element`：

```javascript
{
  type: 1,
  tag: 'template',
  attrsList: [
    {name: '#bodySlot', value: '{link, text}'},
  ],
  attrsMap: {
    '#bodySlot': '{link, text}',
  },
  parent, // parent为其父my-component
  children: [], 
}
```

之后先执行`processElement`方法，`processElement`函数在`src/compiler/parser/index.js`，主要负责`element`本身的编译解析。

#### `processSlotContent`

>调用核心方法`processSlotContent`用来编译`slot`。

* 先是兼容`template`的`scope`属性，并解析`slot-scope`至`el.slotScope`，用来保存作用域插槽需要传递的`props`。
* 解析`slot`用来处理默认插槽和具名插槽，插槽名设置到`el.slotTarget`，`el.slotTargetDynamic`用以判断是否为动态解析的插槽，支持`:slot="dynamicSlot"`这种动态写法。
* `v-slot`在`tag`为`template`的标签上时，等效于将`slot`和`slot-scope`作合并处理。
* `v-slot`在`component`上时，此时的`slotScope`为`component`自身的插槽上所提供。[the scope variable provided by a component is also declared on that component itself.](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0001-new-slot-syntax.md)
 
  - `getAndRemoveAttrByRegex(el, slotRE)`根据正则取到`slotBinding`，并将其从`attrsList`里移除。
  - `getSlotName(slotBinding)`得到`{name: "bodySlot", dynamic: false}`。
  - `slotContainer = el.scopedSlots.bodySlot`调用`createASTElement('template', [], el)`增加一层新的`ASTElement`，用以传递`slotScope`至`slotContainer`，使得与`template`解析方式保持一致。
  - 将`el.children`过滤带有`slotScope`的`element`，并添加至`slotContainer.children`，因为重新梳理了`el.children`中`element`的父子关系，故而需要移除`el.children`。

>如果`component`的`el.children`中有`element`存在`slotScope`，如下示例，此时在`my-component`和`template`上均有`slotScope`，然而事实上这些域变量均来自于`my-component`，因而为避免`scope ambiguity`，在开发模式下会给出错误提示！

```html
<my-component v-slot="{msg}">
  <template v-slot="{foo}"></template>
</my-component>
```

至此完成了`processSlotContent`对当前`element`及其`children`的插槽语法编译。

#### `processAttrs`

```javascript
for (let i = 0; i < transforms.length; i++) {
  element = transforms[i](element, options) || element
}
```

继续执行`processElement`，便是对`element`依次调用`transforms`，通过查看对应文件`src/platforms/web/compiler/modules/index.js`，该方法主要处理`class`和`style`的静态属性和动态绑定，并设置到`el`的`staticClass`、`classBinding`、`staticStyle`、`styleBinding`四个属性。

最后通过`processAttrs`方法来处理`attrs`各个属性：

`modifiers = parseModifiers(name.replace(dirRE, ''))`将各个`attr`的修饰符依次存入对应的`modifiers`，并设置为`true`。如`:foo.sync`返回`{sync: true}`。

```javascript
syncGen = genAssignmentCode(value, `$event`)
```

返回一个`foo=$event`字符串，当然，在`v-bind`时也支持`foo.bar`、`foo['bar'].tab`等这种，再此就不做展开。

之后调用`addHandler`方法，根据有无`.native`修饰符将`update:foo`方法添加到`el.events`或`el.nativeEvents`，在这里可以得到：

```javascript
{
  events: {
    'update:foo': {
      value: 'foo=$event',
      dynamic: undefined,
    }
  }
  plain: false
}
```

根据是否为`.prop`修饰符或非动态`component`、`DOM`保留的`Property`，分别调用`addProp`添加到`el.props`和`addAttr`添加至`el.attrs`、`el.dynamicAttrs`。最后将未匹配上的`v-`指令调用`addDirective`添加到`el.directives`，至此完成了一个完整`element`包括其子`element`的编译工作。

```!
这里用一张导图来梳理一下parse过程中的主要知识点
```

![](https://user-gold-cdn.xitu.io/2019/9/15/16d32cdc64c4fc34?w=2425&h=1902&f=png&s=333446)

通过`parse`我们就能根据一段字符串`html`解析得到一个`root`节点及其`children`和子孙后代，生成一棵完整的`AST`语法树，最终得到如下结构的`root`：

```javascript
{
  alias: 'item',
  attrs: [{name: 'foo', value: 'foo', dynamic: false}],
  attrsList: [{name: ':foo.sync', value: 'foo'}, {name: '@barTap', value: 'barTap'}],
  attrsMap: {
    ':foo.sync': 'foo',
    '@barTap': 'barTap',
    'v-for': '(item, index) in barList',
  },
  children: [{
    attrsList: [{name: 'v-model.number', value: 'value'}],
    attrsMap: {'v-model.number': 'value'},
    directives: [{
      arg: null,
      isDynamicArg: false,
      modifiers: {number: true},
      name: 'model',
      rawName: "v-model.number",
      value: "value",
    }],
    plain: false,
    prop: [{name: 'value', value: '(value)', dynamic: undefined}],
    tag: 'input',
    type: 1,
  }],
  events: {
    barTap: {value: 'barTap', dynamic: false},
    'update:foo': {value: 'foo=$event', dynamic: undefined},
  },
  for: 'barList',
  forProcessed: true,
  hasBindings: true,
  iterator1: 'index',
  key: 'index',
  plain: false,
  scopedSlots: {
    'bodySlot': {
      ...,
      children: [{
        ...,
        if: 'link',
        // aElement和pElement分别为<a>和<p>创建的两个ASTElement
        ifConditions: [{exp: 'link', block: aElement}, {exp: undefined, block: pElement}],
        ifProcessed: true,
      }],
      slotScope: '{link, text}',
      slotTarget: '"bodySlot"',
      slotTargetDynamic: false,
      tag: "template",
      type: 1,
      }
    }
  },
  tag: 'my-component',
  type: 1,
}
```

## `generate`

在通过通过`parse`方法，我们成功将一段字符串形式的`html`转化为一棵包含`parent`和`children`引用关系的`AST`语法树，接下来我们就需要根据这个`AST`重新生成所需的`render`和`staticRenderFns`渲染函数，而这个转化过程就是通过调用`generate`方法来实现。查阅`src/compiler/index.js`，我们发现其在`generate`之前，还有`optimize`方法。顾名思义，该方法是用来优化渲染的，事实上，通过添加`static`标志（`staticInFor`、`staticRoot`和`static`），使得`patch`简化`VNode`的`Diff`比较，在`render`时能直接渲染缓存的`tree`，在这里就不做深入展开了。

回到`generate`，`CodegenState`只有一个构造函数，主要提供一些实例属性，保存在`generate`过程中的状态，并最终获取到`staticRenderFns`。

```javascript
export function generate (
  ast: ASTElement | void,
  options: CompilerOptions
): CodegenResult {
  const state = new CodegenState(options)
  const code = ast ? genElement(ast, state) : '_c("div")'
  return {
    render: `with(this){return ${code}}`,
    staticRenderFns: state.staticRenderFns
  }
}
```

根据我们生成的`AST`，先调用`genElement`：

* `genStatic`：将`el.staticProcessed`标记为`true`，并重新执行`genElement`方法，将对应的`render`存入`staticRenderFns`，并返回`renderStatic`的字符串语句。
* `genOnce`：将`el.onceProcessed`标记为`true`，
  
  - 如果包含`v-if`，执行`genIf`，先转化`ifConditions`，获取到`genCode`结果后再调用`genStatic`，确保缓存到最终结果。
  - 如果位于`v-for`循环的`children`内（`el.staticInFor === true`），确保`v-for`包含`key`，避免`v-once`影响到循环体中的其他节点。
  - 直接调用`genStatic`，缓存结果语句。

* `genFor`: 将`el.forProcessed`标记为`true`，根据`el.for`、`el.alias`、`el.iterator1`和`el.iterator2`，用`renderList`循环`el.for`，并返回`genElement`的执行结果，可得到结果如：

  ```javascript
  `_l((barList), function(item, index) {
    return ${genElement(el, state)}
  })`
  ```

* `genIf`：将`el.ifProcessed`标记为`true`，避免无限递归调用，而后浅拷贝`el.ifConditions`，并调用`conditions.shift()`，根据`condition.exp`依次递归调用`genIfConditions`，拼接得到字符串语句：

  ```javascript
  // genIfConditions递归调用，直到conditions.length === 0
  `(link)
   ? ${genElement(condition.block, state)}
   : ${genIfConditions(conditions, state)}`
  ```

* `genChildren`：对`children`遍历调用`genElement`。
* `genSlot`：

  - 获取插槽里的`el.children`，在不存在`$slots[name]`插槽和`this.$scopedSlots[name]`作用域插槽时，渲染默认`children`。
  - 获取`slot`上的`attrs`和`dynamicAttrs`，供作用域插槽取值时使用。

* `genComponent`：动态`component`，根据`el.component`返回`_c`字符串语句。
* `element`：

  - 根据我们得到的`AST`语法树中的属性，生成`data`字符串。
  - 返回`_c`形式的字符串：
    
    ```javascript
    _c('${el.tag}'${
        data ? `,${data}` : '' // data
      }${
        children ? `,${children}` : '' // children
      })
    ```

通过`generate`，将所有的`el`的`AST`转化成字符串，最后通过调用`Function`构造函数，生成真正的`render`函数。
