---
layout: post
title: 《HTTP权威指南》读书笔记（二）
category: blog
tag: 读书笔记, HTTP权威指南, web server
description: web server干了哪些事儿
---

web服务器实现了HTTP和相关的TCP连接处理，管理着服务器资源，以及对于web服务器本身的配置，控制，扩展管理。

实际的web服务器会做以下七件事：

	1. 建立连接：接受一个客户端的请求活着将连接关闭
	
	2. 接收请求：读取一条http请求报文
	
	3. 处理请求：对请求报文解释，如读取请求header，判断请求需要请求哪些资源文件
	
	4. 访问资源：访问报文中请求的资源－－如果是有权限访问的话
	
	5. 构建响应：无论是访问资源成功或者失败，这里都开始构建一条响应的报文，带有正确的响应首部
	
	6. 发送响应：将响应回送给客户端
	
	7. 记录事务处理过程：记录日志，这一步很有必要，出错时可以跟踪错误日志，当访问有异常时可以观察access日志
	
当然以上七件事只是粗略的总结，在实际运行过程中还有许多细节需要web server来处理，比如客户端主机名/IP认证，出错处理，超时处理，各种mime资源映射以及与后台多种应用程序的合作：如客户端访问的不是一个静态资源而是动态内容index.php，web server会根据配置将一些动态内容的请求发送给后台的CGI接口，例如php的会发送给php-fpm进程来处理，再接受由CGI程序返回的结果。

在生产环境上web server的配置与优化是一件持之以恒的事，需要考虑到实际访问量，带宽，机器负载能力，代码是否“合格”等因素。

以下是《HTTP权威指南》一书中介绍的一个perl程序，代码很短但是可以实现接收客户端请求，打印request header，reponse header和实体的功能。

```
#!/usr/bin/perl

use Socket;
use Crap;
use FileHandle;

# (1) use port 8080 by default, unless overridden on command line
$port = (@ARGV ? $ARGV[0] : 8080);

# (2) create local TCP socket and set it to listen for connections
$proto = getprotobyname('tcp');
socket(S, PF_INET, SOCK_STREAM, $proto) || die;
setsockopt(S, SOL_SOCKET, SO_REUSEADDR, pack("l", 1)) || die;
bind(S, sockaddr_in($port, INADDR_ANY)) || die;
listen(S, SOMAXCONN) || die;

# (3) print a startup message
printf("    <<<Type-O-Serve Accepting on Port %d>>>\n\n", $port);

while(1)
{
    # (4) wait for connection C
    $cport_caddr = accept(C, S);
    ($cport, $caddr) = sockaddr_in($cport_caddr);
    C->autoflush(1);

    # (5) print who the connection is from
    $cname = gethostbyaddr($caddr, AF_INET);
    printf("    Request From '%s'>>>\n", $cname);

    # (6) read request msg until blank line, and print on screen
    while ($line = <C>)
    {
        print $line;
        if ($line =~ /^\r/) {
            last;
        }
    }

    # (7) prompt for response message, and input response lines,
    #     sending response lines to clinet, until solitary "."
    printf("    <<<Type Response Followed by '.'>>>\n");

    while ($line = <STDIN>)
    {
        $line =~ s/\r//;
        $line =~ s/\n//;
        if ($line =~ /^\./) {
            last;
        }
        print C $line . "\r\n";
    }
    close(C);
}
```

可以这样使用：```sudo perl type-o-serve.pl 8080```，这样这个程序就会监听8080端口，所以当访问```localhost:8080```时可以看到command打印出来的请求记录如下。

```
    Request From 'localhost'>>>
GET / HTTP/1.1
Host: localhost:8080
Connection: keep-alive
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_10_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/41.0.2272.76 Safari/537.36
Accept-Encoding: gzip, deflate, sdch
Accept-Language: zh-CN,zh;q=0.8,en;q=0.6

    <<<Type Response Followed by '.'>>>
```
