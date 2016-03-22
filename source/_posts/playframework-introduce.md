title: playframework简介
date: 2015-05-17 23:56:38
tags:
- scala
- playframework
categories:
- jvm
description: play的controller需要继承Controller，controller中的每个方法都是一个Action ...

----------------

好久没写博了，因为工作忙成狗了  →\_→  ， 还有一个原因就在最近还在看scala in action这本书。 in action系列的就不用多介绍了，像我这种英文渣渣都能看的懂。 →\_→。

几个web框架说明一下：

grails：工作上使用的框架，grails虽然开发也很快，但是是用groovy语言写的，不是很喜欢groovy。grails老是遇到一些莫名其妙的问题，之前还碰到过domain有个enum类型的属性，使用ordinal映射，但是数据库中字段的类型。domain做query查询，debug居然提示Variable没有找到。后来在try catch块内找到了相对正确一点的error message。而且groovy语法感觉设计地很糟糕，完全不能跟scala对比。
SpringMVC：不用多说了，还看过源码，还是不错的。当年写java的时候的第一选择。虽然现在还在写 →\_→。
Struts2：不想说了，不好用，不知道现在用的人还多不。
Playframework：java，scala都可以使用。跟grails一样，开发速度也很快。

## Playframework简介 ##

记录一下play常用的功能，做个笔记。

从MVC角度分别说明一下。

### Controller ###

play的controller需要继承Controller，controller中的每个方法都是一个Action。一个Action本质上就是一个 (play.api.mvc.Request => play.api.mvc.Result) 函数，这个函数处理请求并生成对应的结果。

1最简单的Controller：

	package controllers

    import play.api._
    import play.api.mvc._

    object Application extends Controller {

      def index = Action {
        Ok("This is index page")
      }

    }
    
index方法就是一个Action。Ok代表一个返回值为200的Response。

Action可以带request参数：

	def index = Action { request =>
    	Ok("request: " + request)
    }
    

request可以加上implicit，这样其他的api也可以使用：

	def index = Action { implicit request =>
    	Ok("request: " + request)
    }
    
响应json：

    val list = Action { implicit request =>
      val items = Item.findAll
      render {
        case Accepts.Html() => Ok(views.html.list(items))
        case Accepts.Json() => Ok(Json.toJson(items))
      }
    }
    
除了Ok返回，还有一些其他封装好的方法(这些方法都是在play.api.mvc.Results中定义的)：

	NotFound
    BadRequest
    InternalServerError
    Status
    TODO // def index(name: String) = TODO
    Redirect
    
还可以自定义header和response：

	def index = Action {
    	Result(
        	header = ResponseHeader(200, Map(CONTENT_TYPE -> "text/plain")),
            body = Enumerator("Hello World!".getBytes())
        )
    }
    
**controller中定义的方法如果想被http访问，需要在routes中定义路由规则和对应的controller中的方法**。 这点是我不大爽的地方，项目一大，routes中的配置内容也会变多。

	GET   /clients/:id          controllers.Clients.show(id: Long)
	GET   /items/$id<[0-9]+>    controllers.Items.show(id: Long)
    // 默认值
	GET   /clients              controllers.Clients.list(page: Int ?= 1)
    // Option类型的参数不需要一定传递
    GET   /api/list-all         controllers.Api.list(version: Option[String])


### View ###

play的模板引擎中可以写一些scala代码。

play的模板引擎会根据对应的模板文件，生成一个类，这个类有个apply方法。

比如views目录下有个index.scala.html文件，play会生成一个views.html.index类。

也可以使用views.html.index方法得到对应的html文本：

	val content = views.html.index(c, o)

一段简单的模板代码：

	@(customer: Customer, orders: List[Order])

    <h1>Welcome @customer.name!</h1>

    <ul>
    @for(order <- orders) {
      <li>@order.title</li>
    }
    </ul>

一般模板文件第一行会定义参数。然后接下来就是具体的逻辑。

play的模板引擎使用@来表示scala代码。之前例子的for循环就是使用了@。

还有if：

	@if(items.isEmpty) {
    	<h1>Nothing</h1>
    } else {
    	<h1><@items.size items/h1>
    }
    
模板中还可以定义函数：

	@display(product: Product) = {
      @product.name ($@product.price)
    }

    <ul>
    @for(product <- products) {
      @display(product)
    }
    </ul
    
模板的继承：

    @(title: String)(content: Html)
    <!DOCTYPE html>
    <html>
      <head>
        <title>@title</title>
      </head>
      <body>
        <section class="content">@content</section>
      </body>
    </html>

    @main(title = "Home") {

      <h1>Home page</h1>

    }

2个子模板：

    @(title: String)(sidebar: Html)(content: Html)
    <!DOCTYPE html>
    <html>
      <head>
        <title>@title</title>
      </head>
      <body>
        <section class="sidebar">@sidebar</section>
        <section class="content">@content</section>
      </body>
    </html>

    @main("Home") {
      <h1>Sidebar</h1>

    } {
      <h1>Home page</h1>

    }

或者：

    @sidebar = {
      <h1>Sidebar</h1>
    }

    @main("Home")(sidebar) {
      <h1>Home page</h1>

    }



### Model ###

play的Model使用的是Ebean这个ORM框架
