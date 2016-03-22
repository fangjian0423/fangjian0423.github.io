title: html页面左右滑动菜单效果的实现
date: 2015-10-29 21:15:13
tags:
- html
- css
categories:
- css
description: 最近要实现一个微信端页面左边弹出菜单的实现，效果如下 ...

---------------

## 正文 ##

最近要实现一个微信端页面左边弹出菜单的实现，效果如下：

{% iframe //jsfiddle.net/format/t0xda6zv/embedded/result,js,html,css 100% 500 %}

绿色部分是左菜单内容，高度填充满整个页面。且有滚动条，并且滚动条内容随着滚动条的滚动不会影响正文的滚动，正文内容的滚动不会影响左菜单的滚动。

下面说下自己实现这种效果的思路。

1.首先由于左右两边的滚动不影响双方，这就需要将菜单和内容的position设置为绝对定位，设置都需要滚动条: overflow: auto;  。

2.菜单的内容会覆盖正文的内容：所以菜单的z-index比正文要大。

3.给左边的菜单加点width动画，需要显示的时候设置width，需要隐藏的时候width设置为0即可。


如果不想把菜单的内容覆盖在正文内容上面，而是正文内容向右偏移菜单的宽度：

{% iframe //jsfiddle.net/format/of445qxn/embedded/result,js,html,css 100% 500 %}

这个效果与上一个效果一样，唯一的区别就是正文的z-index比菜单大，而且正文需要配置一个背景色。最后切换菜单的时候正文内容加点向右便宜的动画即可。


刚换了next主题.... 发现这个主题右边的sidebar也是这样的效果实现  →_→ 。 囧 。


## 基础小知识 ##

上面第二个例子中内容的z-index比菜单的z-index要大，而且正文需要配置一个背景色。为什么正文需要配置一个背景色呢？

**因为html中的元素没指定背景色的话，那说明这个元素的背景色是透明的**

比如下面这个效果，上面的内容没设置背景色，所以是透明的，虽然它的z-index比上面的块要大，但是还是显示了。


{% iframe //jsfiddle.net/format/yxseu05z/embedded/result,html 100% 500 %}


给上面的内容设置黄色的背景色就可以隐藏下面的内容的。


{% iframe //jsfiddle.net/format/kec1up6t/embedded/result,html 100% 500 %}





