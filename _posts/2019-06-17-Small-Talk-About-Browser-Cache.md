---
layout: post
title: '浅谈浏览器缓存机制与H5游戏缓存策略'
subtitle: '不同浏览器环境的抓包验证'
date: 2019-06-21
author: 周子博
categories: H5
cover: 'https://i.loli.net/2019/08/02/5d44240213bbc70794.jpg'
tags: H5
---

<h2>缓存类型</h2>
<p>HTTP缓存有很多种，广义上分为共享缓存(public)和私有缓存(private)。共享缓存包括代理缓存、网关缓存、CDN、反向代理缓存和负载均衡器等，本文讨论的是私有缓存：浏览器缓存。从H5游戏开发的角度看，如何在解决不同浏览器环境下缓存带来的热更新问题的同时，最大化利用缓存是追求的目标，为此需要了解缓存机制，制定一个合适的缓存策略并验证分析其正确性。</p>
<h2>先了解浏览器的缓存逻辑</h2>
<h3>1. 第一次请求</h3>
<p>第一次请求，因为浏览器没有缓存该内容，所以缓存不会命中。向服务器请求资源返回的响应头中，会有缓存结果和缓存指示(directives)，浏览器将内容和缓存指示一同存入缓存中。</p>
<h3>2. 第二次请求同样的资源</h3>
<p>浏览器在缓存中找到该资源的缓存指示，根据指示选择之后的缓存策略。根据是否需要向服务器重新发起HTTP请求，我们可以将缓存过程分为两个部分，分别是强缓存和协商缓存。</p>
<h4>1)强缓存阶段</h4>
<p>不会向服务器发送请求，直接从缓存中读取资源，在chrome控制台的Network选项中可以看到该请求返回200的状态码，并且Size栏显示from disk cache(从硬盘缓存读取)或from memory cache(从内存缓存读取)。<br />
<img src="https://i.loli.net/2019/06/21/5d0c3f553ed4f29614.png" alt="图1，强缓存命中" data-action="zoom"/><br />
强缓存可以通过设置三种 HTTP Header 实现：<code>Expires</code>，<code>Pragma</code> 和 <code>Cache-Control</code>。</p>
<pre><code class="language-html">Expires: Wed, 19 Jun 2019 15:00:00 GMT</code></pre>
<blockquote>
<p><strong>Expires</strong> :<br />
<strong>HTTP/1.0</strong>的产物，属于HTTP响应的头部字段，值为HTTP Date类型，指定了所请求的资源到期的具体时间。浏览器在下一次请求时会判断本地时间，如果没有超过Expires设置的过期时间，则直接从缓存中读取，否则需要发起HTTP请求。但是需要说明的是，<code>Expires</code>已经被<code>Cache-Control</code>替代，如果响应头中存在<code>Cache-Control</code>控制缓存过期时间的字段，则<code>Expires</code>会被忽略。P.S. 搭建的HTTP服务器使用的HTTP/1.1协议，抓包没有看到响应中有<code>Expires</code>字段。</p>
</blockquote>
<pre><code class="language-html">Pragma:no-cache</code></pre>
<blockquote>
<p><strong>Pragma</strong> :<br />
<strong>HTTP/1.0</strong>的产物，属于HTTP请求与响应通用的头部字段，只有一个<code>no-cache</code>命令，行为与<code>Cache-Control: no-cache</code>一致，优先级低于<code>Cache-Control</code>。作为请求头时作用是强制要求缓存服务器在返回缓存的版本之前将请求提交到源头服务器进行验证，确保返回最新资源(强制刷新时浏览器会在请求头中加入<code>Pragma:no-cache</code>)。作为响应头时行为不统一，依赖浏览器具体实现，所以该字段不可靠。MDN建议只在需要兼容 <code>HTTP/1.0</code> 客户端的场合下应用 Pragma 首部。</p>
</blockquote>
<p><img src="https://i.loli.net/2019/06/21/5d0c9f23dfecd58526.png" alt="强刷新请求头中的Pragma字段与Cache-Control字段" data-action="zoom"/>  
</p>
<pre><code class="language-html">Cache-control: must-revalidate,no-cache,no-store,no-transform,public,private,proxy-revalidate,max-age=&lt;seconds&gt;,s-maxage=&lt;seconds&gt;</code></pre>
<blockquote>
<p><strong>Cache-Control</strong> :<br />
<strong>HTTP/1.1</strong>的产物，属于HTTP请求与响应通用的头部字段,命令可用于控制内容可缓存性、到期时间、是否重新验证和重新加载等。可通过组合命令实现具体的缓存策略。</p>
<p><code>no-cache</code>: 作为请求头时同Pragma字段行为一致，强制要求缓存服务器在返回缓存的版本之前将请求提交到源头服务器进行验证，确保返回最新资源，强制刷新请求会携带该字段。作为响应头时表现为客户端在使用缓存前需要发送请求验证缓存可用，即跳过强缓存，进行协商缓存。</p>
<p><code>max-age=&lt;seconds&gt;</code>: 设置缓存有效的最大周期，超过这个时间缓存被认为过期(单位秒)，需要重新向服务器请求资源。</p>
<p><code>must-revalidate</code> :  只能作为响应头字段，指示一旦资源过期（比如已经超过max-age），在成功向原始服务器验证之前，缓存服务器不能用该资源响应后续请求。</p>
<p><code>public/private</code>: <code>public</code>表明响应可以被任何对象（包括：发送请求的客户端，代理服务器，等等）缓存，<code>private</code>表明响应只能被用户缓存，不能作为共享缓存（即代理服务器不能缓存）。</p>
<p>其他字段含义可参考<a href="https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Cache-Control">MDN</a></p>
</blockquote>
<h4>2)协商缓存阶段</h4>
<p>协商缓存就是强制缓存失效后，浏览器携带缓存指示向服务器发起请求，由服务器根据缓存指示决定是否使用缓存的过程。协商缓存使用的缓存指示存在请求和响应的HTTP头部中，包括<code>响应头Last-Modified/请求头If-Modified-Since</code>和<code>响应头ETag/请求头If-None-Match</code>。协商缓存的结果有两种：</p>
<ol>
<li>协商缓存失效，服务器返回200和最新资源</li>
<li>协商缓存有效，服务器返回304 Not Modified，浏览器从缓存中读取对应资源</li>
</ol>
<pre><code class="language-html">Last-Modified: Wed, 19 Jun 2019 15:00:00 GMT
If-Modified-Since: Wed, 19 Jun 2019 15:00:00 GMT</code></pre>
<blockquote>
<p><strong>Last-Modified/If-Modified-Since</strong>:<br />
服务器会在响应头部添加<code>Last-Modified</code>字段，值为资源在服务器上的最后修改时间。浏览器将该缓存指示同缓存结果一同存入缓存中。浏览器请求资源时，如果存在<code>Last-Modified</code>字段，会把该值通过<code>If-Modified-Since</code>字段添加到请求头中。服务器再次收到这个资源请求，会根据<code>If-Modified-Since</code>中的值与服务器中该资源的最后修改时间对比。如果没有变化，返回304和空的响应体，浏览器直接从缓存读取。如果If-Modified-Since的时间小于服务器中这个资源的最后修改时间，说明文件有更新，于是返回新的资源文件和200。</p>
</blockquote>
<p><img src="https://i.loli.net/2019/06/21/5d0c40d1e448c35360.png" alt="Last-Modified Example" data-action="zoom"/></p>
<pre><code class="language-html">ETag: W/&quot;2251799814418149-5669-2019-06-18T12:14:19.152Z&quot;
If-None-Match: W/&quot;2251799814418149-5669-2019-06-18T12:14:19.152Z&quot;</code></pre>
<blockquote>
<p><strong>ETag/If-None-Match</strong>:<br />
<code>ETag</code>是服务器响应请求时，返回当前资源文件的一个唯一标识(由服务器生成)。只要资源有变化，<code>ETag</code>就会重新生成。服务器通过比较客户端传来的<code>If-None-Match</code>跟自己服务器上该资源的<code>ETag</code>是否一致来决定返回结果。如果不一致，GET返回200，新资源和新<code>ETag</code>，一致则返回304 Not Modified。</p>
</blockquote>
<h5>ETag与Last-Modified对比</h5>
<ol>
<li>精确度上<code>ETag</code>要优于<code>Last-Modified</code>。<code>Last-Modified</code>的时间单位是秒，如果某个文件在1秒内改变了多次，那么他们的<code>Last-Modified</code>其实并没有体现出来修改，但是<code>ETag</code>每次都会改变确保了精度；如果是负载均衡的服务器，各个服务器生成的<code>Last-Modified</code>也有可能不一致。</li>
<li>性能上ETag要逊于<code>Last-Modified</code>，毕竟<code>Last-Modified</code>只需要记录时间，而<code>ETag</code>需要服务器通过算法来计算出一个hash值。</li>
<li>优先级上服务器校验优先考虑<code>ETag</code></li>
</ol>
<h2>打开页面的方式影响浏览器缓存的使用逻辑</h2>
<ol>
<li>地址栏回车：浏览器先查看请求资源有无缓存，无缓存则向服务器请求资源。若该资源有缓存则根据缓存指示采用对应缓存策略。先查找强缓存，强缓存有效则直接使用缓存200(from cache)，强缓存无效则走协商缓存返回304 Not Modified或返回最新资源200</li>
<li>F5/ctrl+R：刷新页面。仅请求该页面，并跳过强缓存阶段，走协商缓存，所以会向服务器发送协商请求，返回304 Not Modified或返回最新资源200。注意：该页面所引用的资源不会跳过强缓存，且对Chrome而言，在页面通过地址栏回车的方式打开与页面相同URL的行为等同于刷新<br />
<img src="https://i.loli.net/2019/06/21/5d0c3ee62087918381.png" alt="图7：F5刷新逻辑" data-action="zoom"/></li>
<li>ctrl+F5/ctrl+shift+R：强制刷新页面及页面所引用的资源，跳过所有缓存阶段，无论如何都会向服务器发送请求，强制返回最新资源200<br />
<img src="https://i.loli.net/2019/06/21/5d0c3f0d7522c59021.png" alt="图8强刷" data-action="zoom"/></li>
</ol>
<h2>H5游戏应该选择怎样的缓存策略</h2>
<p><strong>游戏入口index.html确保不被缓存，游戏依赖的资源文件通过gulp构建工具做版本管理</strong>。</p>
<blockquote>
<p>每次更新文件内容时，更改文件名加上md5后缀。因为入口引用的资源名称变了，浏览器会直接请求新的资源。当没有变化时，又会直接读取缓存。这一步一些H5游戏引擎如laya已经实现了，通过修改laya的gulp脚本，可以自定义修改构建的效果。</p>
</blockquote>
<p>如何确保入口不被缓存？我们可以使用meta标签的<code>http-equiv</code>属性把content内容添加到HTTP响应的头部信息，但是默认的HTTP服务器(比如自己搭建的node http服务器)不会在响应时根据meta标签来处理头部信息。这种情况下需要通过<a href="https://nginx.org/en/docs/http/ngx_http_headers_module.html">nginx设置</a>显示地在请求index.html的响应头中加入这些字段。</p>
<pre><code class="language-html">&lt;!--meta标签--&gt;
&lt;meta http-equiv=&quot;Cache-Control&quot; content=&quot;no-cache,max-age=0,must-revalidate&quot;&gt;
&lt;meta http-equiv=&quot;Expires&quot; content=&quot;0&quot;&gt;  <!--兼容HTTP/1.0-->
&lt;meta http-equiv=&quot;Pragma&quot; content=&quot;no-cache&quot;&gt; <!--兼容HTTP/1.0-->
</code></pre>
<pre><code class="language-nginx">// nginx设置
add_header Pragma no-cache;
expires    epoch;

