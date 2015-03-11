title: bootstrap学习系列之表格布局原理
date: 2015-03-08 22:07:24
tags: [bootstrap,css]
description: 介绍一下bootstrap的grid表格布局的原理
----------------

分析一下bootstrap中表格布局的原理。

自定义css:

	.border {
        margin-top: 40px;
        margin-ledt: 40px;
        border: 1px solid red;
        width: 300px;
    }

##表格布局基础##

页面：

	<div class="col-sm-12 border">
        <div class="col-sm-6">
            123
        </div>
        <div class="col-sm-6">
            456
        </div>
    </div>
    
效果：
![](http://format-blog-image.qiniudn.com/bootstrap-grid-theory1.png)

[表格布局的基础](http://fangjian0423.github.io/2015/01/11/bootstrap-grid/)已经分析过。

重点来看下一些样式的定义。

col-sm-**这些样式都有一些默认的值，分别有：

	width: 100%;  (因为是col-sm-12，除了width在表格布局中不一样，下面的值一样)
    
    float: left;
    
    position: relative;
    min-height: 1px;
    padding-right: 15px;
    padding-left: 15px;

最外层的width本来是100%(col-sm-12)， 在自定义的样式border中覆盖了，改成了width: 300px;

外部红色边框的div的padding-left和padding-right，15个像素。

![](http://format-blog-image.qiniudn.com/bootstrap-grid-theory2.png)

外部红色边框div的Content。

![](http://format-blog-image.qiniudn.com/bootstrap-grid-theory3.png)

内部div内容为123的padding-left和padding-right，15个像素。

![](http://format-blog-image.qiniudn.com/bootstrap-grid-theory4.png)

内部div内容为123的Content。

![](http://format-blog-image.qiniudn.com/bootstrap-grid-theory5.png)

外部红色边框的div的盒子模型。margin-top和margin-left都是40个像素，这是自定义的样式border指定的。border为1个像素的红色，也是自定义样式border指定的。padding-right和padding-left是bootstrap自带的样式col-sm-**定义的，都是15个像素。padding-top和padding-bottom没有任何地方指定，为0。内容Content的高度没有指定，以自字体高度为准，20个像素。内容的宽度为268个像素。

这个268像素是这样来的：

300(自定义样式border定义的宽度) - 1 \* 2(左右边距) - 15 \* 2(左右内边距) = 268

![](http://format-blog-image.qiniudn.com/bootstrap-grid-theory6.png)

内部两个样式为col-sm-6的div的盒子模型。

内容104个像素的宽度是这样算的：

150 - 1 - 15(150边距外部div宽度的一半，也就是col-sm-6的width: 50%算的，1是外部div的左边框，15是外部div的左内边框) - 15 \* 2(左右内边距) = 104

![](http://format-blog-image.qiniudn.com/bootstrap-grid-theory7.png)

##col-sm-offset样式##

    <div class="col-sm-12 border">
        <div class="col-sm-6 col-sm-offset-6">
            123
        </div>
    </div

效果：

![](http://format-blog-image.qiniudn.com/bootstrap-grid-theory8.png)

内部div的盒子模型。 由于使用了col-sm-offset-6。

这个样式属性如下：

	margin-left: 50%;
    
所以盒子模型的margin-left为134。

300 - 1 \* 2 - 15 \* 2 (外层红色边框div的Content) \* 50%  = 134

![](http://format-blog-image.qiniudn.com/bootstrap-grid-theory9.png)


##相对布局样式##

	<div class="col-sm-12 border">
        <div class="col-sm-6" style="left: 100px;">
            123
        </div>
    </div>

效果：

![](http://format-blog-image.qiniudn.com/bootstrap-grid-theory10.png)	

盒子模型。由于col-sm-**使用的是相对布局，left，right，top，bottom这些属性都是有效的。所以left: 100px生效了。内部div在原先的位置上向右偏移了100个像素。

![](http://format-blog-image.qiniudn.com/bootstrap-grid-theory11.png)	


