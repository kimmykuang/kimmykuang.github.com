---
layout: blog
title: 本地Jekyll环境的搭建
category: blog
description: 本地搭建一个Jekyll环境来测试和预览Github Blog，不用再去一次次地提交到远程github代码仓库上去。
---
本地搭建一个Jekyll环境的好处是显而易见的：你不再需要重复地向github add / commit /push，在本地就可以预览和测试页面效果，是非常方便的一件事；你还可以一次性地调试好你的脚本、插件或者css效果，一口气写好多篇文章，本地确认没有问题后再提交到Github远程分支上。

这里稍微记录一下在Windows平台安装配置Jekyll环境的过程和当中发生的问题。注意了，由于安装过程中发生了一些问题而且我也都记录了下来，所以本文不是一篇“教程”，直接照着我写的过程做的话是会绕点弯路的，但是修改起来也很简单，生命在于折腾嘛。

##配置ruby
Jekyll是用ruby写的，所以先要配置ruby环境，由于我是初次接触ruby环境而且不是在Linux下，所以就不用二进制编译的方式了，直接下载ruby的安装包<a href="http://rubyinstaller.org" target="_blank">RubyInstaller</a>来配置了，我将其安装在<code>D:\ruby</code>路径下。记得安装的时候选上“Add Ruby executables to your PATH”（添加系统环境变量）。

然后下载<a href="http://rubyinstaller.org/downloads/" target="_blank">Devkit</a>，这个是在win下安装环境必须要的开发配置，我将其解压在<code>D:\devkit</code>路径下。

解压好后打开<code>cmd</code>，进入到devkit目录下，输入：
<pre>
D:\devkit > ruby dk.rb init
D:\devkit > ruby dk.rb install
</pre>

没有出现错误提示的话就表明配置成功了。

##安装Jekyll
使用<code>gem</code>命令安装Jekyll，注意：修改gem命令代码源为<a href="http://ruby.taobao.org" target="_blank">ruby.taobao.org</a>，修改gem命令的代码源地址：
<pre>
$ gem sources --remove https://rubygems.org/
$ gem sources -a http://ruby.taobao.org/
$ gem sources -l
**** CURRENT SOURCES ****

http://ruby.taobao.org
#请确认只有 ruby.taobao.org

$ gem install jekyll
#开始安装jekyll

# ....
# 安装提示略过
# ....

$ jekyll -v
jekyll 1.4.3
</pre>

[注意]：当前默认的jekyll版本是1.4.3，其实这个版本是有点问题的，反正我安装后运行<code>jekyll serve</code>总是提示莫名格式错误，所以到这里还没有结束呢，请继续看下去。

##本地调试
安装好jekyll后就可以开始在本地搭建一个jekyll站点了，只要站点目录符合jekyll的基本结构。关于这部分内容可以参见我的另一篇文章：<a href="/github-blog.html" target="_blank">使用Github Pages建立个人博客</a> 中关于Jekyll模板系统的介绍。

我本地的jekyll站点目录在<code>D:\wamp\www\git_blog</code>路径下，而且目录都已经配置好，所以直接开启ekyll自带的server：

<pre>
D:\wamp\www\git_blog > jekyll serve
</pre>

结果启动serve连续出错：

<p><img src="/images/local-jekyll/git_blog_1.jpg" width="90%" /></p>

<p><img src="/images/local-jekyll/git_blog_2.jpg" width="90%" /></p>

最后在Stackoverflow上看到的，更换jekyll的版本至1.4.2，才能够显示“正常的错误”：字符不正确（...invalid byte sequence in GBK...），Jekyll默认的字符集应该是UTF-8，这个问题可以参照<a href="http://blog.jsfor.com/skill/2013/09/07/jekyll-local-structures-notes/" target="_blank">这篇博客</a>来解决，我也是参照的这篇。

如果发现找到的ruby源码里有些不一样，这是因为jekyll1.3.0源代码做出了一些修改：

<pre>
convertible.rb :
#self.content = File.read_with_options(File.join(base, name),
# merged_file_read_opts(opts))
self.content = File.read_with_options(File.join(base, name),:encoding=>"utf-8")

include.rb :
def source(file, context)
#File.read_with_options(file, file_read_opts(context))
File.read_with_options(file, :encoding=>"utf-8")
end
</pre>

解决了中文支持的情况后，再运行<code>jekyll serve</code>出现：

<p><img src="/images/local-jekyll/git_blog_3.jpg" width="90%" /></p>

好像没什么问题了嘛，不！

不知道为什么，<code>jekyll serve --auto/-w/--watch</code>（使jekyll默认自动regenerate的参数，否则本地修改了每次都要重新开启jekyll serve，实在太麻烦）都不行，继续上Stackoverflow上查看问题，发现他们都是建议安装jekyll的稳定版本1.2.1而非更高版本，所以我就只有去掉当前的1.4.2再安装1.2.1了，比较坑。。

<pre>
D:\wamp\www\git_blog > gem uninstall jekyll
D:\wamp\www\git_blog > gem install jekyll -v 1.2.1
#指定版本为1.2.1
</pre>

然后再重新设置一下中文字符的支持，运行<code>jekyll serve --watch</code>，终于没有错误了，而且更新的内容server也可以自动regenerate了：

<p><img src="/images/local-jekyll/git_blog_4.jpg" width="90%" /></p>

最后历经了多番磨砺终于可以在浏览器里通过<code>http://localhost:4000</code>来访问本地的github blog了。