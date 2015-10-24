---
layout: post
title: "关于ScrollView的学习"
modified:
categories: IOS/record
description:
tags: []
image:
  feature:
  credit:
  creditlink:
comments:
share:
date: 2015-10-22T17:10:07+08:00
---

ScrollView应该是ios里面最常用的控件之一了，前两天在项目<a href="https://www.9panart.com/html/passport/passport_login.html?ran=19406317709945142">泛艺术App</a>里面寻找一个UI的解决方案的时候曾经想过使用ScrollView，虽然最后使用的是它的儿子TableView，中间还是在ScrollView上吃了点苦头，
于是才决定写个demo好好研究下ScrollView。

### ScrollView的探究过程
ScrollView的特性相信大家都知道，有几个非常重要的属性：frame，contentSize，contentOffset，contentInset
    {% highlight html %}
    -(UIScrollView *)scrollView{
        if (_scrollView == nil) {
            _scrollView                 = [[UIScrollView alloc] init];
            _scrollView.delegate        = self;
            _scrollView.scrollEnabled   = true;
            _scrollView.bounces         = true;
            _scrollView.showsHorizontalScrollIndicator  = false;
            _scrollView.showsVerticalScrollIndicator    = true;
            _scrollView.userInteractionEnabled          = YES;
            _scrollView.backgroundColor                 = [UIColor blueColor];
            _scrollView.contentInset                    = UIEdgeInsetsMake(100, 100, 100, 0);
            [self.view addSubview:_scrollView];
            [_scrollView mas_makeConstraints:^(MASConstraintMaker *make) {
                make.left.equalTo(self.view.mas_left);
                make.top.equalTo(self.view.mas_top);
                make.right.equalTo(self.view.mas_right);
                make.bottom.equalTo(self.view.mas_bottom);
            }];

        }

        return _scrollView;
    }
    -(UIView *)containerView{
        if (_containerView == nil) {
            _containerView = [[UIView alloc] init];
            [self.scrollView addSubview:_containerView];
            [_containerView mas_makeConstraints:^(MASConstraintMaker *make) {
                make.left.equalTo(self.view.mas_left);
                make.top.equalTo(@(0));
                make.right.equalTo(self.view.mas_right);
                make.height.equalTo(@(SCREEN_HEIGHT * 2));
            }];
            self.scrollView.contentSize = CGSizeMake(SCREEN_WIDTH, SCREEN_HEIGHT * 2);
        }
        return _containerView;
    }
    -(void)viewDidAppear:(BOOL)animated{
        [super viewDidAppear:animated];
         NSLog(@"%f %f",self.containerView.frame.origin.x,self.containerView.frame.origin.y);
    }
    {% endhighlight %}
<ul>
<li>
    contentInset:之所以首先说contentInset这个属性是因为我自己就被网上很多不负责任的言论坑过。contentInset只是contentSize的四个边增加了一块区域，仅此而已。
    网上有人说：“contentInset是scrollView的contentView的顶点相对于scrollView的位置，
    例如你的contentInset = (0 ,100)，那么你的contentView就是从scrollView的(0 ,100)开始显示”
    这句话完全就是在放P。contentInset不会对contentView产生任何影响.</br>
    上面是我demo里面的一段代码，我使用了<a href="https://github.com/SnapKit/Masonry">Masonry这个库</a>在代码里面设置约束，比IOS自带的接口方便好用几百倍。</br>
    可以看到我在scrollView中添加了一个高度为两个屏幕高度的containerView，并且设置了contentSize和contentInset。结果打印出来的containerView的坐标是(0,0)。也就是说contentInset并不会
    影响containerView的位置，这个也很合理，因为我们并不想看到contentInset，必然不能把containerView的坐标给改了，否则程序一开始显示的就会是contentInset的内容了。
</li>

<li>
    frame:也是一个让我迷惑的属性。从我上面的代码里可以看到，我并没有明确的设置frame，由此可以得出结论：frame可以通过scrollView的约束条件来设置，这无疑是一个好消息。
</li>

<li>
    contentSize:这个属性就更加迷惑了。我在网上看到过一篇博客，是讲如何对scrollView使用autoLayout。博客里面的代码很简单，一个scrollView包含一个containerView，然后在containerView里面添加子View。
    然后问题就简化成只需要保证containerView跟scrollView的约束正确，containerView里面的内容布局跟普通的布局一样。</br>
    <a href="http://www.cocoachina.com/ios/20141011/9871.html">原文说</a>："我们知道scroll view除了自身的布局需要考虑（x, y, width, height）外，
    还有一个contentSize属性也必须要在布局的过程中进行确定，contentSize是UIScrollView用于确定它所 要展示的内容尺寸的大小，
    而这个contentSize在布局中实际上是又scroll view的子view :content view的宽和高实现的，注意：我们不能将content view的宽和高的约束设定为由scroll view决定
    （如和scroll view等宽、等高）,否则，Xcode会有警告：scroll view的content size不确定！</br>在这种情况下，我们必须要对content view的布局约束引入scroll view之外其他参照物，
    我们拖进来一个辅助的view作为参照物or锚点。通过这个参考view，确定content view的宽度和高度，尽管content view的尺寸可以不依赖于scroll view，但我们还不得不设定content view 和其父view的关系：
    具体而言就是要确定content view和scroll view的top, bottom, leading和trailing contstraints，这个地方可能比较具有迷惑性，原因是苹果对于这四个约束的使用在scroll view中做了变化：
    它不再是确定content view尺寸的依据，而是帮助scroll view中content view四周的边界（or你可以理解为留白），进而确定scroll view的contentSize属性
    "从这些话我推测如果是通过sb文件创建出来的scrollView会自动设置contentSize大小，问题是这个大小是怎么确定的，是不是就真的是scrollView里面所有内容的大小，还是scrollView的frame大小。
    通过我写的一个demo我得出了最终结论：scrollView的contentView是根据里面所有内容的大小设定的。

</li>

<li>
    contentOffset:这个属性没杀好说的，决定了scrollView显示的内容的起始点。
</li>

</ul>

### ScrollView的额外问题
<ul>
<li>
通过上面的探究过程可以知道我们可以直接在sb文件里面对scrollView使用autoLayout。由此引发了我一个疑问，能够通过修改containerView的高度约束来修改scrollView的contentSize么？
还是说scrollView的contentSize
在awakeFromNib的时候就定死了，只能通过手动设置重新改大小呢？</br>
通过demo测试发现，这个想法是可行的，但是在重新设置了约束之后需要调用<code>[self.view layoutIfNeeded];</code>这样scrollView才会重新计算contentSize。</br>
还有个相似的方法<code>[self.view updateConstraints];</code>这个方法不会立马更新scrollView的contentSize，而是在一定时间后更新，我猜测这个时间应该就是更新UI的动画时间。
为了防止计算出错还是建议都使用layoutIfNeeded方法。
</li>
</ul>

### ScrollView的总结
关于scrollView目前想知道的就这些，这篇博客做个总结：使用scrollView最好是只添加一个UIView作为容器View，然后再在这个容器View里面设置你的布局。
这种方式很像Android的XML文件布局，只有一个LinearLayout作为容器。这样做的好处就是更改contentSize非常方便，直接修改容器View的高度约束，然后调用layoutIfNeeded方法就可。
切记要给容器View设置上下左右间距以及宽高.


