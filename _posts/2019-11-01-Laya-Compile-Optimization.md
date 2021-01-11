---
layout: post
title: 'Laya编译发布优化'
subtitle: '讨论bundle分离，编译速度优化，sourcemap生成与Linux命令行工具'
date: 2019-11-01
author: 周子博
categories: H5
cover: 'https://i.loli.net/2019/11/01/YbJWUTpzCKRDhmP.jpg'
tags: H5 Laya
---

## Laya IDE编译与发布的一些问题

随着游戏开发的推进，越来越多的功能和内容被添加，IDE的编译与发布也越来越慢，逐渐暴露了一些流程问题。

1. bundle文件过大需要拆分。假设把sourcemap的内容独立出去，bundle也有9.4MB之大，uglify压缩后也有4MB多。日后稍微更新一点内容，这个庞大的文件就要用户去重新下载。倘若频繁更新发布，网络请求对服务器造成的负担不说，对用户而言加载速度则是很大问题。
2. 编译速度太慢。稍微改动一点点内容便要花1分钟去编译与查看效果。
3. browserify打包生成内嵌sourcemap导致bundle膨胀。我们使用2.0的IDE版本，在什么也不修改的情况下，bundle体积已经膨胀到30MB。其中21MB都是browserify打包生成的符号文件sourcemap的内容，被base64编码后添加到bundle的最后一行。
4. Laya压缩混淆js不生成符号文件。发布时勾选Laya IDE中的压缩混淆js选项，js被混淆，却不会生成映射到typescript源码的符号文件。这对发布后收集错误堆栈时的解析有很大影响。

Laya IDE的编译与发布实际上是在IDE安装目录的node环境下执行工程目录下.laya文件夹中的gulp脚本。所以基于以上问题，我对Laya IDE的编译发布脚本做了一些改动。

<img src="https://i.loli.net/2019/11/01/zYWEKolCXImRky7.png" alt="编译脚本" data-action="zoom"/>

## 拆分bundle

首先解决bundle过大的问题。browserify工具没有提供很好的解决方案，但是我们可以很轻松的用webpack对其进行模块拆分。二者都是将typescript进行模块打包，但是webpack很友好的提供了splitChunks的插件，简单配置一下我们就能将bundle拆分若干份符合我们大小要求的文件。这也是我们微信小游戏分包使用的策略。

首先我们全局安装拆分需要的npm工具。由于我们使用typescript编写项目，所以还需要安装typescript编译需要的一些包。

~~~shell
npm i typescript  ts-loader clean-webpack-plugin webpack webpack-cli -g
npm link ts-loader clean-webpack-plugin
~~~

其中，在工程根路径下只需要链接全局安装的`ts-loader` `clean-webpack-plugin`两个包，修改tsconfig.json：

~~~javascript
{
  "compilerOptions": {
    "module": "commonjs",
    "target": "es5",
    "lib": [
      "es6",
      "dom"
    ],
    "noEmitHelpers": false,  //需要生成一些helper方法，否则会报__extend undefined
    "sourceMap": false
  },
  "exclude": [
    "node_modules"
  ]
}
~~~

然后配置webpack：

~~~javascript
const path = require('path');
const { CleanWebpackPlugin } = require('clean-webpack-plugin');
const distFolder = "./dist";

module.exports = {
  mode: 'development',
  entry: {
    bundle2:'./src/main.ts', // 项目主入口
    bundle1:'./src/a/b.ts', // 另一个入口
  },
  plugins: [
    new CleanWebpackPlugin()
  ],
  devtool: 'source-map',
  module: {
    rules: [
      {
        test: /\.ts$/,
        use: 'ts-loader',
        exclude: /node_modules/
      }
    ]
  },
  optimization: { // 配置splitChunks插件，拆分公共的代码块出来
    splitChunks: {
      chunks: 'all',
      minSize: 300000, // 拆出来的代码块最小约300kb
      maxSize: 3000000 // 拆出来的代码块最大约3MB
    }
  },
  resolve: {
    extensions: [ ".ts" ]
  },
  output: {
    filename: '[name].js',
    path: path.resolve(__dirname, distFolder)
  }
};
~~~

