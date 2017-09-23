# 原来 CSS 与 JS 是这样阻塞 DOM 解析和渲染的

hello~各位亲爱的看官老爷们大家好。估计大家都听过，尽量将`CSS`放头部，`JS`放底部，这样可以提高页面的性能。然而，为什么呢？大家有考虑过么？很长一段时间，我都是知其然而不知其所以然，强行背下来应付考核当然可以，但实际应用中必然一塌糊涂。因此洗（wang）心（yang）革（bu）面（lao），小结一下最近玩出来的成果。

友情提示，本文也是小白向为主，如果直接想看结论可以拉到最下面看的~

***

由于关系到文件的读取，那是肯定需要服务器的，我会把全部的文件放在[github](https://github.com/ljf0113/how-js-and-css-block-dom)上，给我点个 **star** 我会开心！掘金上再给我点个 **赞** 我就更开心了~

`node`端唯一需要解释一下的是这个函数：

    function sleep(time) {
      return new Promise(function(res) {
        setTimeout(() => {
          res()
        }, time);
      })
    }

嗯！其实就延时啦。如果`CSS`或者`JS`文件名有`sleep3000`之类的前缀时，意思就是延迟3000毫秒才会返回这文件。

下文使用的`HTML`文件是长这样的：

```
    <!DOCTYPE html>
    <html lang="en">
    <head>
    	<meta charset="UTF-8">
    	<title>Title</title>
    	<style>
    		div {
    			width: 100px;
    			height: 100px;
    			background: lightgreen;
    		}
    	</style>
    </head>
    <body>
    	<div></div>
    </body>
    </html>
```

我会在其中插入不同的`JS`和`CSS`。

而使用的`common.css`，不论有没有前缀，内容都是这样的：

    div {
      background: red;
    }

好了，话不多数，开始正文！

## CSS

关于`CSS`，大家肯定都知道的是`<link>`标签放在头部性能会高一点，少一点人知道如果`<script>`与`<link>`同时在头部的话，`<script>`在上可能会更好。这是为什么呢？下面我们一起来看一下`CSS`对`DOM`的影响是什么。

###  `CSS` 不会阻塞 `DOM` 的解析

注意哦！这里说的是`DOM` 解析，证明的例子如下，首先在头部插入`<script defer src="/js/logDiv.js"></script>`，`JS`文件的内容是：
    
    const div = document.querySelecot('div');
    console.log(div);

`defer`属性相信大家也很熟悉了，[MDN](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/script)对此的描述是用来通知浏览器该脚本将在文档完成解析后，触发 DOMContentLoaded 事件前执行。设置这个属性，能保证`DOM`解析后马上打印出`div`。

之后将`<link rel="stylesheet" href="/css/sleep3000-common.css">`插入`HTML`文件的任一位置，打开浏览器，可以看到是首先打印出`div`这个`DOM`节点，过3s左右之后才渲染出一个浅蓝色的`div`。这就证明了`CSS` 是不会阻塞 `DOM` 的解析的，尽管`CSS`下载需要3s，但这个过程中，浏览器不会傻等着`CSS`下载完，而是会解析`DOM`的。

这里简单说一下，浏览器是解析`DOM`生成`DOM Tree`，结合`CSS`生成的`CSS Tree`，最终组成`render tree`，再渲染页面。由此可见，在此过程中`CSS`完全无法影响`DOM Tree`，因而无需阻塞`DOM`解析。然而，`DOM Tree`和`CSS Tree`会组合成`render tree`，那`CSS`会不会页面阻塞渲染呢？

###  `CSS` 阻塞页面渲染

其实这一点，刚才的例子已经说明了，如果`CSS` 不会阻塞页面阻塞渲染，那么`CSS`文件下载之前，浏览器就会渲染出一个浅绿色的`div`，之后再变成浅蓝色。浏览器的这个策略其实很明智的，想象一下，如果没有这个策略，页面首先会呈现出一个原始的模样，待`CSS`下载完之后又突然变了一个模样。用户体验可谓极差，而且渲染是有成本的。

因此，基于性能与用户体验的考虑，浏览器会尽量减少渲染的次数，`CSS`顺理成章地阻塞页面渲染。

然而，事情总有奇怪的，请看这例子，`HTML`头部结构如下：
    
    <header>
        <link rel="stylesheet" href="/css/sleep3000-common.css">
	    <script src="/js/logDiv.js"></script>
    </header>
	
但思考一下这会产生什么结果呢？

答案是浏览器会转圈圈三秒，但此过程中不会打印任何东西，之后呈现出一个浅蓝色的`div`，再打印出`null`。结果好像是`CSS`不单阻塞了页面渲染，还阻塞了`DOM` 的解析啊！稍等，在你打算掀桌子疯狂吐槽我之前，请先思考一下是什么阻塞了`DOM` 的解析，刚才已经证明了`CSS`是不会阻塞的，那么阻塞了页面解析其实是`JS`！但明明`JS`的代码如此简单，肯定不会阻塞这么久，那就是`JS`在等待`CSS`的下载，这是为什么呢？

仔细思考一下，其实这样做是有道理的，如果脚本的内容是获取元素的样式，宽高等`CSS`控制的属性，浏览器是需要计算的，也就是依赖于`CSS`。浏览器也无法感知脚本内容到底是什么，为避免样式获取，因而只好等前面所有的样式下载完后，再执行`JS`。因而造成了之前例子的情况。

所以，看官大人明白为何`<script>`与`<link>`同时在头部的话，`<script>`在上可能会更好了么？之所以是可能，是因为如果`<link>`的内容下载更快的话，是没影响的，但反过来的话，`JS`就要等待了，然而这些等待的时间是完全不必要的。

## JS

`JS`，也就是`<script>`标签，估计大家都很熟悉了，不就是阻塞`DOM`解析和渲染么。然而，其中其实还是有一点细节可以考究一下的，我们一起来好好看看。

### `JS` 阻塞 `DOM` 解析

首先我们需要一个新的`JS`文件名为`blok.js`，内容如下：

    const arr = [];
    for (let i = 0; i < 10000000; i++) {
      arr.push(i);
      arr.splice(i % 3, i % 7, i % 5);
    }
    const div = document.querySelector('div');
    console.log(div);
    
其实那个数组操作时没意义的，只是为了让这个`JS`文件多花执行时间而已。之后把这个文件插入头部，浏览器跑一下。

结果估计大家也能想象得到，浏览器转圈圈一会，这过程中不会有任何东西出现。之后打印出`null`，再出现一个浅绿色的`div`。现象就足以说明`JS` 阻塞 `DOM` 解析了。其实原因也很好理解，浏览器并不知道脚本的内容是什么，如果先行解析下面的`DOM`，万一脚本内全删了后面的`DOM`，浏览器就白干活了。更别谈丧心病狂的`document.write`。浏览器无法预估里面的内容，那就干脆全部停住，等脚本执行完再干活就好了。

对此的优化其实也很显而易见，具体分为两类。如果`JS`文件体积太大，同时你确定没必要阻塞`DOM`解析的话，不妨按需要加上`defer`或者`async`属性，此时脚本下载的过程中是不会阻塞`DOM`解析的。

而如果是文件执行时间太长，不妨分拆一下代码，不用立即执行的代码，可以使用一下以前的黑科技：`setTimeout()`。当然，现代的浏览器很聪明，它会“偷看”之后的`DOM`内容，碰到如`<link>`、`<script>`和`<img>`等标签时，它会帮助我们先行下载里面的资源，不会傻等到解析到那里时才下载。

### 浏览器遇到 `<script>` 标签时，会触发页面渲染

这个细节可能不少看官大人并不清楚，其实这才是解释上面为何`JS`执行会等待`CSS`下载的原因。先上例子,`HTML`内`body`的结构如下：
    
    <body>
    	<div></div>
    	<script src="/js/sleep3000-logDiv.js"></script>
    	<style>
    		div {
    			background: lightgrey;
    		}
    	</style>
    	<script src="/js/sleep5000-logDiv.js"></script>
    	<link rel="stylesheet" href="/css/common.css">
    </body>
    
这个例子也是很极端的例子，但不妨碍它透露给我们很多重要的信息。想象一下，页面会怎样呢？

答案是先浅绿色，再浅灰色，最后浅蓝色。由此可见，每次碰到`<script>`标签时，浏览器都会渲染一次页面。这是基于同样的理由，浏览器不知道脚本的内容，因而碰到脚本时，只好先渲染页面，确保脚本能获取到最新的`DOM`元素信息，尽管脚本可能不需要这些信息。

## 小结

综上所述，我们得出这样的结论：

* `CSS` 不会阻塞 `DOM` 的解析，但会阻塞 `DOM` 渲染。
* `JS` 阻塞 `DOM` 解析，但浏览器会"偷看"`DOM`，预先下载相关资源。
* 浏览器遇到 `<script>`且没有`defer`或`async`属性的 标签时，会触发页面渲染，因而如果前面`CSS`资源尚未加载完毕时，浏览器会等待它加载完毕在执行脚本。

所以，你现在明白为何`<script>`最好放底部，`<link>`最好放头部，如果头部同时有`<script>`与`<link>`的情况下，最好将`<script>`放在`<link>`上面了吗？

感谢各位看官大人看到这里，希望本文对你有所帮助，有不同或更好意见的大佬，还望不吝赐教！谢谢~