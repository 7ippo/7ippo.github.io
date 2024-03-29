---
layout: post
title: 'Laya项目轻松上线微信小游戏'
subtitle: '讨论拆分bundle以满足4MB限制，以及上线微信小游戏可能遇到的问题'
date: 2020-01-18
author: 周子博
categories: H5
cover: 'https://i.loli.net/2020/03/30/2xlYnmaEKhpw37i.jpg'
tags: H5 Laya
---

Laya引擎为了H5游戏上线微信等小游戏平台的无缝对接做了许多工作，但是项目上线微信小游戏还是遇到了不少的问题。不过都已经一一解决，现在把收获的经验和解决问题的方法和同学们分享一下。以下我就按照从Laya发布小游戏开始，问题可能出现的顺序简单讨论一下。

## 挂载全局变量到window域

将发布的目录导入微信开发者工具中编译后，最容易出现的问题是一些已经引入的js文件中的全局变量报undefined，常见于我们自己写的js库。这个问题和微信的[模块机制](https://developers.weixin.qq.com/minigame/dev/guide/framework/module.html){:target="_blank"}有关，微信的JavaScript引擎与浏览器中的不同，浏览器js执行时默认绑定全局变量到window域。在微信小游戏环境下，Laya导出的index.js中require引入的js文件并不会将其全局变量暴露给其他js文件调用。

![模块机制.png](https://i.loli.net/2020/03/27/ZHkmqBbgrAWPFeC.png){:data-action="zoom"}{:style="max-width: 80%;"}  
\**微信小游戏的模块机制*\*

小游戏环境中其实也有一个类似浏览器中window的全局对象：GameGlobal，且微信官方对此做了一定的polyfill工作，其工程目录下的`weapp-adapter.js`允许我们只要显示的将全局变量绑定到window域下，即可在其他文件里通过window调用。比如引入了logline.js第三方库，那在该文件中我们可以显示地绑定:

~~~javascript
// 暴露Logline变量到window域给其他js调用
window.Logline = Logline
~~~

其他js文件此时便可以使用`Logline.debug()`方法了。其实可以查看laya.core.js等等laya自己写的js libs文件，都是有做这个处理的。

~~~javascript
// laya.core.js Laya version 2.2
window.Laya= (function (exports) {...}({}))
~~~

## 引用第三方DOMParser库

微信小游戏环境缺少对HTMLDivElement组件的支持(游戏中UI富文本会用到，而大部分浏览器都是支持的)，所以在微信开发者工具中运行时，laya底层库会抛出`需要引入xml解析文件`的错误。Laya官方给出了[解决方案](https://ask.layabox.com/question/13487){:target="_blank"}。我们只需要拷贝三个js文件至laya libs库中，并且在主包game.js中引入即可：

~~~javascript
// game.js
window.Parser = require("./libs/dom_parser");
~~~

## 部分API不支持

检查是否有使用微信小游戏环境不支持的Web API。比如我们项目使用了`Performance`用于性能统计，在小游戏环境不支持，需要屏蔽掉。

> 微信不支持直接通过`Performance`来调用，而是做了一层[封装](https://developers.weixin.qq.com/minigame/dev/api/base/performance/wx.getPerformance.html){:target="_blank"}，需要通过`wx.getPerformance()`来调用，且只支持其部分功能。所以如果发现API不支持，可以先在[官方文档](https://developers.weixin.qq.com/minigame/dev/api/){:target="_blank"}搜索一下是否有替代办法。

## 合法域名与HTTPS

不出意外的话，到这一步已经可以在**微信开发者工具**中跑通了（因为开发者工具中预览对包体大小没有限制）。记得检查一下是否在*详情 - 项目*中勾选了*不检查合法域名与HTTPS*。否则就要求网络请求均为https协议，WebSocket连接均为wss协议，且在微信后台[配置](https://developers.weixin.qq.com/minigame/dev/guide/base-ability/network.html){:target="_blank"}后才可以继续运行，这也是真机上预览的要求。

![不校验域名.png](https://i.loli.net/2020/03/27/3JprQta7wPKUekB.png){:data-action="zoom"}{:style="max-width: 60%;"}      
\**勾选不校验域名*\*

## 分包

由于微信小游戏限制每个本地包大小最大为4MB，虽然支持多个分包，但总大小也限制为12MB，且无法动态下载执行js。这就意味着我们需要把所有js代码放在包里，而资源尽量动态加载不要占用宝贵的包体大小。

### 资源单独发布

相信没有多少中等规模的项目可以将游戏资源总大小控制在12MB以内，所以第一步要做的就是把资源单独抽离出去，发布在线上。为此需要在加载完本地的资源后，设置一下线上资源地址，这一步往往在index.js文件末尾处理，可以参考Laya官方的[说明](https://ldc2.layabox.com/doc/?nav=zh-ts-5-0-4){:target="_blank"}。

~~~javascript
// index.js
Laya.URL.basePath = "https://XXXX.com/";
~~~

### 拆分bundle.js - 大型项目最重要的一步

Laya执行gulp脚本中的打包工具（旧版本使用browserify.js，新版本使用rollup.js）将typescript源码编译生成一份bundle.js，以管理源码各文件模块的依赖关系。但是这也不可避免的导致了项目上架微信小游戏时的窘迫，当项目规模变大时，bundle文件在压缩后也能很轻松地突破4MB的限制（我们目前的项目压缩后的bundle已经有6.2MB）。这还不算上laya引擎的文件（已经有将近1.8MB）以及我们自己写的一些需要预先加载的js文件，这些都是必须放在主包中先加载的。

所以我们需要使用一些支持拆分bundle的打包工具，将bundle.js拆分成若干部分，使其某一部分可以放在主包先加载，其他部分可以在若干个分包中加载。我第一个想起了webpack。

> 其实在想到用webpack拆分前我也试过很多别的方法，比如升级到Laya 2.3用rollup打包，其比用browserify打包体积可以减少0.2MB（主要归功于ES6），然后删减了所有日志打印、测试类以及性能统计方法，想了很多办法尽量减少压缩后字符数，也只能将bundle再缩减0.2MB，这0.4MB并不能本质上解决问题。恰好我之前在开发H5SDK时接触了webpack，所以才会想到是否可以用webpack的插件来拆分打包，并后来成功验证了这一想法。

webpack很友好的提供了[splitChunks插件](https://webpack.js.org/plugins/split-chunks-plugin/){:target="_blank"}，本身是用于拆分两个包体的公共代码块的，简单配置一下我们就能将bundle拆分成若干份符合我们大小要求的文件。

首先我们全局安装需要用到的npm工具。由于我们使用typescript编写项目，所以还需要安装typescript编译需要的一些包。

~~~shell
npm i typescript  ts-loader clean-webpack-plugin webpack webpack-cli -g
npm link ts-loader clean-webpack-plugin
~~~

其中，在工程根路径下只需要`npm link`链接全局安装的`ts-loader` `clean-webpack-plugin`两个包。修改工程tsconfig.json：

~~~javascript
{
  "compilerOptions": {
    "module": "commonjs",
    "target": "es5",
    "lib": [
      "es6",
      "dom"
    ],
    "noEmitHelpers": false,
    "sourceMap": false
  },
  "exclude": [
    "node_modules"
  ]
}
~~~

因为我们要用`splitChunks`拆分两个bundle的公共代码块，就要新生成一个与主bundle有大量相同代码块的分bundle，所以我们可以选择一些基础模块作为新的打包入口。配置webpack.config.js如下：

~~~javascript
const path = require('path');
const { CleanWebpackPlugin } = require('clean-webpack-plugin');
const distFolder = "./dist";

module.exports = {
  mode: 'production',
  entry: {
    bundle:'./src/Main.ts',
    // 选择某一个入口文件打包用于拆分与主包相似的代码块，尽量选择一些基础模块
    delete:'./src/a.ts'
  },
  plugins: [
    new CleanWebpackPlugin()
  ],
  devtool: 'source-map',
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: 'ts-loader',
        exclude: /node_modules/
      }
    ]
  },
  optimization: {
    splitChunks: { // 配置splitChunks插件
      chunks: 'all',
      minSize: 300000, // 拆出来的代码块最小约300kb
      maxSize: 3000000, // 拆出来的代码块最大约3MB
      name: false
    }
  },
  resolve: {
    extensions: [".ts"]
  },
  output: {
    filename: '[name].js',
    path: path.resolve(__dirname, distFolder)
  }
};
~~~

在根目录下运行`webpack`，会在dist目录下生成打包后的文件以及对应的sourcemap文件。sourcemap文件要保留好，日后出现报错堆栈可以通过该符号文件解析到源码。关于堆栈解析映射可以使用我写的这个[sourcemapping工具](/2019/09/11/JavaScript-Source-Mapping.html){:target="_blank"}。

![dist分包结果.png](https://i.loli.net/2020/03/28/nwtgkrBPoLDG5sy.png){:data-action="zoom"}{:style="max-width: 60%;"}  
\**webpack-splitChunks打包结果*\*

delete.js文件只是我们用来拆分bundle的一个产物，因为其模块也被其他模块所依赖，所以已经被打进公共代码块，故可以将其删除。剩余的js文件则随意分成几个部分，放入小游戏主包与分包中，使其满足4MB大小限制即可。通过微信的分包机制先加载主包，主包中最后再加载各个分包。webpack的加载机制保证了公共代码块乱序引入也可以正确的加载打包模块的依赖关系。所以尽量使主包接近4MB大小限制即可。关于微信的分包实现就不赘述，非常简单可以参考微信官方[文档](https://developers.weixin.qq.com/minigame/dev/guide/base-ability/sub-packages.html){:target="_blank"}。

## 真机测试

如果已经配置好了https与wss连接，并且实现了分包，那么至此就可以成功在真机上预览以及上传发布了。关于微信小游戏，Laya项目还需要做一些别的优化，比如如何高效利用50MB缓存空间（基于Laya那套管理逻辑再优化），内存监控，日志模块实现等，这些日后看是否有需要单独拿出来讨论。

## 参考

1. [微信开放文档 - 微信](https://developers.weixin.qq.com/minigame/dev/guide/){:target="_blank"}
2. [splitChunksPlugin - webpack](https://webpack.docschina.org/plugins/split-chunks-plugin/){:target="_blank"}
3. [微信小游戏适配文档 - Laya](https://ldc2.layabox.com/doc/?nav=zh-ts-5-0-6){:target="_blank"}
