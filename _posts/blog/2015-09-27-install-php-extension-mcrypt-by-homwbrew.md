---
layout: post
title: 使用homebrew安装php扩展mcrypt
category: blog
tag: homebrew, php扩展
description: homebrew安装php扩展
---


使用mac自带的php版本是5.5.27，最近安装laravel发现没有mcrypt扩展，本来我安装php扩展的方法就是进入到预先下载的php源码目录中，进入到/ext/目录下使用phpize动态地编译扩展然后安装。

但是这次进入到/ext/mcrypt后，依次执行:

	phpize
	
	./configure --with-php-config=/usr/bin/php-config
	
	make
	
	sudo make install
	
却报错了，提示```mcrypt.h not found```，目测是系统没有找到libmcrypt/mcrypt，所以这时有两种做法，第一种是自己下载```libmcrypt/mahsh/mcrypt```源码，编译安装，注意请把libmcrypt和mcrypt安装在默认的目录下，应该是```/usr/lib```下，如果安装在其他目录的话在编译mcrypt扩展的时候很可能会找不到，我目前发现是这样的；第二种就是用homebrew来装，会自行解析依赖，很方便。

至于我怎么知道把libmcrypt/mcrypt放在其他目录不行呢，因为我就是用homebrew安装了，然后进到php源码目录下企图编译扩展，结果反复提示找不到mcrypt.h，而且也没有没找到编译时能够指定libmcrypt安装路径的参数，如果有的话请告知我谢谢。

包括执行了如下操作也还是不行：

```sudo cp /usr/local/homebrew/Cellar/mcrypt/2.6.8/lib/libmcrypt.* /usr/lib```

后来一想干脆直接用homebrew来安装这个扩展，但是又能够不需要重新安装整个php，这样的话应该也是可以使用的。遂使用如下命令：

```brew install php55-mcrypt --without-homebrew-php```

等待一会儿后提示```/usr/local/homebrew/Cellar/php55-mcrypt/5.5.29: 3 files, 56K, built in 5.6 minutes```，homebrew确实很好用啊，它会帮你下载php源码，进入到ext目录下编译扩展文件，加了```--without-homebrew-php```参数就不再安装php本身了。下载下来的源码版本是5.5.29，我的是5.5.27，应该是可以使用编译好的扩展的。

把编译好的扩展文件拷贝过去试试：

```sudo cp /usr/local/homebrew/Cellar/php55-mcrypt/5.5.29/mcrypt.so  /usr/lib/php/extensions/no-debug-non-zts-20121212/```

```修改php.ini，添加extension=/usr/lib/php/extensions/no-debug-non-zts-20121212/mcrypt.so```

重启php-fpm & nginx，打印下看看```php -m | grep mcrypt```，ok，搞定。