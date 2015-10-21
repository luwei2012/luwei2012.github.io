---
layout: post
title: "MBXPageViewController的两点优化修改"
modified:
categories: IOS/structure
description: "MBXPageViewController的两点优化修改."
tags: [IOS structure]
image:
  feature: photo01.jpg
  credit:
  creditlink:
date: 2015-10-20T19:21:56+08:00
---

### MBXPageViewController是什么？
<figure>
	<img src="/images/IOS/structure/MBXPageViewController/scrollDelegateBefore.gif" alt="项目实例">
	<figcaption>泛艺术客户端.</figcaption>
</figure>
这里是<a href="https://github.com/Moblox/MBXPageViewController">原作者的主页</a>，Moblox也是参照的其他作者的源代码改的，为了集成更多的样式。
这个控件是我一个同事集成到项目中来的，集成后UI那边添加了两个需求：1.点击title上面的按钮需要让当前页面滚动到顶部 2.title下面的tabIndicator需要跟页面同步滑动。
于是我不得不去研究了下MBXPageViewController的源码。

MBXPageViewController是仿照RKSwipeBetweenViewControllers写的，通过源码可以看出并不是一个完成品，因为RKSwipeBetweenViewControllers是支持tabIndicator与页面同步滑动的。
所以第二个问题就简单了，把MBXPageViewController完成就可以了。

首先我对比了MBXPageViewController和RKSwipeBetweenViewControllers的优缺点。虽然MBXPageViewController的watch和fork远低于RKSwipeBetweenViewControllers，
MBXPageViewController的使用还是远远比RKSwipeBetweenViewControllers简单清晰，而且能适应更多的特殊UI情况。

使用MBXPageViewController.
{% highlight html %}
MBXPageViewController *MBXPageController = [MBXPageViewController new];
MBXPageController.MBXDataSource = self;
MBXPageController.MBXDataDelegate = self;
[MBXPageController reloadPages];
{% endhighlight %}
很简单易懂，有可能有人不理解，为什么没有看到addChildViewController或者addSubview之类的方法？其实这些都封装在了[MBXPageController reloadPages]方法里面
进去看源码就会发现：
{% highlight html %}
- (void)loadControllerAndView
{
    NSAssert([[self MBXDataSource] isKindOfClass:[UIViewController class]], @"This needs to be implemented in a class that inherits from UIViewController");
    [(UIViewController *)[self MBXDataSource] addChildViewController:self];
    [[[self MBXDataSource] MBXPageContainer] addSubview:self.view];
}
{% endhighlight %}
<ul>
<li>addChildViewController我的理解就是重新设置子controller的父controller，也就是层级关系，防止后面再Push或者Present新窗口的时候出错</li>
<li>addSubview就更加直观了不需要多解释，没有这句话咱们就没有View可以显示了。</li>
</ul>
需要注意的是[[self MBXDataSource] MBXPageContainer]这个方法，他是MBXDataSource协议里必选的方法
说到这里就必须说说为什么我认为MBXPageViewController的使用远远比RKSwipeBetweenViewControllers简单清晰，就是因为两个协议：
{% highlight html %}
@protocol MBXPageControllerDataSource <NSObject>
@required
- (NSArray *)MBXPageButtons;
- (NSArray *)MBXPageControllers;
- (UIView *)MBXPageContainer;
@end

@protocol MBXPageControllerDataDelegate <NSObject>
@optional
- (void)MBXPageChangedToIndex:(NSInteger)index;
@end
{% endhighlight %}

<ul>
<li>MBXPageButtons 就是标题栏上面的按钮，需要注意的是MBXPageViewController并不管按钮的显示，MBXPageViewController只负责帮我们绑定按钮的触摸事件，
    显示正确的对应的页面</li>
<li>MBXPageControllers 这个就是咱们页面的数组了。</li>
<li>MBXPageContainer 是页面的容器，只需要返回MBXPageViewController所在的UIViewController的container就行。</li>
<li>MBXPageChangedToIndex 是一个提供给用户更新按钮状态的方法，当页面索引改变后就会触发这个方法，用户应该在这里跟新title
    上面按钮和tabIndicator的状态.</li>
</ul>
整个协议的完整实现大概应该是这个样子：
{% highlight html %}
#pragma mark - MBXPageViewController Data Source

- (NSArray *)MBXPageButtons
{
    return @[self.button1, self.button2, self.button3];
}

- (UIView *)MBXPageContainer
{
    return self.container;
}

