title: macos中jdk版本切换
date: 2015-03-30 00:51:12
tags:
- java
- mac
categories:
- java
description: 简单记录一下macos中的jdk版本切换问题 ...
----------------

简单记录一下macos中的jdk版本切换问题。

macos中安装jdk就有点麻烦... 不像ubuntu一样，ubuntu解压一下放到某个文件夹下，path指定一下就ok了，macos却比较搞。

我在macos上安装了[jEnv](http://www.jenv.be/)。

jEnv is a command line tool to help you forget how to set the JAVA_HOME environment variable

jEnv是一个命令行工具，用来帮助你忘记怎么设置JAVA_HOME环境变量的设置。 具体的install和jdk版本切换可以上官网看下，很easy的。

使用jEnv设置好jdk版本之后，比如要使用maven命令，可能会出现如下错误：

	Error: JAVA_HOME is not defined correctly.
      We cannot execute /usr/libexec/java_home/bin/java
    FAIL
    
这个时候可以使用

	jenv enable-plugin maven
    
这样的话就可以使用maven命令了。

同理，disable的话：

	jenv disable-plugin maven
    
同理，还有其他插件，比如：

	jenv enable-plugin grails
    
    

我的maven命令之所以出现错误是这样的：

一开始安装了jdk8。 后来换成了jdk7。 但是一开始装了jdk8，macos下/usr/libexec/java_home默认指向了jdk8，装了jdk7之后，用jEnv设置默认的jdk为1.7。但是/usr/libexec/java_home还是指向jdk1.8。 然后就会出现以上情况。

PS：

如果不安装jEnv的话，在环境变量中加入JAVA_HOME可能会解决问题。 比如还是有2个jdk版本，1.7和1.8。

使用jdk1.7版本，JAVA_HOME设置如下：

	/usr/libexec/java_home -v 1.7
   
使用jdk1.8版本，JAVA_HOME设置如下：
    
    /usr/libexec/java_home -v 1.8

我设置了貌似不行 → _ → 可能装了jEnv有影响。 暂时先不研究了 → _ →







