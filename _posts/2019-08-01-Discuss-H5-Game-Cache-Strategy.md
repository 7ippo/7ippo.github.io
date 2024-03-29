---
layout: post
title: 'H5游戏缓存策略的讨论'
subtitle: 'H5游戏关于缓存策略标准的讨论'
date: 2019-08-02
author: 周子博
categories: H5
cover: 'https://i.loli.net/2019/08/06/WegjI3rcV9qt4wd.jpg'
tags: H5
---

<h2>缓存策略的几个思考点</h2>
<p>对于H5游戏，缓存的使用主要考虑几点：</p>
<ol>
<li>热更新</li>
<li>在确保缓存命中率的情况下，合理控制缓存大小</li>
<li>尽可能减少网络请求，减轻服务器压力</li>
</ol>
<h2>浏览器的缓存控制的几种方式及其利弊</h2>
<h3>版本控制</h3>
<p><strong>优势</strong>：</p>
<p>通过<strong>版本控制</strong>，通过gulp工具添加md5后缀改资源文件名，可以实现热更。当资源发生更新，通过版本管理工具同步更新引用该资源的位置，指向新的文件。请求的url发生变化，浏览器便不会读取缓存。假如再通过<code>Cache-Control:max-age</code>或<code>Expire</code>设置一个很长的缓存有效期，比如1年，那么在没有发布更新时，缓存的使用率是100%，且不会发生网络请求。</p>
<p><strong>弊端</strong>：</p>
<p>当大文件采取版本管理频繁更新时，会出现缓存大小膨胀的情况。e.g.一份压缩后2.3M的主逻辑js文件<code>bundle-md5.js</code>，如果发布10次更新，每次更新生成不同的文件名。那么如果用户恰好在每次更新后打开游戏，便会在浏览器内存中缓存11份不同md5后缀的该文件，直接占用25.3M的缓存空间，这还只是更新内容的一小部分。缓存大小随更新发布的急速膨胀也会导致触发某些浏览器环境如小程序中的限制：缓存大小达到一定标准(<a href="https://developers.weixin.qq.com/minigame/dev/guide/base-ability/file-system.html">微信小游戏50MB限制</a>)，后续请求的结果将无法写入缓存。</p>
<p><img src="https://i.loli.net/2019/08/01/5d429eab611c431148.png" alt="Chrome中的多份同文件缓存" width="500" data-action="zoom"/><br />
<em>*使用<code>ChromeCacheViewer</code>可以看到存在多份同文件不同后缀名的缓存*</em></p>
<h3><code>Cache-Control:no-cache</code></h3>
<p><strong>优势</strong>：</p>
<p>通过在响应头添加<code>Cache-Control:no-cache</code>实测在主流浏览器环境中都可以实现热更新。每次浏览器请求资源前都会携带缓存文件的缓存指示去找服务器验证，如果资源没更新，服务器返回304，浏览器就从本地缓存中读取。如果资源有更新，服务器返回最新资源，浏览器读取并且更新本地缓存。可见不管更新几次，本地缓存中有且仅有一份资源，解决了版本管理的弊端。</p>
<blockquote>
<p>注意，ETag作为缓存指示正在被大型网站弃用：主要原因是大部分CDN服务器开启了gzip压缩，而同一份文件(主要是css/js，图片压缩率不高一般不会压缩)在不同配置下执行gzip压缩后无法保证其唯一性，这会导致ETag失效，所以<a href="https://help.aliyun.com/knowledge_detail/40071.html">nginx官方在开启gzip模块后会移除ETag</a>。另外如果可以忽略在1秒内多次更新的情况，使用Last-Modified可以实现同样的功能，还能节约服务器计算资源。IETF没有规定ETag的计算方法，具体实现由不同服务器(Apache Nginx Tengine)自己决定。所以ETag是否会因为负载均衡策略导致同一份资源在不同服务器上计算结果不同我们不得而知，为了不浪费带宽资源，还是建议<strong>关闭ETag</strong>。</p>
</blockquote>
<p><img src="https://i.loli.net/2019/08/01/5d429eaab482f33401.png" alt="gzip与ETag" width="500" data-action="zoom"/><br />
<em>*从缓存文件中可以看到内容编码是gzip便不会有ETag指示*</em></p>
<p><strong>弊端</strong>：</p>
<p>直接走协商缓存，跳过了强缓存命中的阶段，每次请求资源时都会发Validation请求给服务器校验资源是否最新。对于不频繁更新的一些资源来说，服务器频繁建立TCP连接的开销应该纳入考量，这样消耗服务器带宽和计算资源是否有必要还需要讨论。如果可以接受每请求一个资源都发起一个网络请求，那所有资源都使用该策略即可。</p>
<h3><code>Cache-Control:max-age</code></h3>
<p>按照RFC的解释，在响应头添加max-age指示，意味着响应的内容从生成之时起(服务器时间)，在经过max-age指定的时间后才失效。也就是说在浏览器先走强缓存，判断缓存的Age是否有效。有效时间内的缓存命中则不会发起网络请求，超出有效时间即缓存陈旧(stale)则需要发起Validation请求验证资源是否有更新。</p>
<p><strong>优势</strong>：</p>
<p>浏览器只会保存一份缓存副本，且在缓存有效期内再次请求资源时不会发起HTTP请求，服务器压力最小。缓存失效后又会通过Validation请求校验实现更新资源。</p>
<p><strong>弊端</strong>：</p>
<p>缓存有效时间内，无法实现热更新，所以一般不会单独采用这个策略，而是结合版本控制一起做。判断缓存是否有效这个阶段属于浏览器的执行逻辑，除了外加别的方案比如代码检测到更新强制获取最新文件外没法改动。</p>
<h2>方案一 较复杂的细分方案及其标准</h2>
<p>为了能满足我们之前提的三点需求，我们先将游戏资源文件按照<strong>修改频次</strong>、<strong>文件大小<strong>、</strong>有无强热更要求</strong>三个方面进行细分，再去讨论对应策略。</p>
<ul>
<li>高频修改的定义：每<strong>3</strong>次更新中会更改到的文件属于高频修改文件，否则属于低频修改文件；</li>
<li>大文件的定义：单个文件超过<strong>1MB</strong>大小属于大文件，否则属于小文件。</li>
</ul>
<blockquote>
<p>注意：对于css/js/json文件通常开启gzip编码压缩后进行HTTP传输，衡量时1MB标准线应适用于压缩后大小，对于png/jpg等不压缩的资源则为原大小。<em>1MB标准线的划分参考于本地25751条Chrome缓存文件，只有15条缓存文件大小超过1MB。另外标准划分不同项目有不同需求，还需要后续调整。</em></p>
</blockquote>
<h3>高频大/小文件</h3>
<ul>
<li>即便是高频修改的小文件，若使用版本管理修改文件名，更新次数多了也可能浪费不少缓存空间。为避免缓存大小膨胀，必须保证浏览器对一个资源多个版本只保留一份副本。所以不使用版本管理来控制，不改文件名。建议针对这类资源使用响应头添加<code>Cache-Control:no-cache</code>策略。</li>
<li>相关文件类型：主逻辑js、入口html文件，入口js，version.json这类映射文件</li>
</ul>
<h3>低频大/小文件</h3>
<ul>
<li>因为更新频次低，建议使用版本管理。<code>Cache-Control:max-age</code>策略设置缓存有效期为<strong>半年</strong>。因为使用了版本管理，缓存有效期已经没有了意义，设置久一点减少网络请求。这意味着顶多半年后，才会对这个资源发起一次Validation请求，这对于一个H5游戏的生命周期来说已经足够长了。这样一来极大的减少了网络请求数量。</li>
</ul>
<blockquote>
<p>为什么<code>max-age</code>不设置更久? 半年比于一年对服务器来说每年一个资源就多一次请求差别不大，设置什么都可以。<br />
设置短一点，缓存过期之后会被清理吗? 不会。除非主动去清理，浏览器和微信小程序都不会自动清理过期的缓存文件。对于<a href="https://developers.weixin.qq.com/miniprogram/dev/framework/ability/file-system.html">微信小程序</a>而言，缓存文件会随代码包被清理而一块删除。而代码包在用户长期不打开的情况下会被微信清理，以及用户手动删除小程序时被清理。</p>
</blockquote>
<ul>
<li>相关文件类型：字体资源ttf(这个缓存文件为什么占8MB这么大)</li>
</ul>
<blockquote>
<p>除此以外，低频可以根据更新的可能性再做细分：普通低频(偶尔修改)与超低频文件(没有特殊情况不会修改)。普通低频文件可以设置max-age为<strong>1个月</strong>，超低频文件设置为<strong>一年</strong>。</p>
</blockquote>
<h3>有强热更要求的大文件</h3>
<ul>
<li>为避免缓存大小膨胀，不使用版本管理修改文件名，同时使用<code>Cache-Control:no-cache</code>策略</li>
<li>相关文件类型：loading<em>bg.png login</em>bg.png logo.png等大场景图、大图集与logo/icon这类可能发生版权问题要替换的资源</li>
</ul>
<h3>有强热更要求的小文件</h3>
<ul>
<li>相关文件类型：重要但是没有特殊情况不会修改的文件，比如做数据统计的js脚本。策略同低频文件。</li>
</ul>
<h2>方案二 代码控制更新方案</h2>
<p>如果要考虑控制缓存大小，就不推荐用版本管理。但是又同时要求热更新，使用<code>no-cache</code>响应头的话又没法减少过多的网络请求数量。浏览器缓存策略的选择其实是相互矛盾、需要取舍的。换一个角度的话，我们可以弃用版本管理，不改文件名，所有文件通过max-age设置一个较长的缓存有效时间比如半年。然后每次游戏打开时：代码控制发送一个网络请求，携带当前游戏的版本号。根据版本号生成一个差异文件清单返回给客户端(类似<a href="https://developer.mozilla.org/zh-CN/docs/Web/HTML/Using_the_application_cache">H5应用缓存机制</a>，虽然该标准已经被废弃)，然后再通过<code>XMLHttpRequest</code>加上强制刷新的请求头<code>Cache-Control:no-cache</code>与<code>Pragma:no-cache</code>提前获取差异文件列表中的最新资源，下载下来以更新本地的旧缓存。那么在缓存有效期内，真正请求该资源时，会直接读取缓存中的内容，不会再发生网络请求。</p>
<p>该方案的优势是：</p>
<ol>
<li>缓存大小最优化，每个文件本地只会存在一份缓存副本</li>
<li>解决热更问题的同时最小化网络请求的数量</li>
</ol>
<p>劣势是这样提前下载更新缓存，某些更新了的资源如果玩家没有用到，也下载了下来，没有做到按需加载。</p>
<blockquote>
<p>如果把差异文件列表存储起来，重写Laya的网络请求类，每次网络请求前去判断当前请求的资源是否属于差异文件(即是否是旧资源)，如果是则在请求头中添加强制刷新字段获取最新资源，如果不是则使用本地缓存。这样就可以做到按需加载，也能保持该方案的优势。</p>
</blockquote>
<h2>微信小程序缓存大小限制50MB环境下的缓存方案</h2>
<p>无论使用上面的哪一种方案，缓存大小在几次更新后必定会超出50MB，触发微信小游戏的缓存限制条件，这也要求面向小程序环境的项目还需要做一些特殊处理。</p>
<ol>
<li>如果不做特殊缓存策略，那么在小程序环境中会自动缓存前期网络请求的50MB资源文件到本地(一些文件类型不会自动缓存)，超过50MB后会抛出写入缓存失败的事件，每个没有被缓存的资源都要走网络请求。</li>
<li>如果使用<a href="https://ldc2.layabox.com/doc/?nav=zh-ts-5-0-5">Laya的自动缓存管理策略</a>，超过50MB后每次按资源时间清理最早的一部分文件,清理的空间大小可以配置。但是早期的资源被清理后，下次又要重新请求，逻辑上不太正确。</li>
<li>Laya允许关闭自动缓存策略，通过Laya封装的微信Adapter调用下载和清理方法，手动去下载文件缓存到本地,手动清理控制缓存在50MB以内。对此可以每个项目维护一个缓存文件清单，把重要的大文件列入，缓存在客户端本地，再结合以上的缓存控制方案保证缓存文件的新鲜度。</li>
</ol>
<h2>参考链接</h2>
<ul>
<li><a href="https://www.jb51.net/article/132710.htm">关于HTTP传输中gzip压缩的分析 - 脚本之家</a></li>
<li><a href="https://www.rfc-editor.org/rfc/pdfrfc/rfc7234.txt.pdf">Hypertext Transfer Protocol (HTTP/1.1): Caching - RFC</a></li>
<li><a href="http://www.nirsoft.net/utils/chrome_cache_view.html">ChromeCacheViewer - Chrome缓存分析工具</a></li>
<li><a href="https://developers.weixin.qq.com/miniprogram/dev/framework/ability/file-system.html">文件系统 - 微信小程序官方文档</a></li>
<li><a href="https://ldc2.layabox.com/doc/?nav=zh-ts-5-0-5">微信小游戏的50M物理缓存管理 - Laya</a></li>
</ul>