- (NSArray *)MBXPageControllers
{
    // You can Load a VC directly from Storyboard
    UIStoryboard* mainStoryboard = [UIStoryboard storyboardWithName:@"Main" bundle:nil];

    UIViewController *demo  = [mainStoryboard instantiateViewControllerWithIdentifier:@"firstController"];
    UIViewController *demo2  = [mainStoryboard instantiateViewControllerWithIdentifier:@"secondController"];

    // Or Load it from a xib file
    UIViewController *demo3 = [UIViewController new];
    demo3.view = [[[NSBundle mainBundle] loadNibNamed:@"View" owner:self options:nil] objectAtIndex:0];

    // Or create it programatically
    UIViewController *demo4 = [[UIViewController alloc] init];
    demo4.view.backgroundColor = [UIColor orangeColor];

    UILabel *fromLabel = [[UILabel alloc]initWithFrame:CGRectMake( (self.view.frame.size.width - 130)/2 , 40, 130, 40)];
    fromLabel.text = @"Fourth Controller";

    [demo4.view addSubview:fromLabel];

    // The order matters.
    return @[demo,demo2, demo3];
}
{% endhighlight %}
怎么样，是不是非常简单清晰。下面需要解决的就是我们UI提出的需求问题了，也很简单，修改绑定button的事件：
{% highlight html %}
- (void)controllerModeLogicForDestination:(NSInteger)destination
{
    __weak __typeof(&*self)weakSelf = self;
    NSInteger tempIndex = _currentPageIndex;
    // Check to see which way are you going (Left -> Right or Right -> Left)
    if (destination > tempIndex) {
        for (int i = (int)tempIndex+1; i<=destination; i++) {
            [self setPageControllerForIndex:i direction:UIPageViewControllerNavigationDirectionForward currentMBXViewController:weakSelf destionation:destination];
        }
    }

    // Right -> Left
    else if (destination < tempIndex) {
        for (int i = (int)tempIndex-1; i >= destination; i--) {
            [self setPageControllerForIndex:i direction:UIPageViewControllerNavigationDirectionReverse currentMBXViewController:weakSelf destionation:destination];
        }
    }else{
        [weakSelf updateCurrentPageIndex:tempIndex];
    }
}
{% endhighlight %}
只加了一句[weakSelf updateCurrentPageIndex:tempIndex];然后我们可以自己在updateCurrentPageIndex方法里判断，如果当前索引跟传入的索引相同则当成点击事件来处理，否则则是页面切换事件，需要跟新button的状态，
这样我们就解决了第一个问题。第二个问题比较复杂，我到现在都还没完成搞清楚why，只知道要这么做。
我们先把MBXPageControllerDataDelegate协议拓展一下，添加两个方法：
{% highlight html %}
@protocol MBXPageControllerDataDelegate <NSObject>
@optional
- (void)MBXPageChangedToIndex:(NSInteger)index;
- (void)MBXPageSelectdViewOffset:(CGFloat)offset;
- (CGFloat)MBXPageTabIndicatorStartOffset:(NSInteger)tabIndex;
@end
{% endhighlight %}
<ul>
<li>MBXPageTabIndicatorStartOffset返回tabIndex被选中时的tabIndicator的X坐标。别问我为什么要这么做，我也是一知半解。</li>
<li>MBXPageSelectdViewOffset就是事实跟新tabIndicator的X坐标的方法，offset就是X的新坐标。这两个接口有了之后就需要判断在那里加了。</li>
</ul>  

通过分析源码可以看到有个很诡异的事情：
{% highlight html %}
-  (void)syncScrollView
{
    for (UIView* view in _pageController.view.subviews){
        if([view isKindOfClass:[UIScrollView class]])
        {
            _pageScrollView = (UIScrollView *)view;
            _pageScrollView.delegate = self;
        }
    }
}
{% endhighlight %}
_pageScrollView.delegate = self;可以整个代码里并没有实现ScrollDelegate的方法，我对比了一下RKSwipeBetweenViewControllers才发现原作者并没有copy这个delegate，原因可能是他的项目里当时不需要这个特性。
很显然我们需要实现这个ScrollDelegate，并在滑动过程中同步调用MBXPageSelectdViewOffset来跟新tabIndicator的X坐标：
{% highlight html %}
#pragma mark - scroll view delegate
//It extracts the xcoordinate from the center point and instructs the selection bar to move accordingly
-(void)scrollViewDidScroll:(UIScrollView *)scrollView{
    if ([self MBXDataDelegate]
        && [self.MBXDataDelegate respondsToSelector:@selector(MBXPageSelectdViewOffset:)]
        && [self.MBXDataDelegate respondsToSelector:@selector(MBXPageTabIndicatorStartOffset:)]) {
//        CGFloat offsetPercent = scrollView.contentOffset.x / scrollView.contentSize.width;
//        [self.MBXDataDelegate MBXPageSelectdViewOffset:offsetPercent];
//        NSLog(@"Scrolled percentage: %f   %f", scrollView.contentOffset.x,scrollView.contentSize.width);
        CGFloat xFromCenter = self.view.frame.size.width-scrollView.contentOffset.x; //%%% positive for right swipe, negative for left

        //%%% checks to see what page you are on and adjusts the xCoor accordingly.
        //i.e. if you're on the second page, it makes sure that the bar starts from the frame.origin.x of the
        //second tab instead of the beginning
        NSInteger xCoor = [self.MBXDataDelegate MBXPageTabIndicatorStartOffset:self.currentPageIndex];
        [self.MBXDataDelegate MBXPageSelectdViewOffset:(xCoor-xFromCenter/[self.viewControllerArray count])];;
    }
}
{% endhighlight %}

必须得承认，如果没有看RKSwipeBetweenViewControllers我到死都不可能完成这个方法。了解pageController这个容器的都知道，这个容器为了节省内容，会在他认为合适的时候把看不到的子controller移除。
这可真是个要命的特性，我刚开始的想法理所当然的认为scrollView.contentOffset.x/scrollView.contentSize.width就是滚动的比例，然后根据这个比例来更新咱们的tabIndicator。可是当我
把scrollView.contentOffset.x和scrollView.contentSize.width打印出来后我傻眼了，简直混乱的不要不要的，完全不对头。之后在一段时间的探索后我猛然发现，原来RKSwipeBetweenViewControllers有
同步跟新tabIndicator这个特性。。。
下面是最后的效果图：
<figure>
	<img src="/images/IOS/structure/MBXPageViewController/scrollDelegateAfter.gif" alt="项目实例">
	<figcaption>泛艺术客户端.</figcaption>
</figure>