---
layout: post
title: "Elixir on Phoenix"
modified:
categories: 
description: "Elixir on Phoenix学习起航，程序猿看世界."
tags: [Elixir on Phoenix]
image:
  feature: photo01.jpg
  credit:
  creditlink:
date: 2015-06-01T13:59:49+08:00
---

### 为什么要学习Elixir on Phoenix？
{% capture images %}
	/images/phoenix-vs-rails.png
{% endcapture %}
{% include gallery images=images caption="Elixir on Phoenix vs Ruby on Rails" cols=1 %}
这里有一篇<a href="http://www.littlelines.com/blog/2014/07/08/elixir-vs-ruby-showdown-phoenix-vs-rails/">帖子</a>，对比了Ruby on Rails和Elixir on Phoenix。
我学过的服务器语言和框架不多，在学校的时候做过一段时间SSH框架，当时写的时候感觉挺屌的， 需要配置一大堆的Spring action，还有hybridization，从数据库里面取出来就是对象，很方便。
进了生命里第一家公司后，我接触到了Ruby on Rails。然后才发现SSH框架简直就是老爷爷玩的东西。(PS:不谈什么效率，小公司目标受众顶天了100万级别)
十分钟写好一个五脏俱全的服务器，简直是敏捷开发的大杀器啊。很可惜的是Ruby on Rails的效率确实不高，所以当我看到了跟它很像的Elixir on Phoenix的时候 ，
我顿时觉得Elixir on Phoenix这个框架必然会取代Ruby on Rails甚至其他一些老牌框架和语言。因为Elixir是基于Erlang写的，有点像Node-js，都是面向过程的语言。

安装Elixir和Phoenix 步骤不赘述了 网上一搜一大堆.
程序员看世界，第一步，写一个Hello world程序。
{% highlight html %}
mix phoenix.new hello_phoenix
{% endhighlight %}
看起来跟Ruby on Rails一样一样的,需要注意的是，看它官网的说明，你需要先安装node-js，然后在提示
{% highlight html %}
Fetch and install dependencies? [Yn]
{% endhighlight %}
的时候，你必须输入Y，否则程序运行起来后会报错.
写到这里我发现我卡在了
{% highlight html %}
* running mix deps.get
{% endhighlight %}
我以为是网络问题，Ctrl+C强制退出去，删了挂上VPN重新来一遍，然后还是卡在那儿不动。。。 草,什么鬼。无奈我换了一种姿势，首先运行

{% highlight html %}
mix phoenix.new hello_phoenix
Fetch and install dependencies? [Yn] n
We are all set! Run your Phoenix application:
$ cd hello_world
$ mix deps.get
$ mix phoenix.server
{% endhighlight %}


按照提示一步步做，然后成了。。。。。。心中十万个草泥马奔腾而过，劳资满心欢喜的想学习这个框架，结果一上来就给劳资出了这么个Bug，后面还玩得动么。。。