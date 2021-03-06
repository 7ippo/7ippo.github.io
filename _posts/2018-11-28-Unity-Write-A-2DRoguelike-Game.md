---
layout: post
title: 'Unity入门:编写一个2D Roguelike游戏'
subtitle: '我的第一个Unity项目'
date: 2018-11-28
author: 周子博
categories: 技术
tags: Unity C#
---

<p>跑完了Unity官方的Basic Tutorial Demo后，意犹未尽的下载了一个Tutorial Project Demo:2D Roguelike。跟着官方的<a href="https://unity3d.com/cn/learn/tutorials/s/2d-roguelike-tutorial">教程</a>从头开始做了一个Roguelike类型的2D生存游戏，每一关随机生成地图，食物和敌人。玩家要尽可能到达更高的关数。每天晚上跟着码代码和理解，花了两天时间终于build出来了windows版本的游戏，自己玩了玩还挺有意思，虽然有一个碰撞的bug...不过也算是上手了Unity的编辑器，对游戏这个概念有一个从代码到动画表现上的多层次理解。<br />
<img src="https://i.loli.net/2019/05/25/5ce905c5492e939016.png" alt="Windows Build" data-action="zoom"/></p>
<h2>对游戏以及Unity一些浅薄的理解</h2>
<p>对这个游戏以及Unity有了一些浅薄的理解:  
</p>
<ol>
<li><p>我们看到的游戏中的物体都被看做是一个对象，<strong>GameObject</strong>。</p>
</li>
<li><p>对象的形态外貌（<strong>Sprite Render</strong>）由我们赋予，比如人物场景等任何元素的样子。外形是对象的一个属性，我们可以赋值成一些贴图。</p>
</li>
<li>
<p>对象的动画由一帧一帧的图片组成，动画（<strong>Animator</strong>）也是对象的一个可以被添加的属性，可以被添加移除以达到复用效果的属性更应该被称为组件（<strong>Component</strong>）。我们可以设置动画播放的快慢，是否连续播放，是否不可被打断等动画效果，也可以通过代码控制动画播放的时机。</p>
<blockquote>
<p>关于动画有一个有意思的事情，就是动画有限状态机。其负责播放动画，以及在不同动画状态间来回切换。<br />
<img src="https://i.loli.net/2019/05/25/5ce905c55bbff23067.png" alt="AnimationController" data-action="zoom"/><br />
比如Demo中的主角，当玩家没有指令时，为状态一：这个状态下动画控制器（Animator Controller）就负责循环播放“PlayerIdle”（待机）这个动画，表现是主角一直在“呼吸抖动”<br />
<img src="https://i.loli.net/2019/05/25/5ce905c55cc8646305.png" alt="AnimationController2" data-action="zoom"/><br />
当触发被攻击或者破坏墙壁的这两件事件时，代码通过Animator  Controller切换动画状态。被攻击事件时切换到“PlayerChop”状态，预先设定是进入这个状态后播放一次然后退出，所以主角动画表现是被攻击晃了一下，然后又回到“PlayerIdle”这个状态，循环播放待机动画。<br />
也就是说，动画控制器负责播放动画（包括播放几次，速度快慢，几个动画状态），代码控制怎么在动画状态之间切换，何时进入，何时退出。</p>
</blockquote>
</li>
<li>
<p>对象组件Component的理解：为了让我们的对象具有某些特性，我们还需要添加一些Component组件，其实际上是让这个对象拥有多一些其他类所拥有的方法和属性。比如这个Demo中，我们把主角和敌人视作可以检测碰撞的刚体，所以都加了Box Collider 2D和Rigidbody 2D两个组件。在控制这两种对象时，我们就可以使用这两种类中的方法来检测碰撞，然后扣取主角的生命值。墙体只检测碰撞来阻止敌人和主角移动，所以都添加Box Collider 2D组件。至于对象之间的互动，何时互动，我们都是通过C#代码来操控的。</p>
</li>
<li><p>地图大小固定是8x8的方块，因为是Roguelike的游戏，墙（可以被主角攻击打碎），敌人，地面上的水果食物都是根据关数在一个范围内随机生成（UnityEngine的Random类），在8x8地图上随机摆放，下一关入口则固定在右上方，主角每一关一开始都出现在左下角。<br />
<img src="https://i.loli.net/2019/05/25/5ce905c57b0dc60751.png" alt="Game" data-action="zoom"/></p>
</li>
<li><p>这个demo还涉及一些简单的UI和音频，分别通过Text类和AudioSource类来实现，通过脚本设置内容，在移动或被攻击的时候去更新内容，字体和颜色等属性可以在Unity中通过Text Component来改，也可以通过脚本GetComponent来修改。</p>   
</li>
</ol>
<h2>游戏设计需要面向对象</h2>
<p>设计这些游戏对象的方法，我们肯定是采用面向对象的编程方法，极大的节省了代码量和维护成本，更好理解对象之间的行为方式。对于这个很简单的Roguelike游戏，我们可以设计这么几个类：</p>
<ol>
<li>GameManager类：用于管理整个游戏基本的初始化，随机地图生成，以及游戏基本逻辑（每个回合敌人都会移动），还有过场动画，角色死亡动画等;</li>
<li>MovingObject类：考虑到主角和敌人都会移动，可以设计这样一个类，用于解决对象的移动表现（从一个地块到相邻的地块其实动画上可以有很多表现形式，具体怎么表现需要代码决定），是否可以移动（是否遇到障碍了）以及处理对象之间的碰撞关系（设计一个抽象类，由具体类去实现）;</li>
<li>Player类：继承MovingObject类，处理详细的逻辑，比如向墙壁、食物、敌人移动分别是什么表现，还得检查食物是否小于0（死亡）;</li>
<li>Enemy类：有点像写AI的移动逻辑了，每个回合什么表现，遇到主角时播放攻击动画等都由这个类控制;</li>
<li>BoardManager类：地图场景的管理类，具体怎么生成随机地图，怎么生成墙体、食物、敌人，随机位置还不可以重复;</li>
<li>Wall类：除了周围的碎石，场景中间的石块也是有表现的，可以被击碎，所以也要写一个具体的类来描述这一类对象;</li>
<li>SoundManager类：管理背景音乐的循环播放。  
</li>
</ol>
<p>采用面向对象的编程方法，设计由顶向下，对整体的概念就会非常清晰，可以一点一点实现细节，也不会出现后期发现设计有问题，要推倒重来的情况。如果你思维很健壮，能考虑很多情况和可能性，还可以避免很多bug，亦或者是使很多bug更容易理解。比如这个例子中，只有主角和敌人可以移动，那就设计一个MovingObject的类，把二者通用的属性和方法都写进去，然后单独设计主角和敌人时，只需要继承这个类，实现独有的逻辑即可。  
</p>
<h2>也只是刚刚入门</h2>
<p>经过了这么一个demo，基本有了对小型简单游戏的设计理解：</p>
<ol>
<li>有了一个比较好的idea后，能描述出游戏的雏形，核心玩法是什么，特色是什么，实现机制是什么？比如Roguelike，就是地牢生存，特点是随机事件；</li>
<li>思考如何设计游戏中的对象，和对象间的互动；</li>
<li>在纸上画出游戏的各个元素，动画，原型，音效，用代码合理的组织起来；</li>
<li>通过代码实现具体表现。</li>
</ol>
