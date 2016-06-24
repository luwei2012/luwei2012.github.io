---
layout: post
title: "CoreTextView"
modified:
categories: IOS/CustomView
description: "竖排文字显示"
tags: [Vertical Text]
image:
  feature: /IOS/CustomView/coreTextView.png
  credit:
  creditlink:
comments:
share:
date: 2015-11-04T11:39:43+08:00
---

### 如何实现一个轻量级的竖排文字显示控件 
<figure>
	<img src="/images/IOS/CustomView/releaseV1.1.png" alt="项目实例">
	<figcaption><a href="https://github.com/luwei2012/CoreTextView">CoreTextView</a></figcaption>
</figure>
感谢<a href="http://geeklu.com/2013/03/core-text/">卢克</a>的博客和代码，给了我很多帮助和启发。卢克的博客里详细介绍了如何使用Core Text来绘制富文本，
而且展示了苹果如何处理字体的分解图，加上<a href="https://developer.apple.com/library/mac/documentation/StringsTextFonts/Conceptual/CoreText_Programming/LayoutOperations/LayoutOperations.html">苹果官方的demo</a>,
最终我拼凑出来一个竖排显示文字的控件。

### 竖排显示文字的核心思想
说起来很简单，可能大家也都看到过，其实在IOS的富文本属性中有两个不常用的属性：NSWritingDirectionAttributeName和NSVerticalGlyphFormAttributeName。
故名思议NSVerticalGlyphFormAttributeName就是控制文字竖向还是横向绘制，而NSWritingDirectionAttributeName控制的是文本绘制方向。
下面就很简单了，只需要把NSVerticalGlyphFormAttributeName设置为1(0 means horizontal text.  1 indicates vertical text.),NSWritingDirectionAttributeName设置为
@[@(NSWritingDirectionRightToLeft)]，理论上就能实现一个古文风格的竖排显示效果了。

于是我满心欢喜的构建了一个富文本对象，并添加了这两个属性，然后发现并没有任何卵用。首先是NSVerticalGlyphFormAttributeName只针对英文起作用，其次NSVerticalGlyphFormAttributeName并没有改变绘制的方向。
这里我需要拎清楚两个概念：我在上面说NSWritingDirectionAttributeName控制的是文本绘制方向意思是文本从左到右绘制或者从右到左，这个我称之为文本绘制方向；我说NSVerticalGlyphFormAttributeName并没有改变绘制的方向
意思是把文字渲染到屏幕的方向。

在我设置了NSVerticalGlyphFormAttributeName之后，文本里面的字母顺时针旋转了90°，变成了竖向显示，但是整个文本还是一行。也许有人就会想直接把控件旋转90°不就行了么，对于单行文本
这种做法确实是最简单有效地，但是需要绘制多行文本的时候，这个方法就很难控制了。你需要创建多个UILabel并设置他们的坐标、宽高等等，还需要自己切分文本，分段显示。。。听起来就觉得不可能实现。。


偶然的机会，我在stackoverflow上看到一个提问，贴了一段代码，居然能实现了文本行变列，他提的问题是如何能使汉字和英文字符高度整体居中对齐。这种感觉就像是我还在解决温饱问题
而人家开始像怎么找乐子了。。。于是我那段代码扒了下来仔细研究了下。最核心的部分分为两个：

1.创建CTFrameRef的时候需要添加一个属性kCTFrameProgressionAttributeName 

<pre class="brush: cpp;auto-links: true;collapse: true;first-line: 1;gutter: true;html-script: true;light: true;ruler: false;smart-tabs: true;tab-size: 4;toolbar: true;">
/*!
	@const		kCTFrameProgressionAttributeName
	@abstract	Specifies progression for a frame.
	
	@discussion Value must be a CFNumberRef containing a CTFrameProgression.
				Default is kCTFrameProgressionTopToBottom. This value determines
				the line stacking behavior for a frame and does not affect the
				appearance of the glyphs within that frame.

	@seealso	CTFramesetterCreateFrame
*/
CTFrameRef frame = CTFramesetterCreateFrame(framesetter,
                                            CFRangeMake(0, 0),
                                            path,
                                            (CFDictionaryRef)@{(id)kCTFrameProgressionAttributeName: @(kCTFrameProgressionRightToLeft)});
</pre>
<pre class="brush: cpp;auto-links: true;collapse: true;first-line: 1;gutter: true;html-script: true;light: true;ruler: false;smart-tabs: true;tab-size: 4;toolbar: true;">
/*!
    @enum		CTFrameProgression
    @abstract	These constants specify frame progression types.

    @discussion The lines of text within a frame may be stacked for either
                horizontal or vertical text. Values are enumerated for each
                stacking type supported by CTFrame. Frames created with a
                progression type specifying vertical text will rotate lines
                90 degrees counterclockwise when drawing.

    @constant	kCTFrameProgressionTopToBottom
                Lines are stacked top to bottom for horizontal text.

    @constant	kCTFrameProgressionRightToLeft
                Lines are stacked right to left for vertical text.

    @constant	kCTFrameProgressionLeftToRight
                Lines are stacked left to right for vertical text.
*/

