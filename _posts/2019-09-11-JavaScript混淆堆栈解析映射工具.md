---
layout: post
title: 'js混淆堆栈本地解析映射工具'
subtitle: 'sourcemapping - 我为node社区贡献的第一个npm工具'
cover: 'https://i.loli.net/2019/09/11/DwE4xOzsCRGyt9H.jpg'
date: 2019-09-11
author: zpo
categories: H5
tags: H5 npm 工具
---

## 为什么要写这个工具

发布环境的产品都经过压缩混淆，并且没有attach sourcemap。因此前端收集到的错误堆栈都是被混淆过的，文件与行号列号变量名都可能发生了变化。因此需要通过sourcemap文件映射到源码上去。如果使用typescript编译+压缩混淆生成的sourcemap文件，可以直接映射到typescript源码。因此针对前端`window.onerror`捕获到的错误堆栈写了这样一个工具用于解析与映射，已经在[npm](https://www.npmjs.com/package/sourcemapping)发布。  
![npm发布.png](https://i.loli.net/2019/09/11/ynP6tEe3Nwo9Tpb.png)

## 使用方法

需要传入3个参数：

- `window.onerror`中捕获的`JSON.stringfy(errorObj.stack)`即需要被解析的堆栈字符串
  > e.g.  
  > `ReferenceError: exclued is not defined\n    at getParameterByName (http://localhost:7777/aabbcc/logline.min.js:1:9827)\n    at http://localhost:7777/aabbcc/index.js:15:11`

- `window.onerror`捕获的`errorMessage`即错误信息
  > e.g. `Uncaught ReferenceError: a is not defined`

- 存放sourcemap文件的路径，绝对路径或相对路径都可以
  > e.g. `./test`  
  > 注意：sourcemap文件命名规则为各压缩混淆工具的默认规则，即:javascript文件名.map，需要直接存放在传入的路径下

<pre><code class="language-shell">Usage: sourcemapping [options]

Options:
  -v, --version         output the version number
  -s, --stack &lt;string&gt;  stack string which can obtain from JSON.stringfy(Error.stack)
  -i, --msg &lt;string&gt;    error message. e.g. Uncaught ReferenceError: a is not defined
  -m, --map &lt;string&gt;    sourcemap dir path. Where to find sourcemap
  -h, --help            output usage information

</code></pre>

For example:

<pre><code class="language-shell">sourcemapping -s "ReferenceError: exclued is not defined\n    at getParameterByName (http://localhost:7777/aabbcc/logline.min.js:1:9827)\n    at http://localhost:7777/aabbcc/index.js:15:11" -i "Uncaught ReferenceError: exclued is not defined" -m "./test"
</code></pre>

## 输出

默认输出到控制台

<pre><code class="language-shell">----Sourcemap Result----
Uncaught ReferenceError: exclued is not defined
    at Logline (../src/logline.js:62:31)
    at (index.js:15:11)
------------------------
</code></pre>

## [☆Github源码☆](https://github.com/7ippo/sourcemapping)

<pre><code class="language-typescript">// sourcemapping.ts
#!/usr/bin/env node

import * as path from 'path';
import * as ErrorStackParser from 'error-stack-parser';
import * as commander from 'commander';
import { readFileSync, existsSync } from 'fs';
import { SourceMapConsumer } from 'source-map';

let raw_stack_string_array: string[];

function stackStringProcess(value: any, previous: any): string {
    // input '\n' will be translated into '\\n' and cause ErrorStackParser parsing failure
    raw_stack_string_array = value.split(String.raw`\n`);
    return raw_stack_string_array.join('\n');
}

function printToConsole(error_msg: string, stack_frames: ErrorStackParser.StackFrame[]): void {
    console.log("----Sourcemap Result----")
    console.log(error_msg);
    for (const frame of stack_frames) {
        let msg = "    at ";
        if (frame.functionName) msg += frame.functionName + " ";
        msg += "(";
        if (frame.fileName) msg += frame.fileName + ":";
        if (frame.lineNumber) msg += frame.lineNumber + ":";
        if (frame.columnNumber) msg += frame.columnNumber;
        msg += ")";
        console.log(msg);
    }
    console.log("------------------------")
}

async function loadAllConsumer(dir_path: string, stack_frame_array: ErrorStackParser.StackFrame[],
    sourcemap_map: Map&lt;string, SourceMapConsumer&gt;) {
    // load all sourcemap files into memory
    const sourcemap_list = new Set();
    const regExp = /.+\/(.+)$/;
    for (const frame of stack_frame_array) {
        if (frame.hasOwnProperty('fileName')) {
            const name = regExp.exec(frame.fileName)[1];
            frame.fileName = name;
            if (!sourcemap_list.has(name)) {
                sourcemap_list.add(name);
                let sourcemap_filepath = path.join(dir_path, name + '.map');
                if (existsSync(sourcemap_filepath)) {
                    let sourcemap: any;
                    try {
                        sourcemap = JSON.parse(readFileSync(sourcemap_filepath, 'utf-8'))
                    } catch (error) {
                        console.error('Read&Parse sourcemap:' + sourcemap_filepath + 'failed. ' + error.toString());
                        process.exit(0);
                    }
                    const consumer = await new SourceMapConsumer(sourcemap);
                    !sourcemap_map.has(name) && sourcemap_map.set(name, consumer);
                }
            }
        }
    }
}

const program = new commander.Command();
program.version('0.0.1', '-v, --version');
program.option('-s, --stack &lt;string&gt;', 'stack string which can obtain from JSON.stringfy(Error.stack)', stackStringProcess);
program.option('-i, --msg &lt;string&gt;', 'error message. e.g. Uncaught ReferenceError: a is not defined');
program.option('-m, --map &lt;string&gt;', 'sourcemap dir path. Where to find sourcemap');
program.parse(process.argv);

if (program.stack && program.msg && program.map) {
    let error_obj: Error = {
        'stack': program.stack,
        'message': program.msg,
        'name': program.msg.split(':')[0]
    }
    let stack_frame_array: ErrorStackParser.StackFrame[] = [];
    try {
        stack_frame_array = ErrorStackParser.parse(error_obj)
    } catch (error) {
        console.error('ErrorStackParser parsing failed' + error.toString());
        process.exit(0);
    }

    const sourcemap_map = new Map&lt;string, SourceMapConsumer&gt;();

    // load all sourcemap files
    loadAllConsumer(program.map, stack_frame_array, sourcemap_map).then(() => {
        // parse stack_frame_array
        stack_frame_array.forEach(stack_frame => {
            let name = stack_frame.fileName;
            if (sourcemap_map.has(name)) {
                let consumer = sourcemap_map.get(name);
                let origin = consumer.originalPositionFor({
                    line: stack_frame.lineNumber,
                    column: stack_frame.columnNumber
                });
                if (origin.line) stack_frame.lineNumber = origin.line;
                if (origin.column) stack_frame.columnNumber = origin.column;
                if (origin.source) stack_frame.fileName = origin.source;
                if (origin.name) stack_frame.functionName = origin.name;
            }
        });

        // print result
        printToConsole(program.msg, stack_frame_array);
    });

    // destroy all consumers
    for (let consumer of Array.from(sourcemap_map.values())) {
        consumer.destroy();
    }
} else {
    console.error("No error stack string OR error msg string OR sourcemap dir found. Please Check input.");
}
</code></pre>
