title: css 盒子模型
date: 2015-03-07 20:51:12
tags:
- css
- boxmodel
categories:
- css
description: 盒子模型由margin, border, padding和元素内部的内容(Content)组成 ...
----------------

废话不多讲，直接进入主题。

所有的HTML元素都具有盒子模型的特性。

*盒子模型由margin, border, padding和元素内部的内容(Content)组成。*

margin指的是 border边框外的一块区域， 俗称外边距
padding指的是 包裹着content的一块区域， 俗称内边距
border指的是 包裹着padding的一块区域，俗称边框
Content指的是 盒子中的内容，文本和图片出现的地方，俗称元素内容

一个盒子模型从外到里一次是

Margin -> Border -> Padding -> Content

以下就是一个box model的样子。

![](http://format-blog-image.qiniudn.com/css_boxmodel1.gif)

接下来看一个div的css：

	div {
        width: 300px;
        padding: 25px;
        border: 25px solid navy;
        margin: 25px;
	}

这样一个div在页面上是这样的：

![](http://format-blog-image.qiniudn.com/css_boxmodel2.png)

通过chrome的console看到它的盒子模型：

![](http://format-blog-image.qiniudn.com/css_boxmodel3.png)

从外到里分析一下：

margin占了25个像素， border也是25个像素，padding还是25个像素，content内容占了300个像素。

这样的话整个box盒子的宽度占了：

300(Content宽度) + 25 \* 2(两个padding宽度，padding-left，padding-right) + 25 \* 2(两个border宽度，border-left, border-right) + 25 \* 2(两个margin宽度,margin-left, margin-right) = 450个像素


总结一下：

** css盒子模型由4部分组成，分别是margin(外边距)， border(边框)，padding(内边距)和Content(内容)。 盒子的宽度不仅仅是内容的宽度，还包括其他3部分的宽度。 盒子的高度不仅仅是内容的高度，还包括其他3部分的高度。**

** 盒子宽度 = 元素宽度 + 左内边距 + 右内边距 + 左边框 + 右边框 + 左外边距 + 右外边距 **

** 盒子高度 = 元素高度 + 上内边距 + 下内边距 + 上边框 + 下边框 + 上外边距 + 下外边距 **

