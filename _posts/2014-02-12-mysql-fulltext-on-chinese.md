---
layout: blog
title: MySQL数据库中文全文检索
category: blog
description: MySQL对于英文的全文检索可以很方便地用SELECT..MATCH..AGAINST语句，但是对于不以空格分隔的中文词语，MySQL一直以来都是不支持的，所以需要有一种方法来对中文进行分词，再对分词的数据进行全文检索。
---
希望MySQL能够支持中文全文检索的问题演变成了中文分词的问题，网上有很多的解决方案，有一些是基于MySQL插件的，有一些是基于算法来分隔单词的，我在项目（在线简历，需要在中后台有一个全文检索简历的功能）中使用的是由hightman开发的<a href="http://www.xunsearch.com/scws/" target="_blank">SCWS</a>。这是一种基于词库的分词方法，并且使用了SCWS的PHP扩展模块方式。

对于使用SCWS，有几点评估：

- 1.安装、使用便捷，对PHP+MySQL只要将SCWS作为PHP扩展安装在服务器上既可以使用scws类来对字符串进行分词。
- 2.分词准确率比较高，官方数据是准确率95%以上，实际测试下来也基本符合要求。

##安装scws

<pre>
#获取scws-1.2.2源代码，下载后可以在C/C++中使用
$ > wget http://www.xunsearch.com/scws/down/scws-1.2.2.tar.bz2
#解压缩
$ > tar -xvjf scws-1.2.2.tar.bz2
#进入目录
$ > cd scws-1.2.2
#执行配置脚本和编译
$ > ./configure --prefix=/usr/local/;make;make && install
#查看是否安装成功
$ > ls -al /usr/local/scws/lib/libscws.la
$ > /usr/local/scws/bin/scws -h
#安装两个中文词典，分别对应gbk和utf8字符集
$ > cd /usr/local/scws/etc
$ > wget http://www.xunsearch.com/scws/down/scws-dict-chs-gbk.tar.bz2
$ > wget http://www.xunsearch.com/scws/down/scws-dict-chs-utf8.tar.bz2
$ > tar xvjf scws-dict-chs-gbk.tar.bz2
$ > tar xvjf scws-dict-chs-utf8.tar.bz2
</pre>

但是需要把scws作为PHP扩展模块安装，还需要预先配置<code>autoconf</code>、<code>automake</code>和php的<code>phpize</code>，phpize一般是在php安装配置的时候就装好了。

##安装autoconf和automake

<pre>
#安装autoconf

$ > wget http://mirrors.kernel.org/gnu/autoconf/autoconf-2.65.tar.gz
$ > tar -xzvf autoconf-2.65.tar.gz
$ > cd autoconf-2.65
$ > ./configure --prefix=/usr/local
$ > make && make install
$ > cd ..

#安装automake

$ > wget http://mirrors.kernel.org/gnu/automake/automake-1.11.tar.gz
$ > tar xzvf automake-1.11.tar.gz
$ > cd automake-1.11
$ > ./configure --prefix=/usr/local
$ > make && make install
$ > cd ..
</pre>

##安装PHP扩展

<pre>
$ > cd SCWS-1.2.2/phpext
$ > phpize
$ > .configure --with-php-config=PHP_HOME/bin/php-config
$ > make && make install
</pre>

其中PHP_HOME是你的服务器上的php目录。再将上面生成的<code>/sur/local/scws-1.2.2/phpext/modules/scws.so</code>拷贝到php的扩展目录中，并且编辑php.ini，添加如下选项：

<pre>
[SCWS]
extension = scws.so
scws.default.charset = utf8
scws.default.fpath = /usr/local/scws/etc
</pre>

##验证测试PHP扩展安装

<pre>
$ > cd scws-1.2.2/phpext
$ > php scws_test.php
</pre>

输出如下表示成功：

- Test[1] ... PASS!
- Test[2] ... PASS!
- ...

下面可以写php来测试分词的效果了，实际体验一下，如果有本地测试服务器的话可以先在服务器上用脚本生成好测试用的MySQL数据库和表，批量生成一些中文字符数据存储。

一般来说通常的解决方案是建两站表，如：db_userinfo和db_userinfo_index，db_userinfo存储引擎可以使用InnoDB，而作为检索表db_userinfo_index就是MyISAM的存储引擎。在db_userinfo中存储的是raw data，为分词和编码的数据；在db_userinfo_index中存储的是经过scws分词以后再<code>urlencode</code>编码过的数据，使用urlencode编码是为了防止MySQL的<code>ft_min_word_len</code>默认为4的影响，把中文字符变长，否则两个字符的汉字查询是无法全文检索的。

简单分词测试：
<pre>
	
        $input = "测试一下分词的效果如何，还有英文字符串存在如：hello world！再测试一下特殊字符的分割情况：哈哈%嘻嘻&*(如何)。";
        $output = "";
        $data = array();
        $encoded_data = "";
        $sc = scws_new();
        if($sc){
                $tmp = "";
                //分词
                $sc->set_charset('utf8');
                $sc->set_ignore(false);
                $sc->send_text($input);
                while($tmp = $sc->get_result()){
                        foreach($tmp as $item){
                                $output .= $item['word'].' ';
                        }
                }
                $sc->close();
                var_dump($output);
                //去除重复项
                $data = array_filter(explode(" ", $output));
                $data = array_flip(array_flip($data));
                var_dump($data);
                //编码
                foreach($data as $d){
                        if(strlen($d) > 1){
                                $encoded_data .= str_replace('%','',urlencode($d)).' ';
                        }
                }
                var_dump($encoded_data);
        }else{
                exit("初始化scws对象失败");
        }
	
</pre>