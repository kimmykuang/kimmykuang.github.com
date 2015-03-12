---
layout: post
title: 使用Github Pages建立个人博客
category: blog
description: 很早就知道Github的Pages服务可以用来做个人/项目主页，最近尝试了一下，发现很不错，而且还支持个性域名绑定，所以我也捣腾创建了一个。
---

很早就知道Github的Pages服务可以用来做个人/项目主页，最近尝试了一下，发现很不错，而且还支持个性域名绑定，所以我也捣腾创建了一个。


个人认为使用Github建立个人博客需要满足以下几点“基础条件”

1. 基本的Git操作
2. 一些Html或Markdown（Markdown是一种轻量级标记语言，可以用易读写的标记方式来编写文档，这种文档也可以被转换成有效的HTML/XHTML文档，具体可以google了解下） & CSS和JS脚本的知识
3. Jekyll模板系统的一些知识
4. 坚持捣腾的决心和毅力

这里不介绍Git的基本流程了，有需要的话可以看以下几篇博文：

* [Git教程](http://www.liaoxuefeng.com/wiki/	0013739516305929606dd18361248578c67b8067c8c017b000)
	
* [Git详解教程](http://my.oschina.net/nalenwind/blog/141866)
	
* [Github使用教程(一)--搭建Github环境](http://blog.csdn.net/gavincook/article/details/11992827)

Github Pages服务的一个显而易见的好处是：无需自己搭建服务器，Github Pages服务后台采用的是Jekyll模板系统，类似于静态页面的发布系统，你自己用html或markdown标记语言编写的文件上传到Github后就可以发布出来了。

当然新手使用Github Pages会显得很不习惯，特别是如果你之前使用的是像Wordpress之类的博客系统的话。Wordpress发布文章后台有一个富文本编辑器，在线编辑文章就像在Word里一样，很简单、容易上手，但是使用Github Pages你想要发布文章的话，就需要自己编写出符合Jekyll系统能够解析的文件，否则Github是会给你发邮件的，告诉你文件哪几处地方不符合模板系统的语法导致无法编译。

Github Pages还是比较能够简单直观的，便于我记录学习心得、表达工作的历程和目标，所以目前Github Pages对我而言是一个十分完美的“个人博客解决方案”。

###建立Github Pages项目

这里假设你已经完成了Git的基本设置，对于Git也有了一些了解，那么可以开始使用Github提供的Pages服务了。

GitHub Pages分两种，一种是你的GitHub用户名建立的```username.github.io```这样的用户&组织页（站），另一种是依附项目的pages，一般项目的url是```username.github.io/your_reponame/```这种形式。我的博客使用的是第一种。

登录Github，创建一个和你的username同名的repo，名字叫做：```username.github.com```，这个资源库的名字和你将来用来访问博客的链接名字一样。

然后，你可以提交一个```index.html```到```username.github.com```这个repo的master分支中，第一次页面生效要等个几分钟。

生效之后就可以通过```username.github.com```来访问你刚才创建的页面了。

PS.```username.github.com```和```username.github.io```是一样的，只要保证你的代码仓库名和url一致就行了。

###Jekyll模板系统

to be updated...
