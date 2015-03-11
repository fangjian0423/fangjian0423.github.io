title: Idea使用svn的问题
date: 2014-08-11 23:49:50
tags: [idea,svn]
description: Idea使用svn出现svn E204899 Cannot run program "svn"
---

**Idea使用svn出现svn: E204899: Cannot run program "svn"**

## 解决方法 ##
安装TortoiseSVN的时候**command line client tools(命令行客户端工具)**这个选项是默认不选的
![Alt text](http://format-blog-image.qiniudn.com/idea_svn.png)

然后idea使用svn的时候就是使用svn命令来完成一系列操作的。 没有安装命令行工具，然后就不行啦。

![Alt text](http://format-blog-image.qiniudn.com/idea_svn2.png)

安装的时候这个命令行选项选择安装就行。

![Alt text](http://format-blog-image.qiniudn.com/idea_svn3.png)

安装完成之后idea的Subversion这些设置全部不勾选。

PS：一开始自己也是没有选择安装命令行工具，但是之后安装svn命令行工具SlikSVN，然后在idea中设置使用，但是好像没有起作用～。 