在工程目录执行webpack打包后，就可以在dist文件夹下看到拆分后的bundle了。将其引入index.js中即可运行查看效果，推荐在bin目录使用[live-server](https://www.npmjs.com/package/live-server)快速搭建http服务器查看效果。

> webpack拆分出来的模块可以异步加载，可以打乱顺序引入index.js中运行，这也是微信小游戏分包的关键。

## 加快编译速度

目前工程体量里，typescript源文件有1000多个，执行一次编译需要耗时50多秒。编译时间之久，导致很难实现编写代码-浏览器查看效果-修改代码-浏览器自动刷新查看效果这样理想的工作流了。不过可以通过增量编译来加快编译速度，尽可能实现理想的工作流。关于增量编译可以有三种实现。

### tsc -w

在Laya2.0版本以前，项目并不会用import，export去管理模块关系，每个js文件都是全局执行，顺序引入，所以可以使用`tsc -w -p . --outDir bin/js`命令开启typescript增量编译输出js文件，然后用脚本处理index.js依照依赖关系顺序引入执行。使用`-w`选项开启增量编译后，每修改一个typescript文件就会在毫秒内编译成js。浏览器刷新后就能立刻看到效果。这是最符合理想情况的开发环境了，但是考虑到现有的项目都是模块化编写，写法不同于全局变量的方式，不同模块间都有import与export引用关系。使用`tsc`编译出来的js文件并不能顺序执行，这也是需要打包的原因。所以最后也没有使用这种方案。

> 想要获取源码文件的依赖关系顺序，可以使用[dependency-tree](https://github.com/dependents/node-dependency-tree)工具。  
> 想要浏览器监测到文件变化自动刷新，推荐使用[webpack-dev-server](https://www.npmjs.com/package/webpack-dev-server)或者是[live-server](https://www.npmjs.com/package/live-server)工具，使用webpack打包或者自行修改一些js代码时，比起耗费编译的大量时间用Laya的运行调试来看效果，不如使用本地工具快速跑一个http服务器来测试。

### watchify

第二种方法是继续使用browserify打包，同时使用watchify工具。可以参考[gulp官网](https://v3.gulpjs.com.cn/docs/recipes/fast-browserify-builds-with-watchify/)使用watchify进行增量编译。这需要修改编译脚本.laya/compile.js，安装两个npm包:

~~~powershell
npm i watchify gulp-util -g

# 链接Laya IDE所使用的Node环境中一些包到全局安装的包上去
cd LayaAirIDE_2.0.2\resources\app
npm link watchify gulp-util
~~~

然后修改Laya工程根目录.laya中的compile.js编译脚本，以下提供compile.js***添加或修改***的部分用于参考:

~~~javascript
//添加引用watchify
let watchify = require(ideModuleDir + "watchify");
let gutil = require(ideModuleDir + "gulp-util");

let b = watchify(browserify({
  basedir: workSpaceDir,
  //是否开启调试，开启后会生成jsmap，方便调试ts源码，但会影响编译速度
  debug: true,
  entries: ['src/Main.ts'],
  cache: {},
  packageCache: {}
}).plugin(tsify));//使用tsify插件编译ts

function bundleOnWatch() {
  console.time("增量编译耗时");
  console.log("[" + Date().toLocaleString() + "]", "检测到文件更改，开始增量编译...")
  b.bundle()
    //使用source把输出文件命名为bundle.js
    .pipe(source('bundle.js'))
    //把bundle.js复制到bin/js目录
    .pipe(gulp.dest(workSpaceDir + "/bin/js")
    .on('end', function() {
        console.log("[" + Date().toLocaleString() + "]", "编译完成!");
        console.timeEnd("增量编译耗时");
      })
    );
}

//使用browserify，转换ts到js，并输出到bin/js目录
gulp.task("compile", prevTasks, function () {
  /**
   * ...省略
   * */

  return b
    .bundle()
    //使用source把输出文件命名为bundle.js
    .pipe(source('bundle.js'))
    // 把bundle.js复制到bin/js目录
    .pipe(gulp.dest(workSpaceDir + "/bin/js"));
});

b.on("update", bundleOnWatch); // 每次监测到typeScript文件改变时执行打包
b.on("log", gutil.log); // 将日志打印到控制台
~~~

实测效果第一次编译和正常编译一样，需要50s，但是增量编译缩短至25s，打包速度提升了50%，如果不需要调试文件即`debug: false`，增量编译缩短至19s。再使用热重载的本地http服务器，勉强可以实现相对理想的工作流。

### webpack --watch

webpack4本身的打包速度就比Laya使用的browserify快，同样的项目体量，不生成sourcemap符号文件，browserify需要42s，webpack打包只需要37秒。并且使用`webpack --watch`开启增量编译模式后，第一次以后的打包时间缩短到18秒，再配置webpack-dev-server实现自动reload，这样的工作流也算是比较理想了。

## 编译browserify打包时拆分出独立sourcemap文件

如果继续使用browserify，在工程体量变大后还会遇到一个问题就是browserify打包生成的是inline-sourcemap，内嵌到bundle.js文件中。如果不做改动，现在编译出一个bundle可以达到30MB，其中base64编码后的sourcemap内容占21MB，先不考虑发布是否压缩的问题，连内网浏览时去下载执行js都需要耗很长的时间，游戏此间一直为白屏。为此可以使用`exorcist`抽离出sourcemap为独立文件。需要安装`exorcist`包:

~~~powershell
npm i exorcist -g

# 链接Laya IDE所使用的Node环境中一些包到全局安装的包上去
cd LayaAirIDE_2.0.2\resources\app
npm link exorcist
~~~

然后修改Laya工程根目录.laya中的compile.js编译脚本，以下提供compile.js***添加或修改***的部分用于参考:

~~~javascript
// 提取分离sourcemap
let exorcist = require(ideModuleDir + "exorcist");

// 使用browserify，转换ts到js，并输出到bin/js目录
gulp.task("compile", prevTasks, function () {
  /**
   * ...
   * */
  return browserify({
    basedir: workSpaceDir,
    // 是否开启调试，开启后会生成jsmap，方便调试ts源码，但会影响编译速度
    debug: true,
    entries: ['src/Main.ts'],
    cache: {},
    packageCache: {}
  })
    // 使用tsify插件编译ts
    .plugin(tsify)
    .bundle()
    // 分离sourcemap
    .pipe(exorcist(workSpaceDir + "/bin/js/bundle.js.map"))
    // 使用source把输出文件命名为bundle.js
    .pipe(source('bundle.js'))
    // 把bundle.js复制到bin/js目录
    .pipe(gulp.dest(workSpaceDir + "/bin/js"));
});
~~~

<img src="https://i.loli.net/2019/11/01/PsJnZLhbz8e4kGm.jpg" alt="分离sourcemap" data-action="zoom"/>  
\**被分离后的bundle与sourcemap文件*\*

## 发布时生成sourcemap

<img src="https://i.loli.net/2019/11/01/iBYR5TsDKc7jyzo.jpg" alt="laya发布" data-action="zoom"/>
\**laya发布界面*\*

发布时Laya提供压缩混淆js的选项，其使用`gulp-uglify`进行压缩，却不生成sourcemap符号文件，给开发者日后解析外网收集的报错带来了不少困难。为此需要使用`gulp-sourcemaps`，在uglify压缩过程生成sourcemap文件，并且最好还能与上一步`browserify`打包分离出来的sourcemap文件合并成一个map文件，可以直接从js混淆后代码映射到typescript源码。为此需要npm安装`gulp-sourcemaps`:

~~~powershell
npm i gulp-sourcemaps -g

# 链接Laya IDE所使用的Node环境中一些包到全局安装的包上去
cd LayaAirIDE_2.0.2\resources\app
npm link gulp-sourcemaps
~~~

然后修改Laya工程根目录.laya中的publish.js发布脚本，以下提供publish.js***添加或修改***的部分用于参考:

~~~javascript
// 引入gulp-sourcemaps生成sourcemap
const sourcemaps = require(ideModuleDir + 'gulp-sourcemaps');
/**
 * ...
 * */
// 压缩js
gulp.task("compressJs", ["compressJson"], function () {
  if (config.compressJs) {
    return gulp.src(config.compressJsFilter)
      .pipe(sourcemaps.init({loadMaps:true, largeFile:true}))
      .pipe(uglify())
      .on('error', function (err) {
        console.warn(err.toString());
      })
      .pipe(sourcemaps.write("sourcemaps", {addComment:false}))
      .pipe(gulp.dest(releaseDir));
  }
});
/**
 * ...
 * */
gulp.task("publish", ["version2"], function () {
    /**
     * ...
     * */
    // 删除编译打包过程中生成的符号文件
    let tmpSourcemapFile = `${releaseDir}/js/bundle.js.map`;
    if (fs.existsSync(tmpSourcemapFile)) {
        fs.unlinkSync(tmpSourcemapFile);
    }
    console.log("All tasks completed!");
});
~~~

> 外网收集的报错堆栈如何解析请参考我的另一篇文章[sourcemapping-JavaScript混淆堆栈解析映射工具](/2019/09/11/JavaScript-Source-Mapping.html)

## 关于Linux下命令行工具的讨论

最终我们使用Linux机器执行发布编译，使用的是Laya官方提供的`layaair2-cmd`这个npm工具，用于命令行编译与发布，实际上也是执行的gulp脚本。但是linux上没有Laya IDE，使用的是机器全局的node环境。所以可以全局安装以上依赖的npm包，手动修改`layaair2-cmd`工具中的编译发布脚本即可实现以上的特性。

> 官方提供的命令行工具`layaair2-cmd`经常还未开发测试完成便发布新版本，从1.3.0以后的版本开始，到目前最新发布的1.4.5都没法正常跑编译与发布，建议保留1.2.0版本，不要轻易升级。并且命令行工具版本与Laya引擎版本有对应关系，也不建议轻易升级Laya引擎版本。像这样关键内容的版本发布如此随意，可以看出Laya内部并没有做很好的流程管理和版本控制。希望还是能把流程把控好，虽然可能效率低一些，但是正式和规范更能够赢得开发者的信任。多学习crbug.com管理和修复Chrome的流程，一个庞大且优秀的项目，只有流程化规范化才能走的更远。

## 参考

1. [使用watchify加速browserify编译 - Gulp](https://v3.gulpjs.com.cn/docs/recipes/fast-browserify-builds-with-watchify/)
2. [watchify - npm](https://www.npmjs.com/package/watchify)
3. [exorcist - npm](https://www.npmjs.com/package/exorcist)
4. [gulp-sourcemaps - npm](https://www.npmjs.com/package/gulp-sourcemaps)
5. [live-server - npm](https://www.npmjs.com/package/live-server)