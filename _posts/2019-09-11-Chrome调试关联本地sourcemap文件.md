---
layout: post
title: 'Chrome调试关联本地sourcemap文件'
subtitle: '针对线上混淆代码与堆栈的调试方法'
description: '使用Chrome dev-tool查看混淆代码报错堆栈时关联本地sourcemap文件对其进行解析，映射源码'
date: 2019-09-11
author: zpo
categories: H5
tags: Chrome 浏览器
---

## Why

外网发布环境都是被混淆的JavaScript代码，因为发布环境下不会保存或关联sourcemap(否则混淆没有任何意义了)，所以日志与报错的堆栈都指向混淆后的代码。开发人员持有sourcemap文件，除了有本地解析映射的需求外，有时也存在直接调试外网混淆代码的需求，这时候就可以关联上本地的sourcemap文件，对线上的混淆代码进行映射和解析，直观反映成源码，可以通过Chrome自带功能解决该需求。

## 准备

- 本地有对应线上混淆JavaScript文件的sourcemap文件

## 方法

1. F12打开Chrome - dev-tool - Sources  
![步骤1.png](https://i.loli.net/2019/09/10/TIt6r3jyeOfMbQB.png)

2. 选中被混淆的代码，或者通过控制台的链接打开被混淆的代码  
![步骤2.png](https://i.loli.net/2019/09/10/BQN51ou4vdqnteI.png)

3. 右键 - Add source map  
![步骤3.png](https://i.loli.net/2019/09/10/BqWGS4PQCbfw5Xc.png)

4. 通过file协议选择本地的map文件,先在浏览器地址栏中输入确保可以访问到。  
e.g. file:///D:/H5Research/CatchJSError/logline.min.js.map
![步骤4.png](https://i.loli.net/2019/09/10/sYdumbNcy2XJ5Qa.png)

5. 添加后Sources左侧栏出现源码  
![步骤5.png](https://i.loli.net/2019/09/10/XAC2MaUj9yqlvKt.png)

6. 回到控制台发现已经映射到源码了  
![步骤6.png](https://i.loli.net/2019/09/10/C9kYeTNK83L6ZUd.png)
