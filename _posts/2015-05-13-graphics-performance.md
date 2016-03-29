---
layout: post
title: Graphics Performance
date: 2015-05-12
excerpt: 在上一篇文章中,我们通过不同的方法实现了几种自定义Button，我特意想展现的是这几种方法在性能上的不同。
category: "translation"
---

在上一篇[文章](http://robots.thoughtbot.com/designing-for-ios-taming-uibutton)中,我们通过不同的方法实现了几种自定义Button，我特意想展现的是这几种方法在性能上的不同。

###幕后花絮
为了了解影响性能的因素，我们需要仔细地观察iOS底层绘图技术栈(technology stack)。下图展示了和绘图相关的框架和库及其它们之间的关系：

![](/images/post/2015-05-13-graphics-performance-img01.png)

在最上层，`UIKit`管理着与用户界面的交互。它由一系列符合特定功能的UI控件组成，如：`UIButton`和`UILabel`。`UIKit`构建在`Core Animation`框架的最上层，`Core Animation`又是从OS X Leopardy一直到iOS中的，用于对动画进行平滑的转换。<br/>

更深一层的是`OpenGL ES`，一个开放标准的库用于在移动设备中对2D和3D计算机图形进行渲染。它被广泛的应用于游戏动画和`Core Animation`与`UIKit`底层动画的渲染上。软件层上另外一部分是`Core Graphics`——集成自`Quartz`（OS X中基于CPU的绘画引擎）。这两个底层的框架都是基于`C`语言开发的。<br />

最底层的是由图形处理器(CPU)和主处理器(CPU)组成。<br />

`GPU`会对图形的合成和渲染进行硬件加速，`OpenGL`和`CoreAnimation/UIKit`都是基于`GPU`硬件加速去实现的。相比于Android硬件加速是iOS的一大优势，由于Android在绘制动画过程中过分的依赖CPU导致在复杂动画过程中会出现微弱的延迟。<br />

屏幕外绘制（`Offscreen drawing`）是指在用到图像之前，提前由`GPU`在后台将位图渲染后，然后用到时候再有`CPU`绘制到屏幕中。<br/>

在iOS中，屏幕外绘制会自动的应用在以下几种情况：<br />
• `Core Graphics`（任何以“CG”为开头的类）<br />
• `drawRect()`方法中，即使他没有实现是空的<br />
• `CALayers`的`shouldRasterize`属性被设置为YES的时候<br />
• `CALayers`使用遮罩（`setMasksToBounds`）和动态阴影（`setShadow`）的时候<br />
• 任何文本显示在屏幕上的时候，包括`Core Text`<br />
• `UIViewGoupOpacity`<br />
作为一般的规则，当动画发生的时候屏幕外绘制会影响到性能。你可以连接iOS设备使用Instruments工具查看哪部分UI正在进行屏幕外绘制：<br />
 
*1.* 用USB连接iOS设备<br />
*2.* 打开Instruments<br />

![](/images/post/2015-05-13-graphics-performance-img02.png)

*3.* 选择 “iOS > Graphics > Core Animation template”

![](/images/post/2015-05-13-graphics-performance-img03.png)

*4.* 点击中间的纽扣图标

![](/images/post/2015-05-13-graphics-performance-img03.png)

*5.* 选择你的设备座位目标设备<br />
*6.* 勾选`"Color Offscreen-Rendered Yellow"`调试选项<br />
*7.* 此时在你设备中任何屏幕外绘制的图形都会被黄色覆盖。<br />
**Update:**你也可以勾选"Debug > Color Offscreen-Rendered"选项，在模拟器中观察屏幕外渲染的UI。除非你是在进行性能测试，否则在模拟器中去观察屏幕外渲染是非常直接和方便的。<br />

###UIButton的案例
现在让我们来比较一下[之前](http://robots.thoughtbot.com/post/33427366406/designing-for-ios-taming-uibutton)几种自定义Button方法的性能。<br />

####使用预渲染的资源
自定义使用`UIImage`做为背景的button，该`UIImage`完全是依赖GPU去对存储在硬盘中的图像资源进行渲染而得到的。可调整大小的背景图片是另外一种选择，这种做法的好处是可以减小app应用的大小，并且在进行像素拉伸和平铺的时候可以利用硬件加速。<br />

####使用CALayers
基于CALayer的方法的实现需要借助屏幕外绘制，因为它使用了遮罩去渲染圆角。我们还要明确的禁止掉使用`Core Animation`时默认打开的动画。除非你需要动画的切换，否则底线（Bottom line）这种技术是不适用于自定义画图的。<br />

####使用`drawRect`
`drawRect`方法依赖`Core Graphics`来完成自定义绘图，但是他最大的缺点在于它处理touch事件的方式：每当button被按下时，`setNeedDisplay`强制它进行重绘；简单的一次点击就有可能进行一次甚至是两次的重绘。这并不是一个利用CPU和内存的好的方式，尤其是在同一个界面中同时存在多个通过`drawRect`方法创建的UIButon实例的情况下。<br />

####混合的方法
那是不是意味着使用预渲染的资源是唯一可行的方案呢？并不是这样的。如果你仍然需要使用代码去进行绘制时，有一些方法可以优化你的代码并减少性能损耗。其中一种方法就是生成一张可拉伸的位图图像然后在所有的实例中复用这张图片。<br />
下面我们一步一步的去创建一个`UIButton`的子类，然后再定义我们的类级别的静态变量：
<pre><code>// In CBHybrid.m
#import "CBHybrid.h"

@implementation CBHybrid
// Resizable background image for normal state
static UIImage *gBackgroundImage;

// Resizable background image for highlighted state
static UIImage *gBackgroundImageHighlighted;
// Background image border radius and height

static int borderRadius = 5;
static int height = 37;</code></pre>
接下来我们将绘制代码从`drawRect`中移到一个新的helper方法中，同时我们将生成一张可拉伸的图片替代那张全尺寸的图片，然后我们将输出的图片存储到一个静态变量中以便接下来使用：
<pre><code>- (UIImage *)drawBackgroundImageHighlighted:(BOOL)highlighted {
    // Drawing code goes here
}</code></pre>
首先，我们需要知道这张图片的宽度。为了使得性能最优，我们只在中心留出1pt的拉伸区域。
<pre><code>float width = 1 + (borderRadius * 2);
</code></pre>
*The height matters less in this case, as long as the button is tall enough for the gradient to be visible.*为了使得高度与其他button匹配我们设置图片高度为37pt。<br />

接下来，我们需要一个位图上下文来绘图：
<pre><code>UIGraphicsBeginImageContextWithOptions(CGSizeMake(width, height), NO, 0.0);
CGContextRef context = UIGraphicsGetCurrentContext();
CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
</code></pre>
第二个布尔参数设为`NO`确保我们的图形上下文是非透明的。最后一个参数是比例因子（屏幕密度）。当设置为`0.0`的时候，他将被设置为设备默认的缩放因子。<br />

下面的代码块与之前在`CBBezier`中用`Core Graphics`所实现的功能类似，保存更新的数据后用`highlighted`参数代替默认的`self.highlighted`
<pre><code>// Gradient Declarations
// NSArray *gradientColors = ...

// Draw rounded rectangle bezier path
UIBezierPath *roundedRectanglePath = [UIBezierPath bezierPathWithRoundedRect: CGRectMake(0, 0, width, height) cornerRadius: borderRadius];

// Use the bezier as a clipping path
[roundedRectanglePath addClip];

// Use one of the two gradients depending on the state of the button
CGGradientRef background = highlighted? highlightedGradient : gradient;

// Draw gradient within the path
CGContextDrawLinearGradient(context, background, CGPointMake(140, 0), CGPointMake(140, height-1), 0);

// Draw border
// [borderColor setStroke...

// Draw Inner Glow
// UIBezierPath *innerGlowRect...
</code></pre>
剩下来我们唯一要做的是在`CBBezier`中将绘制的输出存储到一个`UIImage`中，然后调用`UIGraphicsEndImageContext`清理上下文环境。
<pre><code>UIImage* backgroundImage = UIGraphicsGetImageFromCurrentImageContext();
// Cleanup
UIGraphicsEndImageContext();
</code></pre>
现在我们有了生成背景图片的方法，接下来我们得实现一个初始化方法来实例化这些图片并将它们设置为`CBHybrid`实例的背景。
<pre><code>- (void)setupBackgrounds {
    // Generate background images if necessary
    if (!gBackgroundImage && !gBackgroundImageHighlighted) {
        gBackgroundImage = [[self drawBackgroundImageHighlighted:NO] resizableImageWithCapInsets:UIEdgeInsetsMake(borderRadius, borderRadius, borderRadius, borderRadius) resizingMode:UIImageResizingModeStretch];
        gBackgroundImageHighlighted = [[self drawBackgroundImageHighlighted:YES] resizableImageWithCapInsets:UIEdgeInsetsMake(borderRadius, borderRadius, borderRadius, borderRadius) resizingMode:UIImageResizingModeStretch];
    }
    // Set background for the button instance
    [self setBackgroundImage:gBackgroundImage forState:UIControlStateNormal];
    [self setBackgroundImage:gBackgroundImageHighlighted forState:UIControlStateHighlighted];
}
</code></pre>
初始化时我们设置按钮的属性为`custom`并实现`initWithCoder`(或者如果需要使用代码去初始化button时需要实现`initWithFrame`)：
<pre><code>+ (CBHybrid *)buttonWithType:(UIButtonType)type
{
    return [super buttonWithType:UIButtonTypeCustom];
}
- (id)initWithCoder:(NSCoder *)aDecoder {
    self = [super initWithCoder:aDecoder];
    if (self) {
        [self setupBackgrounds];
    }
    return self;
}
</code></pre>
为了确保新的类能够正常工作，在Interface Builder中复制一个button修改他的类为`CGHybrid`.修改button的`content`为`CGContext-generater image`然后创建运行。

![](/images/post/2015-05-13-graphics-performance-img05.png)

完整的子类的代码在[这儿](https://github.com/kaishin/custom-UIButton/blob/master/Custom%20UIButtons/CBHybrid.m)。

###结束语
该说的都说了该所的也都做了，其实预渲染资源比任何基于代码的方案表现的都要好。但是话说回来，就灵活性和效率而言`Core Graphicsis`的使用已经足够了，我们上面所做的性能上的优化，在当今的硬件环境中已经看不出明显的差别了。

*原文地址：[https://robots.thoughtbot.com/designing-for-ios-graphics-performance](https://robots.thoughtbot.com/designing-for-ios-graphics-performance)*