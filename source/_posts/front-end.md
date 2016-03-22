title: 最近前端的一点总结
date: 2015-10-24 20:15:13
tags:
- css
- html
categories:
- css
description: 这段时间被拉去当做前端了，做了快1个多月了，也好久没写博客了。 做了两个项目的前端，这周是第二个项目的启动 ...

---------------

这段时间被拉去当做前端了，做了快1个多月了，也好久没写博客了。 做了两个项目的前端，这周是第二个项目的启动，而且这周简直是灾难的一周，基本上都是23点后下班的，有两天还是凌晨2点。悲剧。

做前端经验不是很多，这个月还是学到了一些前端的自己没掌握的知识。做个总结吧。

1.box-sizing属性。

box-sizing是css3引入的。有两个值，分别content-box和border-box。 默认为content-box。这个属性的作用是这样的：

比如我们定义一个div，宽度和高度都是50px。padding 5px， border: 5px。 那么这个div实际的宽度和高度是：

	50 + 2 * 5 + 2 * 5 = 70。
    
如果使用border-box的话，那么这个div的宽度还是50。 因为border-box会把padding和border都一起算到宽度和高度里面。所以使用border-box后div的高度和宽度为还是50。但是实际上真正显示内容的高度和宽度是 50 - 5 * 2 - 5 * 2 = 30。

直接来点实际的代码，使用content-box，也就是默认情况：

	<div style="background-color: blue; width: 100px; height: 100px;">
    	<div style="background-color: yellow; width: 50px; height: 50px; padding: 5px; border: 5px solid red; box-sizing: content-box;"></div>
    </div>
    
![](http://7x2wh6.com1.z0.glb.clouddn.com/content-box1.png)
![](http://7x2wh6.com1.z0.glb.clouddn.com/content-box2.png)

	<div style="background-color: blue; width: 100px; height: 100px;">
    	<div style="background-color: yellow; width: 50px; height: 50px; padding: 5px; border: 5px solid red; box-sizing: border-box;"></div>
    </div>
    
![](http://7x2wh6.com1.z0.glb.clouddn.com/border-box1.png)
![](http://7x2wh6.com1.z0.glb.clouddn.com/border-box2.png)

Bootstrap3中也大量使用了border-box。

2.white-space属性。

使用white-space是为了让多张图片在同一行展示，不会换行。

	<div style="width: 200px;">
        <img src="http://7x2wh6.com1.z0.glb.clouddn.com/border-box1.png" width="50px"/>
        <img src="http://7x2wh6.com1.z0.glb.clouddn.com/border-box1.png" width="50px"/>
        <img src="http://7x2wh6.com1.z0.glb.clouddn.com/border-box1.png" width="50px"/>
        <img src="http://7x2wh6.com1.z0.glb.clouddn.com/border-box1.png" width="50px"/>
        <img src="http://7x2wh6.com1.z0.glb.clouddn.com/border-box1.png" width="50px"/>
	</div>
    
![](http://7x2wh6.com1.z0.glb.clouddn.com/white-space1.png)
    
将外层的div改为：

	<div style="width: 200px; white-space: nowrap; overflow: scroll;">
    
![](http://7x2wh6.com1.z0.glb.clouddn.com/white-space2.png)

white-space设置为nowrap后，文本内容不会换行直到遇到br标签为止。


3.CSS3的动画。

一开始没有使用CSS3的动画，使用了JQuery的animate，结果发现页面动画有点卡，后来改成了CSS3的动画。

因为都是一些比较简单的动画，所以只能写下简单的动画属性了。

	<div style="width: 50px; height: 50px; background-color: yellow; transition: width .2s;">
    </div>
    
div的点击事件：

	$("div").click(function() {
        var $div = $(this);
        if($div.width() == 50) {
          $div.width("100");  
        } else {
          $div.width("50");  
        }
    });
    
动画还有延迟效果，这里就不举例了。


4.垂直居中

高度固定的垂直居中，设置line-height为容器高度，text-align为center即可：

	<div style="width: 100px; height: 100px; background-color: yellow; line-height: 100px; text-align: center;">
  	  我居中了
    </div>
    
高度不固定的垂直居中：

	html, body {
      height: 100%;
    }
    body {
      display: -webkit-box;
      display: -webkit-flex;

      display: -moz-box;
      display: -moz-flex;

      display: -ms-flexbox;

      display: flex;

      /* 水平居中*/
      -webkit-box-align: center;
      -moz-box-align: center;
      -ms-flex-pack:center;/* IE 10 */

      -webkit-justify-content: center;
      -moz-justify-content: center;
      justify-content: center;/* IE 11+,Firefox 22+,Chrome 29+,Opera 12.1*/

      /* 垂直居中 */
      -webkit-box-pack: center;
      -moz-box-pack: center;
      -ms-flex-align:center;/* IE 10 */

      -webkit-align-items: center;
      -moz-align-items: center;
      align-items: center;
    }
    
    ...
    <body>
    	垂直居中
    </body>
	...

还有其他的一些比如绝对定位，固定定位，相对定位，overflow等问题就不一一举例了。 估计之后还是得做前端的一些工作，到时候用到了一些新内容的话还会继续更新前端相关的博客的。
