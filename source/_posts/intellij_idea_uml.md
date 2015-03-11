title: Intellij Idea中一个非常牛逼的UML图功能
date: 2014-10-31 10:40:05
tags: [Intellij,Idea,UML]
description: Intellij Idea中一个非常牛逼的UML图功能，看开源框架架构的时候非常有用~
---



今天一来公司，旁边leader立马给我炫耀了intellij中一个非常牛逼的功能，先上效果图。
![](http://format-blog-image.qiniudn.com/intellij_idea_uml2.jpg)

于是立马想到了上篇博客mybatis自己画的uml图：
![](http://format-blog-image.qiniudn.com/intellij_idea_uml1.jpg)

我靠！。 intellij内部有自动生成uml图的功能，我自己还去画了！  简直就是弱爆了。！

于是点开keymap，搜索uml：
![](http://format-blog-image.qiniudn.com/intellij_idea_uml4.jpg)

这里uml图的生成是由子类找父类的，如果要找出父类的所有子类，需要手动show implementations，然后add要查看的子类。

同时还可以Show Categories -> Field，找出类的所有属性：
![](http://format-blog-image.qiniudn.com/intellij_idea_uml3.jpg)


nice！ 有木有发现逼格瞬间就高了！。
以后看源码架构方面的话就方便多了， 噢耶！