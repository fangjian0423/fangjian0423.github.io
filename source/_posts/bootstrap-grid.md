title: bootstrap学习系列之表格布局
date: 2015-01-11 00:07:24
tags: [bootstrap,css]
description: 介绍一下bootstrap的grid表格相关的知识
----------------

bootstrap框架不用多说了，本文来讲讲bootstrap中的grid相关的知识。

grid，也就是表格，可以用来展示一些表单的数据。

废话不多说，直接进入主题。 

## grid样式基本知识 ##

bootstrap中grid样式有几个基本的知识，我们先来看一下这几个基本的知识。

1. grid样式有12种比例设置，分别为1-12.  1代表宽度为1/12，2代表宽度为2/12，以此类推。
2. grid样式可以在多种设备中生效，且针对各个设备会有不同的样式。
	小于768像素的， 比如手机。 样式以.col-xs-为前缀
	大于等于768像素的，比如平板。 样式以.col-sm为前缀
    大于等于992像素的，比如笔记本。 样式以.col-md-为前缀
    大于等于1200像素的，比如大屏显示器。 样式以.col-lg-为前缀
3. grid样式的position都是relative，即相对布局。

具体的一些细节方便的知识请参考[bootstrap官网](http://getbootstrap.com/css/#grid-options)介绍。

## 例子 ##

由于使用markdown编写，无法进行展示，具体的效果读者可以自行尝试，当然也可以查看[我的学习笔记](https://github.com/fangjian0423/ReadingNotes/blob/master/bootstrap/grid/grid.html)。下面的说明一般都使用col-md-这种设备，这都是在我的PC上测试过的。

例子1：
这行占3列，每列占宽度的1/3。由于使用的是 md样式 这个样式只针对笔记本这个设备，该例子在笔记本上显示3列正常，但是在手机上看的话这里会有3行，因为手机端无法识别md这个样式。因此如果不区分设备，所有的设备都显示同样的效果，那应该使用col-xs-为前缀的样式。

	<div class="row">
        <div class="col-md-4">.col-md-4</div>
        <div class="col-md-4">.col-md-4</div>
        <div class="col-md-4">.col-md-4</div>
	</div
    
例子2：
这行有2列，笔记本上这2列各占一半，但是在手机上这2列各占1/3。 之前已经分析过，col-md-为前缀的代表笔记本设备，col-xs-为前缀的代表手机设备。
    
    <div class="row">
        <div class="col-md-6 col-xs-4">.col-md-6 .col-xs-4</div>
        <div class="col-md-6 col-xs-4">.col-md-6 .col-xs-4</div>
    </div>
    
例子3：
布局的嵌套, 嵌套内部也是以12为总和进行计算的。

	<div class="row">
        <div class="col-md-6">
            .col-md-6
            <div class="row">
                <div class="col-md-6">.col-md-6</div>
                <div class="col-md-6">.col-md-6</div>
            </div>
        </div>
        <div class="col-md-6">.col-md-6</div>
    </div>

例子4：
位置的偏移。 如果想在列与列之间有间隔，可以使用col-md-offset这个样式，这个样式也是分1-12种，分别代为1/12, 2/12...的margin-left样式。 有了这个样式，间隔的代码就不需要使用一个空的div隔开了。

	<div class="row">
        <div class="col-md-5">.col-md-5</div>
        <div class="col-md-5 col-md-offset-2">.col-md-5 .col-md-offset-2</div>
    </div>
    
col-md-offset-2的样式的margin-left， 还有另外两个类似的样式，分别是 col-md-push-2和col-md-pull-2。 分别代表left和right百分比。 由于col-md-5等样式使用的是相对布局，因此这些right，left样式会生效。














