title: Hexo框架下Tinny主题中评论框的使用
date: 2014-07-30 16:58:18
tags: [Hexo,Tinny,duoshuo]
description: Hexo中使用Tinny主题后评论框的设置
---

##Hexo及Tinny介绍

[Hexo](http://hexo.io/)是一个快速、简单、功能强大的博客框架，基于[Node.js](http://www.nodejs.org/)。

[Tinny](https://github.com/hexojs/hexo/wiki/Themes)主题基于[Pacman](https://github.com/A-limon/pacman)主题，可以在Hexo中使用这些主题。


##具体操作

在Tinny中使用[多说评论框](http://duoshuo.com/)的通用代码的时候需要改掉 **站点ID**，**标题**，**文章地址** 这3个参数。

具体代码如下，站点ID对应 <%= page.path %>, 标题 <%= page.title %>, 文章地址 <%= page.permalink %>：
	
	<div class="ds-thread" data-thread-key="<%= page.path %>" data-title="<%= page.title %>" data-url="<%= page.permalink %>"></div>

这段代码(js代码已省略)放到Tinny主题的 layout/_partial/post/comment.ejs 文件中。
	
跟JSP好像 0 0. 但是为什么不用$呢，由于没有了解过Hexo，所以就不晓得了。

有时间看看Hexo的一些底层代码。