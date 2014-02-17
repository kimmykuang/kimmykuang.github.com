---
layout: blog
title: 本地Jekyll环境的搭建
category: blog
description: 本地搭建一个Jekyll环境来测试和预览Github Blog，不用再去一次次地提交到远程github代码仓库上去。
---
本地搭建一个Jekyll环境的好处是显而易见的：你不再需要重复地向github add / commit /push，在本地就可以预览和测试页面效果，是非常方便的一件事；你还可以一次性地调试好你的脚本、插件或者css效果，一口气写好多篇文章，本地确认没有问题后再提交到Github远程分支上。

这里稍微记录一下在Windows平台安装配置Jekyll环境的过程和当中发生的问题。

##环境搭建与配置
Jekyll是用ruby写的，所以先要配置ruby环境，由于我是初次接触ruby环境而且不是在Linux下，所以就不用二进制编译的方式了，直接下载ruby的安装包<a href="rubyinstaller.org" target="_blank">RubyInstaller</a>来配置了。记得安装的时候选上“Add Ruby executables to your PATH”（添加系统环境变量）。

##调试、补丁与问题