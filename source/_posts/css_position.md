title: css position属性记录
date: 2015-03-01 19:23:54
tags:
- css
categories: css
description: html文档中是基于流式布局的，可以使用position属性修改元素的布局方式 ...
----------------

## 前言 ##

前几天在公司看到一个关于div disable的功能。 

html中一些input会有disable属性，但是div块这种元素是没有disable属性的，可以自己实现一个。

公司里要实现的div disable功能基于css的布局原理。好奇心使用研究了它的原理。它的原理是使用position属性完成的， 在这里记录一下。

## css position属性 ##

html文档中是基于流式布局的，可以使用position属性修改元素的布局方式。

position属性规定元素的定位类型。所有的元素都有position这个属性。

下面是它的一些值的定义以及解释：

static: 默认值。没有定位，出现在正常的流中，会忽略top，bottom，left，right或者z-index声明。

absolute: 绝对定位。元素的位置通过top, bottom, left, right规定。 **相对于static定位以外的第一个父元素进行定位**

relative: 相对定位。元素的位置通过top, bottom, left, right规定。**相对其正常位置进行定位**

fixed：固定定位。元素的位置通过top, bottom, left, right规定。**相对浏览器窗口进行定位**

## 各个position属性验证 ##

一些css代码如下：

	.box {
      border: 1px solid;
      width: 100px;
      height: 100px;
    }

### static 流式布局 默认值 ###

position为static表示遵从流式布局。

	<div class="box">盒子1</div>
	<div class="box">盒子2</div>

两个div盒子块。不设置postion属性，遵从流式布局，div的display是block，是个块，所以会换行。

因此两个盒子一上一下换行。

![All](http://format-blog-image.qiniudn.com/css_position1.png)

### relative 相对布局 ###

相对布局相对的是其正常位置。 也就是说相对布局在它原先的流式布局的位置上使用top, bottom, left, right进行定位。

	<div class="box">盒子1</div>
    <div class="box" style="position: relative; left: 20px; bottom: 20px;">盒子2</div>
    <div class="box">盒子3</div>
    
![](http://format-blog-image.qiniudn.com/css_position2.png) 

绿色部分表示left属性的生效，有20个像素的距离，间隔原先位置左边20个像素。 红色表示bottom属性的生效，有20个像素的距离，间隔原先位置下边20个像素。

盒子2不使用相对布局的话，效果如下：

	<div class="box">盒子1</div>
    <div class="box" style="position: static; left: 20px; bottom: 20px;">盒子2</div>
    <div class="box">盒子3</div>

![](http://format-blog-image.qiniudn.com/css_position3.png) 

static布局，top，bottom，left，right属性不生效。3个盒子都在流式布局上。

相对布局另外一个例子：

	<div class="box">
        <div style="border: 1px solid; width: 30px; height: 30px; position: relative; left: 20px; top: 20px;">
        </div>
    </div>
    
![](http://format-blog-image.qiniudn.com/css_position4.png) 

里面小的盒子使用了相对布局，在原先的位置上进行了便宜(原先位置在左上角跟外面那个盒子相交)。

### absolute 绝对布局 ###

绝对定位。相对其**static定位以外**的第一个父元素进行定位。

	<div class="box" style="width: 200px; height: 200px; background-color: yellow;">
        <div class="box" style="width: 150px; height: 150px; background-color: green; position: relative; left: 20px; top: 20px;">
          <div class="box" style="background-color: blue; position: absolute; top: 20px; left: 20px;">
          </div>
        </div>
    </div>

![](http://format-blog-image.qiniudn.com/css_position6.png) 

蓝色盒子是绝对布局，默认是在绿色盒子的左上角相交的，但是是绝对布局，在它的非static父元素(绿色盒子)上想左上各便宜20个像素。

绝对布局另外一个例子：
	
    <div class="box" style="width: 200px; height: 200px; background-color: yellow; position: relative;">
        <div class="box" style="width: 150px; height: 150px; background-color: green; position: static; margin-left: 20px; margin-top: 20px;">
          <div class="box" style="background-color: blue; position: absolute; top: 20px; left: 20px;">
          </div>
        </div>
    </div>

![](http://format-blog-image.qiniudn.com/css_position7.png) 

蓝色盒子是绝对布局，但是蓝色盒子的父元素绿色盒子是static布局，所以被无视了。但是黄色盒子是非stati布局(相对布局)所以以黄色盒子为标准，左上各便宜20个元素，绿色盒子是static布局，这里使用了margin-top和margin-left属性。所以蓝色盒子相交了。

### fixed 固定布局 ###

固定布局相对的是浏览器窗口。

比较简单，就不写了。 

### relative 和 absolute 的区别 ###

![](http://format-blog-image.qiniudn.com/css_position8.png)

	<div class="box" style="width: 400px; height: 400px; background-color: yellow; position: relative;">
        <div class="box" style="position: relative; top: 20px; left: 20px;">
        </div>
        <div class="box">
        </div>
        <div class="box" style="position: relative; top: 0; left: 5px;">
    	</div>
    </div>
    
**relative布局虽然偏离了它原先的流式布局位置。但是那块原本属于它的地盘别人无法侵占。**


	<div class="box" style="width: 400px; height: 400px; background-color: yellow; position: relative;">
        <div class="box" style="position: relative; top: 20px; left: 20px;">
        </div>
        <div class="box" style="position: absolute; background-color: red; top: 50px;">
        </div>
        <div class="box" style="position: relative;">
        </div>
    </div>

![](http://format-blog-image.qiniudn.com/css_position9.png)  

**absolute布局不会占用它原先的流式布局位置。**


## 总结 ##

position属性值总结：

static，默认值，流式布局。top, bottom, left, right属性值无效。

relative，相对布局，相对的是原先的流式布局位置。会占用原先的流式布局位置。top, bottom, left, right属性值有效。

absolute，绝对布局，相对的是非static父元素。不会占用原先的流式布局位置top, bottom, left, right属性值有效。

fixed，固定布局，相对的是浏览器窗口。top, bottom, left, right属性值有效。



