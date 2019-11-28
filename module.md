最近学习了下模块加载，顺便写了个实现[liberty](https://github.com/imtaotao/liberty)，可以直接运行在浏览器端的类 cjs，写完了后想写点东西记录一下，怕以后思路给忘记了。

## 我需要哪些功能
一个模块中，我想要的是和 cjs 一样，有五个基础的全局变量，他们的功能与 cjs 中一样。
+ `require`
+ `module`
+ `exports`
+ `__filename`
+ `__dirname`

require 得到的模块是同步的，exports 是导出的模块。

## 具体的思路分为以下几步
1. 拿到文件源码
2. 建立一个沙箱环境，运行模块代码
3. 缓存模块
4. 以 url 当成唯一 id，一个模块代码只运行一次，后续的 require 会从缓存中获取

这是最初的想法，我也是按照这个简单的想法写下去的，但是在后面有些地方发生了变化，后面慢慢说。那此处值得说下的是，为什么要拿到文件源码，然后建立一个沙箱环境来运行，而不是像 seaJs 一样通过 script 标签 src 来加载代码，原因是，我加了插件系统，可以实时编译源码，想一下 webpack 是如何更改源码的。当然在浏览器中，这样实现安全问题不能保障，但在前端，安全本来就是个伪命题，何况这里只探讨原理。

## 架构流程
下面是模块加载的大概流程
```
             url                  url                     responseURL             code                  module
  import ----------> getCache ----------> requestResource ----------> getCache ----------> runPlugins ----------> exportModule
                       |                                                |
                       | module                                         | module
                       v                                                v
                  exportModule                                      exportModule 
```

上面整个流程是整个模块运行时的加载流程，那我们 require 一个模块的时候，其实是有两种选择的，是**同步加载**还是**异步加载**，显然的，同步加载更好，更加简单。但浏览器端不比在 node 环境中，在我们需要获取源码的这个要求中，只有同步的 xhr 能够满足我们，但是谁又能知道一个 xhr request 需要多少时间呢？更重要的是，同步的 xhr 被废弃了。关于是如果处理同步的问题的，请往下继续看

## 入口函数
对于入口函数，统一处理同步和异步
```js
  function importModule (path, parentInfo, config, isAsync) {
    const envPath = parentInfo.envPath
    const pathOpts = realPath(path, parentInfo, config)

    // 如果有缓存，就获取缓存
    if (cacheModule.has(pathOpts.path)) {
      const Module = cacheModule.get(pathOpts.path)
      const result = getModuleResult(Module)

      return !isAsync
        ? result
        : Promise.resolve(result)
    }

    // 否则就需要获取文件，然后走插件处理，最后得到结果了
    return isAsync
      ? getModuleForAsync(pathOpts, config, envPath)
      : getModuleForSync(pathOpts, config, envPath)
  }
```

## 缓存
模块缓存分为分为三块，`modulesCache`、`responseURLModulesCache`、`resourceCache`，其中，modulesCache 和 responseURLModulesCache 都是用来缓存 `module.export`，但是不一定是 `module.export`，只有通过 jsPlugins 插件处理过的模块，才会是这个，而这个 jsPlugins 是我们内置的。

我们为什么需要两个缓存对象来缓存模块？我们以 url 当 id 来保证模块的代码始终只会执行一次，对传入的 url 会做一次处理得到真实的 request path，因此而内置了一些 nodeJS path 模块的工具方法。
+ `path.normalize`
+ `path.isAbsolute`
+ `path.join`
+ `path.dirname`
+ `path.extname`

通过多级缓存，对 normalize 后的 url 进行判断，从而使得模块得以以 id 的形式被缓存下来，但是~ 如果同一份文件，url 完全不一样，就会导致这个模块的代码被执行多次，所以在这里添加了二级缓存，通过对 xhr 的 responseURL 缓存和判断，得以更加精准的缓存模块。由于两个 map 缓存的模块都是同一份，所以不用担心内存大小翻倍。

如果做到通过 responseURL 来缓存判断，我们不能等 xhr request 完全返回才去判断，多余的 request 流量消耗是需要避免的，所以可以在 `xhr.onreadystatechange` 进行获取，一但得到 responseURL 就终止掉整个 xhr request。

resourceCache 这个缓存是干嘛用的？往下看。

## 插件系统的设计
先确定插件怎样添加
```js
  Liberty.addPlugin('.js', fn)

```
每一种文件类型单独建立一个 set 队列，存放所有的回调，等源码文件获取到之后，runPlugins 的时候，遍历整个回调队列，把上一个回调的结果（修改后的源码字符串）传入到下个回调中，最终得到的结果就是我们需要返回模块，所以需要一个默认的插件用来返回模块（建立一个全局匹配符，默认返回所有的字符）。没错，类似 vue 中的管道符。

每一种类型我们都 new 一个 Plugins。
```js
  class Plugins {
    constructor (type) {
      this.type = type
      this.plugins = new Set()
    }

    add (fn) {
      this.plugins.add(fn)
    }

    forEach (params) {
      let res = params
      // 把上一个函数的参数传到下一个
      for (const plugin of this.plugins.values()) {
        res.resource = plugin(res)
      }
      return res
    }
  }
```

然后再建一个 map 存放所有类型的 Plugins
```js
  const map = {
    allPlugins: new Map(),

    add (type, fn) {
      if (!this.allPlugins.has(type)) {
        const pluginClass = new Plugins(type)
        pluginClass.add(fn)
        this.allPlugins.set(type, pluginClass)
      } else {
        this.allPlugins.get(type).add(fn)
      }
    },

    get (type = '*') {
      return this.allPlugins.get(type)
    },

    run (type, params) {
      const plugins = this.allPlugins.get(type)
      return plugins
        ? plugins.forEach(params)
        : params
    }
  }
```

最终我们的插件系统简单的实现，这样其实就够了，只需要一个暴露出去的接口就可以了，如下
```js
  function addPlugin (exname, fn) {
    // 可以 addPlugin('.js .html', fn) 同时对多种类型进行支持
    const types = exname.split(' ')
    if (types.length) {
      if (types.length === 1) {
        map.add(types[0], fn)
      } else {
        for (const type of types) {
          map.add(type, fn)
        }
      }
    }
  }
```

## jsPlugin
当插件系统设计好了之后，我们需要默认对 js 类型的文件进行处理，所以可以通过插件的形式，默认添加一个 jsPlugin 的形式处理。对于 js 源码，我们需要通过一个沙箱环境来运行。看了下 nodeJs 是如何来建立一个沙箱运行的，发现是调用了一个 v8 的接口 `runInThisContext`，这个 api 可以传入一个 code 源码和可选的 options。
```js
  const result = vm.runInThisContext(code, {
    filename: 'xx.js',
    displayErrors: true,
  })
```
这样就建立一个沙箱环境来运行模块代码，并且追踪文件名，于是打印的信息和 error 都会被更改成模块文件的文件名，so cool! 浏览器可没有这个 api。我们剩下的选择不多，就以下几种。

+ eval
+ new Function
+ 动态添加 script 标签

首先 `eval` 被排除掉了，这个东西效率极其低下，然后 `new Function`，这个比 `eval` 要好很多，但是通过 `new Function` 的方式来执行代码，可能不会得到 js 引擎的优化，如果所有的代码都是通过这种方式来执行，会特别影响整个项目的效率（其实一开始就是通过这种方式实现的），所以这种方式后来也被排除了。由于我们实现的是在浏览器这个平台上，所以最终通过动态添加 script 标签的方式来同步执行代码，然后我们可以用 sourcemap 的方式来定位源文件，这样就达到了和 `vm.runInThisContext` 差不多的功能，但是 soucemap 这个东西一言难尽，继续往下看。

### 如果执行 js 模块代码
每一个模块都是独立的作用域，用一个自执行 function 包裹起来就可以，模块需要的 5 个全局变量注入进去就好，所以大概可以分为以下几步来做。
1. 模块 function 的字符串拼接
2. 创建一个需要注入的对象（就是 module 和一些需要的参数）
3. 动态创建 script 标签
4. 把 script 标签添加到页面，执行代码
5. 得到结果

**创建模块的自执行函数**
```js
  // windowModuleName 需要自己随机取一个
  const parmas = Object.keys(rigisterObject) // ['require', 'module', 'exports', '__filename', '__dirname']
  const randomId = Math.floor(Math.random() * 10000)
  const windowModuleName = '__rustleModuleObject' + randomId

  let scriptCode =
    `(function (${parmas.join(',')}) {` +
    `\n${basecode}` +
    `\n}).call(undefined, window.${windowModuleName}.${parmas.join(`,window.${windowModuleName}.`)});`

  // 生成 soucemap
  if (config.sourcemap) {
    scriptCode += `\n${sourcemap(scriptCode, responseURL)}`
  }
```

**创建需要注入到 window 的参数对象**
```js
  const Module = { exports: {} }
  const parentInfo = getParentConfig(path, responseURL)
  readOnly(Module, '__rustleModule', true)

  const require = path => importModule(path, parentInfo, config, false)

  // 加上俩异步加载模块的方法
  require.async = path => importModule(path, parentInfo, config, true)
  require.all = paths => importAll(paths, parentInfo, config)

  const rigisterObject = {
    require,
    module: Module,
    exports: Module.exports,
    __filename: responseURL,
    __dirname: parentInfo.dirname,
  }
```

**动态创建 script 标签并执行代码**
```js
  const node = document.createElement('script')
  node.text = scriptCode

  window[windowModuleName] = rigisterObject
  document.body.append(node)
  document.body.removeChild(node)

  delete window[windowModuleName]
```

最后 return 这个 `rigisterObject` 就可以了，需要注意的是，在执行代码之前需要先缓存，因为要处理循环 require 的问题。
```js
  // 缓存模块，在运行代码之前缓存，这样在执行代码的时候，有循环 require 的问题，就和 cjs 差不多了
  cacheModule.cache(path, rigisterObject.module)
  responseURLModules.cache(responseURL, rigisterObject.module)

  // 运行代码
  run(scriptCode, rigisterObject)

  // 然后清除掉，因为有可能代码的执行报错了，那就需要清除掉缓存
  // 这里直接清除掉缓存是因为在整个模块执行完毕后会再次缓存，包括其他类型的模块，所以这里直接删除即可
  cacheModule.clear(path)
  responseURLModules.clear(responseURL)
```

## 其他的 plugin
除了 jsPlugin，还可以有其他的 plugin，用来处理各种各样的文件类型，比如对所有的 json 文件自动 parse 就可以这样写。
```js
const JSONPlugin = res => JSON.parse(res.resource)
Liberty.addPlugin('.json', JSONPlugin)
```

比如要对 css 文件做 cssModule，[cssModule](https://github.com/imtaotao/css-module/blob/master/index.js)，这是一个简单的 cssModule 插件。还可以对 vue 的单文件做处理，都是可以的。

## 整个架构
```         
  +------+            +----------+         +--------+
  | core | ---------> | plugins  | ------> | result |
  +------+            +----------+         +--------+
                      
```

+ core 是整个模块加载器的核心部分，他只负责保证以同步或异步的方式加载模块资源，并缓存和读取模块的缓存，但不对模块内容做具体的处理
+ plugins 是模块真正处理的部分，他只对不同的模块做不同的处理，不关心模块从何而来，也不关心模块最终会流向哪里，只单纯的做转换处理

## resourceCache 是干嘛用的
要理解缓存静态资源和缓存模块在于，这俩东西有什么区别，在 cjs 中，我们只需要缓存模块，没有缓存资源一说。几个前提需要说一说。

+ 静态资源和模块的区别
+ 如果需要同步加载模块，在浏览器中需要如何保证
+ 同步处理模块时，如果保证相对可靠的性能

下面一一做解答，看完后，你就能明白 `resourceCache` 是干嘛用的。

Q: 静态资源和模块的区别<br>
A: 静态资源就是文件的内容，是通过 xhr 去获取的，由于我们没有对音视频这种特殊的模块做兼容，所以我们拿到的文件内容都是字符串，而模块，就是这些文件内容通过插件系统处理后的内容，所以说，只有经过插件处理过才会是模块，而最终的使用方能够拿到的只能是模块。看看下面例子。

```js
Liberty.addPlugin('.json', res => {
  // res.resource --> 在这里就是静态资源
  return res.resouce // 虽然这个模块对静态资源没有做任何的改变，但这就是模块，因为经过了静态资源的转换
})

Liberty.addPlugin('.json', res => {
  return JSON.parse(res.resouce) // 而这个插件做了 parse, 返回的也是模块
})
```

Q: 如果需要同步加载模块，在浏览器中需要如何保证<br>
A: 我们是通过 xhr 去加载的，一般情况下我们都是异步加载的，这就难以保证我们同步得到模块。所以有了以下的思考。
  + 如果我们通过同步的 xhr 去加载，会如何
  + esm 是怎么做的
  + http1.1 有什么局限

#### 如果我们通过同步的 xhr 去加载，会如何
通过同步的 xhr 去加载当然可以做到同步加载模块，但问题是，每一个模块都会阻塞 js 线程，必须等模块加载完成才会继续执行代码，一个还好，要是有几十上百个模块，这种体验你能忍？关键是，同步的 xhr 已经被标准给[废弃了](https://s0developer0mozilla0org.icopy.site/en-US/docs/Web/API/XMLHttpRequest/Synchronous_and_Asynchronous_Requests)。

#### esm 是怎么做的
`tree shaking` 这个词听腻了吧，为什么只能 esm 可以做，cjs 不能做呢。理由是 esm 是静态的，并不是在运行时加载，所以以下这种语法是不被允许的。
```js
import m from 'a' + 'b.js'
```
esm 会深度遍历你的模块依赖关系，并去请求文件，当所有的资源准备完毕后，才会运行代码。既然如此，我们也可以这样做。我们每次加载一个静态资源的时候，也便利依赖关系，并将资源缓存起来。文件资源可能很大，所以换成我们可以用 `storeage`，这样就避免了内存撑爆。当然，在 `liberty` 这个中还是缓存在内存中的。
```js
// 这种写法不会被检测到
const urls = ['/a.js', '/b.js']
urls.forEach(v => require('/dev' + v))


// 而下面这种纯字符串路径是可以被检测到的
require('/dev/a.js')
require('/dev/b.js')
```
同样的，我们不允许运行时的 `require`。 但在 `liberty` 中做的处理更多的，兼容的情况也更多，这里只简述原理。

#### http1.1 有什么局限
http1.1 的问题是，即使可以同时建立多条 tcp 链接，这个数量也是极其有限的，现在一般常用的是 6 条，这就需要你能够合理的对你的模块文件做合并与切分，需要做的优化类似于在 webpack 中做 `code splitting`。如果换成 http2，这些问题就都不存在了，因为在 http2 中你可以使用 `多路复用` 特性。

## sourcemap
sourcemap 的资料，推荐以下几篇文章
  + [Source Map Revision 3 Proposal](https://docs.google.com/document/d/1U1RGAehQwRypUTovF1KRlpiOFze0b-_2gc6fAH0KY0k/edit?hl=en_US&pli=1&pli=1#)
  + [Source Map的原理探究](https://cloud.tencent.com/developer/article/1354581)

需要注意的是，行与列都是相对位置，也就是相对上一行或者列的位置，弄错了会让你的 soucemap 位置与实际位置差了很远很远~（被坑了一整天）

## 其他的问题
当然还有其他的问题，很多不完善的地方，比如
+ 现在所有的插件都是在第一次 require 的时候运行，这导致在插件中只能同步的返回，那如何增加能够异步处理的能力？
+ 有什么更高效的办法来检测依赖关系，而不是现在这种通过正则全文搜索的方式
+ sourcemap 能够做成一套 sdk，以便于插件使用，而不是自己去生成

文笔有限，期待与你一起思考~