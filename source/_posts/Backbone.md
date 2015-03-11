title: Backbone小记录
date: 2014-08-06 09:49:50
tags: [JavaScript,Backbone]
description: 记录学习Backbone中的一些知识

--------------
## 前言 ##
这两天看了下[Backbone.js](http://backbonejs.org/)的知识，大概了解了这个框架的一些知识。 写篇博客总结一下。

Backbone.js是一个web端javascript的轻量级MVC框架。为什么说是轻量级呢？因为它基于[underscore](http://underscorejs.org/)(基于jQuery)和[jQuery](http://jquery.com/)这2个框架，熟悉jQuery的小伙伴们可以快速入门Backbone.js。

## 为何使用Backbone ##
使用Backbone.js可以让你像写Java代码一样对js代码进行组织，比如定义类，类的属性、方法，这点非常重要。
比如传统的js开发是这样的：

	<input type="button" value="save" onclick="save()"/>
	<input type="button" value="add" onclick="add()"/>

	...

	<script type="text/javascript">
		function save() {
			...
		}
		function add() {
			...
		}
	</script>

页面内容一多，js代码会分布在页面的各个角落，维护、开发都很麻烦。

使用Backbone处理：

	var AppView = Backbone.View.extend({
	    el: "body",
	
	    // 事件
	    events: {
	        'click input[type=button][value=add]': 'add',
			'click input[type=button][value=save]': 'save'
	    },
	
	    add: function(event) {
	        ...
	    },

		save: function(event) {
	        ...
	    }

	});

	var appView = new AppView;

使用Backbone之后，所有js的代码都会在同一个地方，也就是对js代码进行了组织，我只需要管理这段js代码即可，前面button里的onclick事件就不需要单独写了。

## Backbone中的几个重要概念 ##
Backbone中有4个重要的概念，分别是Model，Collection，View，Router。

#### Model ####
Model是Javascript应用程序的核心，包括基础的数据以及围绕着这些数据的逻辑：数据转换、数据验证、访问控制。自定义的Model可以继承Backbone.Model。1个Person实体：

	var Person = Backbone.Model.extend({
		initialize: function() {
			// 构造实例的时候会调用这里
		},
		//默认属性
		defaults: {
			name: 'unknow',
			age: 0
		},
		validate: function(attributes) {
			if(!attributes.age || attributes.age < 0) {
				return "年龄小于0，出问题了";
			}
		},
		customMethod: function() {
			console.log("自定义方法");
		}
	});

	// 实例化1个Person对象
	var person = new Person;
	console.log(person.get('name')); // unknow
	console.log(person.get('age'));  // 0
	
	person.set('name', 'format');
	console.log(person.get('name')); // format

	person.set({name: 'james', age: 33});
	console.log(person.get('age') + ', ' + person.get('age')); // james, 33

	person.customMethod(); // 自定义方法

	//设置其他属性
	person.set({birth: '1977-12-15'});
	console.log(person.get('birth')); // 1977-12-15

	console.log(person.attributes); //{name: "james", age: 33, birth: "1977-12-15"}

	
	if(!person.set({age: '-1'})) { //set属性默认不验证，这边set age 0，可以设置进去
		console.log('设置age属性出错------11'); //这里不会打印
	}

	if(!person.set({age: '-1'}, {validate: true})) { //set 的时候加上validate: true， 让其验证，这里验证出错返回false
		console.log('设置age属性出错------22'); //这里会打印
	}
	
更加详细的资料请参考[Model](http://backbonejs.org/#Model)

#### Collection ####
Collection, 集合， 也就是多个Model，很好理解。

可以绑定'change', 'add', 'remove'事件到集合上以监听集合的数据状态。

	// 这里的代码依赖之前的Person这个Model
	var Family = Backbone.Collection.extend({
		model: Person,
		initialize: function() {
			//构造集合会调用这里 
		}
	});

	var father = new Person({name: 'father', age: 42});
	var mother = new Person({name: 'mother', age: 40});

	var family = new Family([father, mother]);

	family.bind('change', function(person) {
		console.log('family集合有数据改变了: ' + person.get('age'));
	});

	family.bind('add', function(person) {
		console.log('family集合有数据进来了: ' + person.get('name'));
	});

	family.bind('remove', function(person) {
		console.log('family集合有数据被删掉了: ' + person.get('name'));
	});

	// 触发add事件
	family.add(new Person({name: 'son', age: 0}));
	// 触发remove事件
	family.remove(father);
	// 触发change事件
	mother.set('age', 41);

	// 查询
	console.log(family.where({name: 'son'}));   [son]

	// 触发add事件
	family.add(father);

	// 比较器，排序使用
	family.comparator = 'age';
    family.sort();

	//根据age排序，并从对象里拿出name属性
    console.log(family.pluck('name')); // ["son", "mother", "father"]

更加详细的资料请参考[Collection](http://backbonejs.org/#Collection)

#### View ####
Backbone的View是用来显示你的model中的数据到页面的，同时它也可用来监听DOM上的事件然后做出响应。

html:

	<ul id="persons">
        
    </ul>
    <input type="button" value="add"/>

javascript:

	$(document).ready(function() {
            
        var AppView = Backbone.View.extend({
			// 注意这里的el是body，不然的话下面的事件绑定不了。下面事件的element是基于el进行查找的
            el: 'body',
        
            events: {
                'click input[type=button][value=add]': 'add',
                'click #persons li': 'remove'
            },
        
            add: function(event) {
                var name = window.prompt("请输入名字");
                this.$('#persons').append('<li>' + name + '</li>');
            },
    
            remove: function(event) {
                $(event.target).remove();
            }
        
        });
    
        var appView = new AppView;
    
    });

更加详细的资料请参考[View](http://backbonejs.org/#View)

#### Router ####

Router既控制器，用来监控一些#(锚)地址的访问情况。

	var AppRouter = Backbone.Router.extend({
        
	    routes: {
	        //* 代表全部， : 代表1个
	        "all/*action": "actionRoute",   // all/*action 会截获all/后面全部的地址
	        "posts/:id": "posts",            // :id 会截获
	        "trigger": "trigger"
	    },
	
	    actionRoute: function(actions) {
	        console.log("all/*action生效了，具体的地址: " + actions);
	    },
	
	    posts: function(id) {
	        console.log("posts/:id生效了，id: " + id);
	    },
	
	    trigger: function() {
	        //手动触发其他route, 参数trigger表示触发事件，如果为false，则只是url变化，并不会触发事件，replace表示url替换，而不是前进到这个url，意味着启用该参数，浏览器的history不会记录这个变动。
	        appRouter.navigate("/posts/" + 404, {trigger: true, replace: true});
	    }
	
	});
	
	var appRouter = new AppRouter;
	
	// 使用Router必须要有以下这个代码
	Backbone.history.start();

更加详细的资料请参考[Router](http://backbonejs.org/#Router)

## 总结 ##
总结了一下Backbone中4大核心概念的知识，接下来准备写一个使用Backbone的小项目。

参考资料：

[https://github.com/the5fire/backbonejs-learning-note](https://github.com/the5fire/backbonejs-learning-note)

[http://www.cnblogs.com/yexiaochai/archive/2013/07/27/3219402.html](http://www.cnblogs.com/yexiaochai/archive/2013/07/27/3219402.html)
