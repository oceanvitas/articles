# translate

## [Naming Things](https://24ways.org/2014/naming-things/)

> 计算科学中最难的两件事是命名和缓存失效。 -- Phil Karlton

作为一名专业的网页开发者意味着对自己编写的代码负责，并确保其他人可以理解。
拥有记录的代码风格是实现此目的的一种手段，尽管你正在开发的项目的大小和类型将决定所使用的约定以及它们的强制执行情况。

在内部工作可能意味着与多个开发人员合作，也许在分布式团队中，他们同时进行更改 - 可能是重要的代码库。
左侧未选中，此代码库可能变得笨重。
编码公约确保每个人都可以做出贡献，并帮助构建一个作为一个整体的产品。

即使在较小的项目中，也许在代理机构内部或由自己进行工作，某些时候产生的产品需要交给第三方。
因此，确保你的代码可以被最终获得所有权的人所理解，这是明智的。

简单来说，代码读取的速度比写入或更改的频率更高。
一致和可预测的命名方案可以使代码更容易让其他开发人员了解，改进和维护，大概让他们免费担心缓存无效。

### 我们来谈论语义

名称不仅允许我们识别对象，而且还可以帮助我们描述被识别的对象。

语义（词的含义或解释）是基于标准的Web开发的基石。
使用恰当的HTML元素可以让我们创建具有隐式结构意义的文档和应用程序。
感谢HTML5，我们能够选择的词汇越来越大。

HTML元素提供一个层次的意义：广泛接受的文档底层结构描述。
只有浏览器供应商和开发人员相互协商，<p>才表示一个段落。

然而，除了广泛接受的微数据和微格式模式之外，只有HTML元素传达了用户代理可以一致解析的任何含义。
虽然为类名使用语义值是一种崇高的努力，但它们不向网站的访问者提供其他信息；把它们拿走，一个文件将具有完全相同的语义价值。

我并不认为总是这样，但是现实世界有改变观点的习惯。
关于语义的大部分思想已经被我同龄人的写作告知了。
在[`关于HTML语义和前端架构`](http://nicolasgallagher.com/about-html-semantics-front-end-architecture/)中，Nicholas Gallagher写道：

> 在不平凡的应用中，类名称语义的重要之处在于它们以实用主义驱动，并且最适合其主要目的 - 为开发人员提供有意义，灵活和可重复使用的演示/行为钩。

Harry Roberts在他的[`CSS指南`](http://cssguidelin.es/#naming)中回应了这些想法：

> 关于语义的辩论多年来一直很激烈，但重要的是我们采取更实际，更合理的方式来命名事物，以便更有效和更有效的工作。
> 相对于专注于‘语义学’，我们应该更加密切地关注感性和长寿 - 根据维护的便利性选择名称，而不是为了感知的意义。

### 命名方法

前端发展近年来发生了革命。
当我们工作的项目变得越来越大和重要，我们的发展实践已经成熟。
面向对象的CSS方法的利弊可以无休止地辩论，但是他们的介绍强调了具有文件化命名方案的有用性。

Jonathan Snook的[SMACSS](http://smacss.com/)（适用于CSS的可扩展和模块化架构）将样式规则收集到五个类别中：基础，布局，模块，状态和主题。
这个分组可以清楚每个规则的作用，并由[命名约定](http://smacss.com/book/categorizing)辅助：

> 通过将规则划分为五个类别，命名约定有助于立即了解特定样式所属的类别及在页面整体范围内的作用。
> 在大型项目中，更有可能在多个文件中分解样式。
> 在这些情况下，命名约定也使得更容易找到某个样式属于哪个文件。

> 我喜欢使用前缀区分布局，状态和模块规则。
> 对于布局，我使用`l-`但`layout-`也可以。
> 使用`grid-`的前缀也提供了足够的清晰度，可以将布局样式与其他样式分开。
> 对于状态规则，我喜欢使用`is-`，比如`is-hidden`或`is-collapsed`。
> 这有助于以非常可读的方式描述事物。

SMACSS相比起僵化的框架更像一套建议，所以它的想法可以纳入你自己的实践。
Nicholas Gallagher的[SUIT CSS](https://github.com/suitcss/)项目的[命名规则](https://github.com/suitcss/suit/blob/master/doc/naming-conventions.md)要严格得多：

> SUIT CSS依赖于结构化类名和有意义的连字符（即不使用连字符来分割单词）。
> 这有助于解决将CSS应用与DOM的当前限制（即缺少样式封装），并更好地传达类之间的关系。

在过去一年中，我喜欢[BEM启发的CSS方法](http://csswizardry.com/2013/01/mindbemding-getting-your-head-round-bem-syntax/)。
BEM代表块，元素，修饰符，它描述了有助于单个组件风格的三种类型的规则。
这意味着，给出以下标记：

	<ul class=“sleigh”>
	    <li class=“sleigh__reindeer sleigh__reindeer––famous”>Rudolph</li>
	    <li class=“sleigh__reindeer”>Dasher</li>
	    <li class=“sleigh__reindeer”>Dancer</li>
	    <li class=“sleigh__reindeer”>Prancer</li>
	    <li class=“sleigh__reindeer”>Vixen</li>
	    <li class=“sleigh__reindeer”>Comet</li>
	    <li class=“sleigh__reindeer”>Cupid</li>
	    <li class=“sleigh__reindeer”>Dunder</li>
	    <li class=“sleigh__reindeer”>Blixem</li>
	</ul>

我知道：
* `.sleigh`是一个包含块或组件。
* `.sleigh__reindeer`仅用作`.sleigh`的后代元素。
* `.sleigh__reindeer--famous`仅用作`.sleigh__reindeer`的修饰符。






