// 注:expires epoch语法等同于设置Expires字段为Thu, 01 Jan 1970 00:00:01 GMT以及Cache-Control字段为no-cache
</code></pre>
<p><img src="https://i.loli.net/2019/06/21/5d0c4766e27bf98921.png" alt="nginx设置后" data-action="zoom"/><br />
<em>*上图为nginx设置应用在<a href="http://develop.h5game.com/h5client1/web/index.html">内网H5RPG项目</a>的结果*</em>  
</p>
<h2>不同浏览器环境的表现验证</h2>
<p>缓存机制的逻辑最终还是由浏览器来实现，部分浏览器不一定严格遵守W3C的规范，为此需要验证不同浏览器环境下我们选择的缓存策略表现。我们选择<a href="http://develop.h5game.com/h5client1/web/index.html">内网H5RPG项目</a>来进行多种浏览器环境的测试。nginx设置如下:<br />
<img src="https://i.loli.net/2019/06/21/5d0c7b516a40c14526.jpg" alt="nginx设置" data-action="zoom"/></p>
<blockquote>
<p>验证方法：</p>
<p>PC开热点让手机连接，在不同的浏览器环境中测试我们使用的缓存策略，使用Wireshark抓包查看请求和响应。</p>
<ol>
<li>打开/允许使用缓存</li>
<li>第一次请求资源</li>
<li>第二次请求资源(<strong>不使用刷新</strong>)查看是否会向服务器发送请求(检查是否使用强缓存)，只要发送了请求，则使用的协商缓存，达到预期的效果，实现了热更新。</li>
</ol>
</blockquote>
<h3>Chrome</h3>
<p><strong>测试环境</strong>: <em>Chrome 74.0.3729.169(Release)(64 bit)</em><br />
<strong>是否有请求</strong>:<br />
<img src="https://i.loli.net/2019/06/21/5d0c7e8a6f56d95279.png" alt="Chrome第二次请求" data-action="zoom"/><br />
<strong>结论</strong>: 只有两个请求，分别是index.html以及version.json，并且使用协商缓存返回304，其余资源都是直接读取缓存。符合预期。</p>
<h3>Safari</h3>
<p><strong>测试环境</strong>: <em>iPhone 7 iOS 12.2</em><br />
<strong>是否有请求</strong>:<br />
<img src="https://i.loli.net/2019/06/21/5d0c812e6e33774720.png" alt="Safari第二次请求" data-action="zoom"/><br />
<strong>结论</strong>: 只有两个请求，分别是index.html以及version.json，并且使用协商缓存返回304，其余资源都是直接读取缓存。符合预期。</p>
<blockquote>
<p>使用iPhone Safari测试，需要关闭无痕模式</p>
</blockquote>
<h3>Wechat Browser</h3>
<p><strong>测试环境</strong>: <em>WeChat 7.0.4</em><br />
<strong>是否有请求</strong>:<br />
<img src="https://i.loli.net/2019/06/21/5d0c83aee434831989.png" alt="微信浏览器第二次请求" data-action="zoom"/>
<strong>结论</strong>: 同样只有两个请求，并且使用协商缓存返回304，其余资源都是读取缓存。符合预期。</p>
<h3>X5 Core(TBS) apk</h3>
<p><strong>测试环境</strong>: <em>TBS V4.5 43646</em><br />
<strong>是否有请求</strong>:<br />
<img src="https://i.loli.net/2019/06/21/5d0c9a3604ea782416.png" alt="X5内核默认设置不缓存" data-action="zoom"/><br />
<em>*demo默认设置不使用缓存，相当于强刷*</em>  
</p>
<p><img src="https://i.loli.net/2019/06/21/5d0c9a82afa7273901.png" alt="修改CacheMode后符合预期" data-action="zoom"/><br />
<em>*设置WebView CacheMode后使用协商缓存*</em>  
</p>
<p><strong>结论</strong>: X5 WebView默认情况下不使用缓存，所有GET请求都会带上no-cache请求头去获取最新资源，每次进入apk都相当于强制刷新。需要设置<code>CacheMode</code>，设置后正常走协商缓存，符合预期。</p>
<pre><code class="language-java">webSetting.setCacheMode(IX5WebSettings.DEFAULT_CACHE_CAPACITY);</code></pre>
<h2>最后制作一张流程图</h2>
<p><img src="https://i.loli.net/2019/06/21/5d0c9e04485d821258.png" alt="GET请求浏览器缓存流程图" data-action="zoom"/></p>
<h3>参考链接</h3>
<ul>
<li><a href="https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Caching_FAQ">HTTP 缓存 - MDN</a></li>
<li><a href="https://www.jianshu.com/p/54cc04190252">深入理解浏览器的缓存机制 - 简书</a></li>
<li><a href="https://www.rfc-editor.org/rfc/pdfrfc/rfc7234.txt.pdf">Hypertext Transfer Protocol (HTTP/1.1): Caching - RFC</a></li>
</ul>
