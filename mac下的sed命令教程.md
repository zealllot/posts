+++
title = "mac下的sed命令教程"
date = 2018-08-16T11:39:35+08:00
tags = ["mac","sed","shell"]
draft = false
+++

## 前言
这里只是提到了几个常见的用法，如果使用情况很复杂，那么建议`man sed`，帮助手册肯定能帮助到你。
## 简介
sed全名叫stream editor，流编辑器，用程序的方式来编辑文本。sed命令在macOS和Linux中是不一样的，mac中是BSD版的sed，Linux中是GNU版的sed。同时，它们的命令用法也是不相同的。  
如果你是Linux系统，那么下面就不用看了，网上随便找个博客，sed的简单用法都会说的比较清楚。  
## 命令用法  
### 用法
先查看它的用法  

    $ sed --help             
    sed: illegal option -- -
    usage: sed script [-Ealn] [-i extension] [file ...]
           sed [-Ealn] [-i extension] [-e script] ... [-f script_file] ... [file ...]
这里`-i`后面接备份文件的扩展名，比如`.bak`,如果你不需要备份文件，那么可以`-i ''`，在mac中不能不写，如果把`-i`参数去掉，那么sed操作后的结果会在命令行打印，但是不会作用到文件中去，文件内容还是老样子。
### 示例
首先我在当前文件夹创建一个示例文件。`touch file`，然后填入内容。  

    $ cat file
    First sentence.
    Second sentence.
    Third sentence.
    Fourth sentence.
#### 插入  
在第一行插入`hello`,并且保存备份文件后缀为`.bak`。在第一行插入的表达式为`1i\`，注意这里必须要'反斜杠'，并且需要插入的语句要换行写，否则会报语法错误:`extra characters after \ at the end of i command`。  

    $ sed -i '.bak' '1i\
    hello
    ' file  
查看效果  

    $ ls
    file     file.bak
查看两个文件内容  

    $ cat file
    hello
    First sentence.
    Second sentence.
    Third sentence.
    Fourth sentence.  
    
    $ cat file.bak 
    First sentence.
    Second sentence.
    Third sentence.
    Fourth sentence.
如果你想在第一行前面插入，而不是另起一行  

    $ sed -i '.bak' '1i\
    hello' file
效果

    $ cat file
    helloFirst sentence.
    Second sentence.
    Third sentence.
    Fourth sentence.  
值得一提的是，插入两行并不能这样使用  

    $ sed -i '.bak' '1i\
    hello
    hello
    ' file
这样写会报错  

    sed: 3: "1i\
    hello
    hello
    ": extra characters at the end of h command
那么，如何插入多行？我们可以使用换行符。不像传统的`\n`作为换行符，Linux版的sed可以使用`\n`，但是mac版需要使用`\'$'\n`。  

    $ sed -i '.bak' '1i\
    insert 1\'$'\ninsert 2
    ' file
查看效果  

    $ cat file
    insert 1
    insert 2
    First sentence.
    Second sentence.
    Third sentence.
    Fourth sentence.
    
### 字符串替换
同样使用上面的示例文件，我们把*sentence*替换为*line*。字符串替换的表达式为`s/被替换的内容/替换的内容/`。替换的表达式就不需要像插入一样，不需要换行了。  

    $ sed -i '.bak' 's/sentence/line/' file  
查看效果  

    $ cat file
    First line.
    Second line.
    Third line.
    Fourth line.  
更多替换命令的详解，可以查看[左耳朵耗子的博客][]，他对sed命令的讲解是我见过最直观，最简洁明了的。  

[左耳朵耗子的博客]:https://coolshell.cn/articles/9104.html "左耳朵耗子的博客"