typedef CF_ENUM(uint32_t, CTFrameProgression) {
    kCTFrameProgressionTopToBottom  = 0,
    kCTFrameProgressionRightToLeft  = 1,
    kCTFrameProgressionLeftToRight  = 2
}
</pre>
至此我们就能控制文本的行变列。

2.让汉字保持竖向

添加了kCTFrameProgressionAttributeName属性后其实就相当于把绘制区域顺时针旋转了90°，汉字躺下了。。。因此我们需要为每个汉字额外设置一些样式：
<pre class="brush: cpp;auto-links: true;collapse: true;first-line: 1;gutter: true;html-script: true;light: true;ruler: false;smart-tabs: true;tab-size: 4;toolbar: true;">
/*!
    @const      kCTVerticalFormsAttributeName
    @abstract   Controls glyph orientation.

    @discussion Value must be a CFBooleanRef. Default is false. A value of false
                indicates that horizontal glyph forms are to be used, true
                indicates that vertical glyph forms are to be used.
*/

extern const CFStringRef kCTVerticalFormsAttributeName CT_AVAILABLE(10_5, 4_3);
NSMutableAttributedString *attrStr = [[NSMutableAttributedString alloc] initWithString:self.text attributes:nil];

for (int i = 0; i < attrStr.length; i++) {
    if ([self isChinese:self.text index:i]) {
        [attrStr addAttribute:(id)kCTVerticalFormsAttributeName
                        value:@YES
                        range:NSMakeRange(i, 1)]; 
    }
}
</pre>

这两步处理完之后我们的文本就能够竖向显示了，只是字母跟汉字不会右对齐。但是到这里我们才刚开始。

### 如何才能让文本居中显示？
使用Core Text绘制文本主要有

1.获取画布并设置好坐标系

<pre class="brush: cpp;auto-links: true;collapse: true;first-line: 1;gutter: true;html-script: true;light: true;ruler: false;smart-tabs: true;tab-size: 4;toolbar: true;">
//获取画布句柄
CGContextRef context = UIGraphicsGetCurrentContext();
//颠倒窗口 坐标计算使用的mac下的坐标系 跟ios的坐标系正好颠倒
CGContextSetTextMatrix(context, CGAffineTransformIdentity);
CGContextTranslateCTM(context, 0, self.bounds.size.height);
CGContextScaleCTM(context, 1.0, -1.0);
</pre>

2.生成需要绘制的内容

<pre class="brush: cpp;auto-links: true;collapse: true;first-line: 1;gutter: true;html-script: true;light: true;ruler: false;smart-tabs: true;tab-size: 4;toolbar: true;">
//生成富文本的信息 具体不懂  反正这是core text绘制的必须流程和对象
CTFramesetterRef framesetter = CTFramesetterCreateWithAttributedString((__bridge CFAttributedStringRef)self.attributedText);
</pre>
这里的attributedText就是一个富文本对象，我们可以设置行间距、字间距、颜色和字体等等一系列属性。

3.生成绘制区域

<pre class="brush: cpp;auto-links: true;collapse: true;first-line: 1;gutter: true;html-script: true;light: true;ruler: false;smart-tabs: true;tab-size: 4;toolbar: true;">
//这一步生成合适的绘制区域
CGPathRef path = [self createPathWithLines];
</pre>
createPathWithLines方法我们后面要讲，最简单的实现就是把整个self.bound作为绘制区域，就跟卢克的博客里写的一样

4.绘制

<pre class="brush: cpp;auto-links: true;collapse: true;first-line: 1;gutter: true;html-script: true;light: true;ruler: false;smart-tabs: true;tab-size: 4;toolbar: true;">
// Create a frame for this column and draw it.
CTFrameRef frame = CTFramesetterCreateFrame(framesetter,
                                            CFRangeMake(0, 0),
                                            path,</br>
                                            (CFDictionaryRef)@{(id)kCTFrameProgressionAttributeName: @(kCTFrameProgressionRightToLeft)});
CTFrameDraw(frame, context);
</pre>

5.释放内存

<pre class="brush: cpp;auto-links: true;collapse: true;first-line: 1;gutter: true;html-script: true;light: true;ruler: false;smart-tabs: true;tab-size: 4;toolbar: true;">
//释放内存
CFRelease(frame);
CFRelease(path);
CFRelease(framesetter);
</pre>

由此可见想要实现居中显示，我们只需要计算出正确的绘制区域就行了。其实不光是居中，你想要啥对齐效果都可以通过设置绘制区域来达到。
计算绘制区域的方法可以参见<a href="https://github.com/luwei2012/CoreTextView">我的代码</a>都有详细的注释。

### 处理横竖屏切换
这个很简单，横竖屏切换会触发layoutSubviews方法，在layoutSubviews方法里重绘即可。