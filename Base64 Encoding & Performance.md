# [译]Base64 编码&性能

原文作者为[Harry](https://csswizardry.com)，分为两部分：[Part1](https://csswizardry.com/2017/02/base64-encoding-and-performance/) & [Part2](https://csswizardry.com/2017/02/base64-encoding-and-performance-part-2/)。

## Part1: Base64有什么用？

**减少请求数量**，是这几年的一个优秀性能建议。虽然如此，也不是说它就没有缺陷。为了使页面加载更快，我们实际上可以通过高效的传输静态资源来实现，而不只是减少几个请求。

其中一个从**减少请求数量**诞生并被推崇的实践是使用Base64编码：将外部资源（e.g. 图片）直接嵌入到使用它的文本（e.g.样式表）中。减少HTTP请求数量的关键是，所有资源（样式表或图片）能够在同一时间到达。听起来像做梦，是吧？

然而并不是。

不幸的是，使用Base64编码是一个反模式[注1]。我希望在这篇文章中去分享关于关键路径优化，Gzip，当然还有Base64的一些思考。


### 我们来看一些代码

我写这篇文章是因为我刚刚为客户做了一个[审计](https://csswizardry.com/code-reviews/)，遇到了下面要讨论到的问题。这是一个来自实际客户端的实际样式表：信息是匿名的，但这是一个完全真实的项目。

我在页面上运行了一个快速的网络配置文件，发现了一个样式表（某方面来说，这是件好事，因为我们是绝对不愿意看到有12个样式表请求的），但是这个样式表在解压缩之后居然有**925K**。实际请求到的字节少得多，但还是有232K。

当我们看到这么大体积的样式表时，开始感到恐慌了。我相当确定，甚至都不用去看，里面肯定有Base64。当然，并不是说它是唯一的原因（插件，缺乏结构，继承等等，都可能有影响），但这么大体积的样式表通常都是因为Base64。并且：

* 不管是不是因为Base64，925K的样式表都很恐怖
* 压缩也只能减少到759K
* Gzip压缩到232K，去除了693K相同的代码
* 232K的请求还是很恐怖

请注意，光是解析这么大的样式表就需要88ms。而把它交给网络只是我们烦恼的开始而已：

![screenshot](https://csswizardry.com/wp-content/uploads/2017/02/screenshot-parse-001.png)

我优化了文件[注2]，把它保存到我的机器，用[CSSO](https://github.com/css/csso)运行，然后用Gzip的常规设置执行缩小后的内容。看看我得到的数字：

	harryroberts in ~/Sites/<client>/review/code on (master)
	» csso base64.css base64.min.css
	
	harryroberts in ~/Sites/<client>/review/code on (master)
	» gzip -k base64.min.css
	
	harryroberts in ~/Sites/<client>/review/code on (master)
	» ls -lh
	total 3840
	-rw-r--r--  1 harryroberts  staff   925K 10 Feb 11:23 base64.css
	-rw-r--r--  1 harryroberts  staff   759K 10 Feb 11:24 base64.min.css
	-rw-r--r--  1 harryroberts  staff   232K 10 Feb 11:24 base64.min.css.gz


接下来要做的就是找出有多少字节是Base64资源。为了做这件事，我简单粗暴的删除了所有包含**data**:字符串（**:g/data:/d**[注3]，Vim用户阅读）的行和声明。这里面大部分是图片/雪碧图，小部分是字体。然后我将这个删除后的文件保存为**no-base64.css**，再执行相同的压缩和Gzip：

	harryroberts in ~/Sites/<client>/review/code on (master)
	» ls -lh
	total 2648
	-rw-r--r--  1 harryroberts  staff   708K 10 Feb 15:54 no-base64.css
	-rw-r--r--  1 harryroberts  staff   543K 10 Feb 15:54 no-base64.min.css
	-rw-r--r--  1 harryroberts  staff    68K 10 Feb 15:54 no-base64.min.css.gz

在还没有压缩之前，我们已经减少了217K的Base64内容。但还是很大（708K），不过我们已经成功移除了23.45%的Base64代码。

在我们使用Gzip之后，相当惊喜。我们把708K降到了68K。整整**减少了90.39%**。

### Gzip保存...

Gzip真的是太让人难以置信了！ 它可能是世界上用于保护用户免受开发者祸害的最好工具了。我们只是通过压缩CSS就成功的节省了90%。从708K到68K。

#### …有时

然而，这是Gzip在**没有Base64样式表上**的成果。如果我们使用原来的CSS（有Base64），我们只能减少74.91%。

| Base64? | Gross Size | Compressed Size | Saving |
| ------- | ---------- | --------------- | ------ |
| Yes     | 925K       | 232K            | 74.91% |
| No      | 708K       | 68K             | 90.39% |

两个选项之间的差异是惊人的164K（70.68%）。**而我们只是通过移开那些更适合其他地方的内容就能够减少164K的CSS**。

所以，对Base64的压缩是很低效的。下次如果有人说’用Gzip...’，可以给他们看看这些结果（如果他们提倡使用Base64的话）。

### 为什么Base64这么糟糕？

我们现在已经很清楚在某种程度上Gzip是没办法帮我们处理Base64增加的文件大小，但这只是其中一小部分的问题。为什么我们这么害怕增加文件的大小？单个图片的大小就有可能超过232K，为什么我们不从图片开始解决呢？

好问题，我很高兴你提到图片...

### 关于图片

为了解释Base64有多糟糕，我们需要先知道图片有多好用。一个颇有争议的观点是：**图片的性能没有你想象中的那么差**。

当然，图片是个问题。实际上它们是页面膨胀的首要贡献者。截止[2016年12月2日](http://httparchive.org/interesting.php?a=All&l=Dec%202%202016)，图片占了平均网页资源的1623K（64.46%）。对比起来，我们232K的样式表就不足为道了吧。但是，浏览器在处理图片和样式表时是有着本质上的差别的：

#### 图片不阻止渲染，样式会。

不管图片是否加载完浏览器都会开始渲染。即使图片一直都加载不成功，浏览器也会渲染。图片不是关键资源，即使它们占用了过多的字节，它们也不是瓶颈。

而CSS是关键资源。浏览器在构建渲染树之前无法开始渲染页面，在构建CSSOM之前无法构造渲染树，在所有样式表加载完、解压缩和解析之前无法构造CSSOM。CSS才是瓶颈。

现在希望你能够明白为什么我们如此在意CSS的大小：它们只会延迟页面渲染，并且让用户盯着空白的屏幕看。希望你同时能够意识到Base64将图片转换为CSS文件内容是一件很荒谬的事情：为了追求性能，**你刚刚将数百K的非阻塞资源变成了阻塞资源**。所有这些图片都可以通过网络准备就绪，但它们现在却被迫与关键资源一起出现。这并不意味着图片会更快；它意味着关键资源会更慢。还有比这更糟糕的吗？！

当然有。

浏览器是很聪明的。它为我们做了很多的性能优化，很显然它们更专业。让我们考虑一下响应式：

	.masthead {
	  background-image: url(masthead-small.jpg);
	}
	
	@media screen and (min-width: 45em) {
	
	  .masthead {
	    background-image: url(masthead-medium.jpg);
	  }
	
	}
	
	@media screen and (min-width: 80em) {
	
	  .masthead {
	    background-image: url(masthead-large.jpg);
	  }
	
	}

我们给浏览器提供了三种可选的图片，但它只会下载其中一个。它决定需要哪个，然后下载，另外两个则不会使用。

但是如果我们用Base64，三个图片都会下载，实际开销是原本的三倍左右。下面是这个项目一段真实的CSS代码（为了显示，我移除了data，在Gzip之前，这段代码总共是26K；之后是18K）：

	@media only screen and (-moz-min-device-pixel-ratio: 2),
	       only screen and (-o-min-device-pixel-ratio:2/1),
	       only screen and (-webkit-min-device-pixel-ratio:2),
	       only screen and (min-device-pixel-ratio:2),
	       only screen and (min-resolution:2dppx),
	       only screen and (min-resolution:192dpi) {

	  .social-icons {
	    background-image:url("data:image/png;base64,...");
	    background-size: 165px 276px;
	  }

	  .sprite.weather {
	    background-image: url("data:image/png;base64,...");
	    background-size: 260px 28px;
	  }

	  .menu-icons {
	    background-image: url("data:image/png;base64,...");
	    background-size: 200px 276px;
	  }

	}

所有用户，不论是否使用retina设备（即便用户的浏览器不支持media queries），都将被迫下载额外的18K CSS，然后他们的浏览器甚至会把它们放在一起。

不论是否会被使用，Base64资源一定会被下载。真浪费，但是当你觉得这是浪费的时候，实际上阻碍渲染才是更糟糕的事情。

### 关于字体

到目前为止，我只提到图片，但是除了浏览器处理无样式/不可见Flash（FOUT或FOIT）的一些细微差别外，字体和图片几乎一样。在这个项目未压缩的CSS中，字体总共有166K（Gzip之后124K，真是可怕的压缩效果）。

不偏离文章主题太远，我们知道字体不是关键资源，这是好事：你的页面渲染不需要它们。但是，不同浏览器会以不同方式处理Web字体：

* **Chrome和Firefox在3s内几乎不显示文字**。如果字体在3s内加载成功，文字会从隐藏显示为你的自定义字体。如果字体在3s后还是获取不到，文字就会从隐藏显示为你定义的默认字体。这就是FOIT。

* **IE会立即显示默认字体**然后在你的自定义字体加载成功时立即切换。这是FOUT。我个人认为这是最优雅的解决方案。

* **Safari则会一直等待你的自定义字体加载成功才显示文字**。如果字体一直获取失败，文字就永远也不会出现。这是FOIT。这真的让人难以接受。你的用户几乎无法看到你网页上的任何文字。

为了解决这些问题，人们使用Base64将字体内联到样式表中：如果CSS和字体同时到达，就不会有什么FOIT或FOUT，因为CSSOM和字体解析几乎会同时发生。

跟图片一样，把你的字体转移到关键资源并不会提高它们的效率，只会延迟你的CSS。实际上有一些非常好的字体加载解决方案，不过Base64不在其中。

### 再来说说缓存

Base64同时也对我们复杂的缓存机制有影响：通过耦合字体，图片和样式，它们都受同样的规则控制。这意味着即使我们只是随便改变CSS的一个hex值（可能最多就六个字节的数据更改），就需要重新下载几百K的样式，图片和字体。

字体在这里是真正的罪魁祸首：它们是不太可能会改变的稳定资源。事实上，我刚刚检查了另外一个客户和我正在开展的一个长时间运行的项目：他们的CSS昨天刚修改；他们的字体却是在8个月之前修改的。想象一下，每次样式表有什么修改，用户都要被迫重新下载那些不变的字体。

Base64编码意味着我们没办法根据自己的变化来单独缓存内容，也意味着无论是否有改变都需要缓存不变的信息。这是一个不论怎样都输的局面。

我们需要关注基本分离：我的字体缓存不应该依赖于我的图片缓存，我的图片缓存也不应该依赖于我的样式缓存。

好了，让我们快速的概括下：

* Base64增加了文件的大小但我们却无法有效压缩（e.g.Gzip）。而这种行为会延迟加载，阻塞渲染。

* Base64把非关键资源（e.g.图片，字体）放到关键资源（e.g.样式表）中。这意味着在这种特殊情况下，在我们开始渲染页面之前，相比起68K的CSS，我们需要下载超过3.4倍的内容。我们白白的让用户等待那些他们原本并不需要等待的内容！

* Base64强制所有内容都需要下载，即使它们根本就不会被用到。这是一种浪费，而且还发生在我们的关键资源中。

* Base64限制了我们独立缓存的能力；我们的图片和字体被样式绑定，反之亦然。

总而言之，请避免使用Base64。

### 用数据说话

这篇文章写的都是我知道的。我并没有执行测试来证明：这只是浏览器的工作原理。但是，我还是决定往前一步，执行一些测试来看看我们所寻求的是怎样的事实和数据。具体请看第二部分内容。

---------------------

## Part2: 数据收集

为了证明使用Base64将静态资源（主要是图片）内嵌到样式表中的缺陷，我决定实际收集一些证据。我设置了一个简单的测试，对比’传统’加载资源和Base64两种方案的一些重要阶段和运行时间。

### 公平的测试

让我们从两个简单的被背景图片覆盖的HTML文件开始，第一个是普通加载，第二个是用Base64:

* [普通图片](http://csswizardry.net/demos/base64/)

* [Base64图片](http://csswizardry.net/demos/base64/base64.html)

[原图](https://www.flickr.com/photos/rockersdelight/26842162186/)是我的朋友[Ashley](https://twitter.com/iamashley)拍摄的。

我把它调整为1440x900px，通过JPEGMini和ImageOptim处理后保存为JPEG，然后才转化为Base64编码：

	harryroberts in ~/Sites/csswizardry.net/demos/base64 on (gh-pages)
	» base64 -i masthead.jpg -o masthead.txt

这是为了使图片适当优化，得到与实际情况更符合的Base64版本。

接着我新建了两个样式表：

	* {
	  margin:  0;
	  padding: 0;
	  box-sizing: border-box;
	}

	.masthead {
	  height: 100vh;
	  background-image: url("[masthead.jpg|<data URI>]");
	  background-size: cover;
	}

我把准备好的演示文件放在了一个实际网址上，这样我们就可以获得真实的延迟和带宽体验。

我在Chrome中打开了一个特定的性能测试配置文件，关闭其他打开的页面，准备好开始。

开启Chrome的Timeline开始测量。整个过程大概是这样：

* 禁用缓存

* 清楚剩余Timeline信息

* 刷新页面并记录网络和Timeline活动

* 丢弃任何关于DNS或TCP的连接（我不希望任何时间受到不相关网络活动的影响）结果

* 记录DOMContentLoaded，Load，First Paint，Parse Stylesheet和Image Decode

* 重复以上步骤直到得到5组干净的数据

* 隔离每个记录的中位数（中位数是[矫正的平均值](https://csswizardry.com/2017/01/choosing-the-correct-average/)）

* 针对Base64再次执行以上所有操作

* 在移动设备上再做一遍（最终得到四组数据：PC和移动端的Base64和非Base64[注4]）

第4点是最重要的：任何连接活动都会使结果发生倾斜并导致不一致，我们只有在绝对零连接开销的情况下才保留结果。

### 测试移动端

我通过调节CPU为3倍，网络为常规的2G，来模拟中档移动设备，并为移动设备完成了大量的测试。

在Google Sheets上你能看到我收集的[所有数据](https://docs.google.com/spreadsheets/d/1P720QU6CQ7pZUgCLtkOdUmwpwp_8nEyTMBN2zq5Ajmc/edit?usp=sharing)（所有数字的单位都是毫秒）。令我震惊的是数据的质量和一致性：很少有异常值。

现在先忽略预加载图片的数据（请看接下来的：第三种方法）。PC和移动端分为不同的表格（看屏幕底部的标签）。

### 一些见解

数据是很直观的，它们证实了我很多的猜想。你自己可以随意查看里面的细节，但我已经提取了最相关和有意义的信息：

* 在PC和移动端之间，**DOMContentLoaded事件在很大程度上保持不变**。这里并没有什么’更好’。

* **Load Event**在移动端的Base64和非Base64相似，但是**PC端的Base64是非Base64的2.02倍**（正常：236ms，Base64：476ms）。Base64更慢。

* **parsing stylesheets**的Base64明显更慢。在PC端**慢了超过10倍**。在**移动端慢了超过32倍**。Base64更慢。

* 在PC端，Base64**解压缩快过普通图片的1.23倍**。Base64更快。

* 但是在移动端，**普通图片解压缩快过Base64图片2.05倍**。Base64更慢。

* **First Paint**测量感知性能的一个很大的指标：它告诉我们用户何时开始看到内容。在PC端，普通图片的First Paint发生在280ms，但是Base64在629ms：**Base64慢了2.25倍**。

* 在**移动端**，普通图片的**First Paint**发生在774ms，Base64在7950ms。**Base64慢了10.27倍**。换句话说，普通图片在1s内开始绘制，Base64几乎要到8s才开始。令人咋舌，Base64明显比较慢。

通过以上这些信息可以很明显看出谁是完美的赢家：几乎所有方面，所有平台，如果我们远离Base64，会更快。我们尤其需要关注具有更高延迟和受限的性能和带宽的低功耗设备，这里是重灾区：**32倍慢的stylesheet和10.27倍慢的First Paint**。

### 第三种方法

普通图片的加载有一个问题是对瀑布流的影响：我们需要下载HTML，HTML请求CSS，CSS请求图片，这是同步的过程。Base64的理论优势是可以同时加载CSS和图片（实际上不是，虽然它们一起出现，但是它们也一起迟到了），这可以让我们使用更加并发的方式来加载资源。

幸运的是，有一种方法可以做到不用把所有图片内嵌入样式表就可以实现并行。通过提前加载，而不是将图片作为一个迟来的资源，像这样：

	<link rel="preload" href="masthead.jpg" as="image" />

我做了另外一个演示页面：

* [预加载图片](http://csswizardry.net/demos/base64/preload.html)

通过将这个标签放到HTML的**头部**，我们可以告诉HTML直接下载图片，而不用等CSS去请求它。这意味着，取代像这样的请求链：
	
	                                               |
	|-- HTML --|                                   |
	           |- CSS -|                           |
	                   |---------- IMAGE ----------|
	                                               |

我们可以这样：

	                                       |
	|-- HTML --|                           |
	           |---------- IMAGE ----------|
	           |- CSS -|                   |
	                                       |	     
	                                       
注意：
我们获取完整内容有多快？
图片在CSS之前加载会怎么样？

预加载可以让我们手动提前加载静态资源，再在之后的页面呈现。

我决定做一个普通图片的页面，取代CSS请求，我打算使用预加载：

	<link rel="preload" href="masthead.jpg" as="image" />
	<title>Preloaded Image</title>
	<link rel="stylesheet" href="image.css" />

我没有发现这个测试用例有多大的改进，预加载在这里并没有很有用：我的请求链太短，我们没有获得重新排序的真正好处。然而，如果我们的页面有许多静态资源，预加载可以给我们带来很大的益处。我在[我的主页](https://csswizardry.com/)上使用它来[预加载标头](https://github.com/csswizardry/csswizardry.github.com/blob/21044ecec9e11998d7a1e12e9f96be2aa990c652/_includes/head.html#L5-L15)：以这种方式使用它确实产生了感知上的一些重大变化。

然而我注意到一个非常有趣的事情，关于解码时间。在移动端，图片在25ms内解码，而PC端需要36.57ms。

* 预加载图片在移动端解码是PC端的1.46倍快。

* 预加载图片在移动端解码是没有预加载的3.53倍快。

我不确定为什么会这样，我简单粗暴的猜测：也许图片在实际需要之前不会被解码，所以如果在实际需要解码之前，在设备上已经有一堆字节了，那么这个过程可以更快地工作吗？任何在读这篇文章的人如果有谁知道这个答案的，请告诉我！ 

### 改进测试

我虽然尽量保持我的测试公平和不受影响，但如果给我更多时间我可以做得更好（不过这是周末嘛...）：

* **在真实设备上测试**。我通过DevTools调节我的CPU和连接，但是在真实设备上运行这些测试无疑会更好。

* **在移动端用一个更合适的图片**。我在尽可能多的变量中保持相同的测试，在PC和移动端使用一样的图片。实际上，我只是是模拟移动设备的网络能力，并没有使用较小的屏幕或资源。希望在现实世界中，我们可以为更小的设备提供一张更小的图片（在尺寸和文件大小上）。而我只是在完全相同的视图中加载完全相同的文件，只修改了连接和CPU。

* **测试一个更真实的项目**。这些都是实验，正如我在预加载中指出的，这并没有很有效。希望在非测试环境能够看到不一样的结果。

这就是我关于Base64性能影响的两篇文章。它虽然看起来像是在描述一些我们已经知道的事实，但是能够看到一些数字证明还是很好的，特别注意低端设备。Base64感觉上还是像一个巨大的反模式。



[注1] 在一些非常特殊的情况下它也许是明智的选择，但除非你绝对确定，否则那可能就是反模式。总之要非常谨慎，并始终认为Base64不是正确的选择。

[注2] 在Chrome的Sources中打开样式表，按文件左下角的{}。

[注3] 在所有行中运行全局命令（：g）; 找到包含数据的行：（/ data :）并删除它们（/ d）。

[注4] 这里需要解释一下：我基本上是在我的笔记本电脑和一个模拟的移动设备上进行测试的，我并不是在说屏幕尺寸。