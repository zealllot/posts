+++
title = "supervisor+inotify 完成监控代码改动并重启程序"
date = 2018-08-17T17:14:45+08:00
tags = ["shell","linux","supervisor","inotify"]
draft = false
+++

## 前言
hugo的watch功能可以监控文件改动，但是对于content目录下的改动，它不会重启server，造成blog页面会有一个bug，不会加载新的tag。所以需要加一个监控文件改动，并重启server的功能。  
趁着这次改动，我决定再给博客加一个进程管理工具。
## Supervisor
supervisor是Unix-like系统中的一个进程管理工具。它可以在进程挂的时候重启程序，也可以在系统重启的时候重启程序，保证了服务的高可用性。  
### 安装
>mac  

    brew install supervisor
>centos  
    
    yum install supervisor
>ubuntu  
    
    apt-get install supervisor
### supervisor的配置  
它的配置文件一般是放在`/etc/supervisord.conf`，或者`/etc/supervisor/supervisord.conf`,可以自行查看。如果找不到，也可以`echo_supervisord_conf > /etc/supervisord.conf`来创建配置文件。  
下面附上上我服务器的配置文件

    root@VM-0-11-ubuntu:~# cat /etc/supervisor/supervisord.conf
    ; supervisor config file
    
    [unix_http_server]
    file=/var/run/supervisor.sock   ; (the path to the socket file)
    chmod=0700                       ; sockef file mode (default 0700)
    
    [supervisord]
    logfile=/var/log/supervisor/supervisord.log ; (main log file;default $CWD/supervisord.log)
    pidfile=/var/run/supervisord.pid ; (supervisord pidfile;default supervisord.pid)
    childlogdir=/var/log/supervisor            ; ('AUTO' child log dir, default $TEMP)
    
    ; the below section must remain in the config file for RPC
    ; (supervisorctl/web interface) to work, additional interfaces may be
    ; added by defining them in separate rpcinterface: sections
    [rpcinterface:supervisor]
    supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface
    
    [supervisorctl]
    serverurl=unix:///var/run/supervisor.sock ; use a unix:// URL  for a unix socket
    
    ; The [include] section can just contain the "files" setting.  This
    ; setting can list multiple files (separated by whitespace or
    ; newlines).  It can also contain wildcards.  The filenames are
    ; interpreted as relative to this file.  Included files *cannot*
    ; include files themselves.
    
    [include]
    files = /etc/supervisor/conf.d/*.conf  ;这里是你要添加的进程的配置文件，supervisor会读取这个目录下的文件，你也可以修改成自己的。  
### 管理进程的配置
从上方我的配置可以看出，include读取`/etc/supervisor/conf.d/`目录下所有`.conf`为后缀的文件，所以我可以创建我的进程配置文件为`blog.conf`。  
    
    root@VM-0-11-ubuntu:~# cat /etc/supervisor/conf.d/blog.conf
    [program:blog]                                                                  ;blog为程序名
    command=hugo server -b http://www.zealllot.com/ --bind="0.0.0.0" -v -p 80 -D    ;启动程序时的命令
    directory=/root/test                                                            ;启动程序时的目录
    autorestart=true                                                                ;是true的话程序会在supervisor启动的时候自动启动
    startsecs = 5                                                                   ;设成5秒表示，supervisor启动程序5秒并成功运行后，才可以认为程序start成功
    user = root                                                                     ;运行程序的用户
    redirect_stderr=true                                                            ;log redirect
    stdout_logfile = /root/test/log                                                 ;log 存放地址  
更多进程管理配置的解释请看[官方文档][]

[官方文档]:http://supervisord.org/configuration.html#program-x-section-values "supervisor官方文档"
### 进程管理
当配置文件准备结束后就可以启动进程了。
* supervisord，初始启动 Supervisord，启动、管理配置中设置的进程。
* supervisorctl stop blog，停止blog进程.
* supervisorctl start blog，启动blog进程
* supervisorctl restart blog，重启blog进程
* supervisorctl stop groupworker: ，重启所有属于名为 groupworker 这个分组的进程(start,restart 同理)
* supervisorctl stop all，停止全部进程，注：start、restart、stop 都不会载入最新的配置文件。
* supervisorctl reload，载入最新的配置文件，停止原有进程并按新的配置启动、管理所有进程。
* supervisorctl update，根据最新的配置文件，启动新配置或有改动的进程，配置没有改动的进程不会受影响而重启。  

这里命令中的`blog`是我在`管理进程配置`中所设置的`program`的名字，你也可以替换成你相应的程序名。
## inotify
想要监控文件系统，如果用轮询的方法会束手束脚，并且效果也不是很好。所以Linux系统推出了inotify，它是Linux系统的api。当对文件系统进行操作时，用inotify就可以接受到事件。  
我们这里就不介绍inotify的api了，我们可以直接使用inotify-tools这个命令行工具来完成相应的功能。
### 安装
>ubuntu  

    apt-get install inotify-tools
>centos  

    yum install inotify-tools  
### 使用
tools安装完之后会有两个命令，`inotifywait`和`inotifywatch`，这里我们只用了`inotifywait`。  
首先查看帮助  

    root@VM-0-11-ubuntu:~# inotifywait --help
    inotifywait 3.14
    Wait for a particular event on a file or set of files.
    Usage: inotifywait [ options ] file1 [ file2 ] [ file3 ] [ ... ]
    Options:
    	-h|--help     	Show this help text.
    	@<file>       	Exclude the specified file from being watched.
    	                排除不需要监视的文件，可以是相对路径，也可以是绝对路径。
    	--exclude <pattern>
    	              	Exclude all events on files matching the
    	              	extended regular expression <pattern>.
    	              	正则匹配需要排除的文件，大小写敏感。
    	--excludei <pattern>
    	              	Like --exclude but case insensitive.
    	              	正则匹配需要排除的文件，忽略大小写。
    	-m|--monitor  	Keep listening for events forever.  Without
    	              	this option, inotifywait will exit after one
    	              	event is received.
    	              	接收到一个事情而不退出，无限期地执行。默认的行为是接收到一个事情后立即退出。
    	-d|--daemon   	Same as --monitor, except run in the background
    	              	logging events to a file specified by --outfile.
    	              	Implies --syslog.
    	              	跟–monitor一样，除了是在后台运行，需要指定–outfile把事情输出到一个文件。也意味着使用了–syslog。
    	-r|--recursive	Watch directories recursively.
    	                监视一个目录下的所有子目录。
    	--fromfile <file>
    	              	Read files to watch from <file> or `-' for stdin.
    	              	从文件读取需要监视的文件或排除的文件，一个文件一行，排除的文件以@开头。
    	-o|--outfile <file>
    	              	Print events to <file> rather than stdout.
    	              	输出事情到一个文件而不是标准输出。
    	-s|--syslog   	Send errors to syslog rather than stderr.
    	                输出错误信息到系统日志
    	-q|--quiet    	Print less (only print events).
    	                指定一次，不会输出详细信息，指定二次，除了致命错误，不会输出任何信息。
    	-qq           	Print nothing (not even events).
    	--format <fmt>	Print using a specified printf-like format
    	              	string; read the man page for more details.
    	              	指定输出格式。
                        %w 表示发生事件的目录
                        %f 表示发生事件的文件
                        %e 表示发生的事件
                        %Xe 事件以“X”分隔
                        %T 显示由–timefmt定义的时间格式
    	--timefmt <fmt>	strftime-compatible format string for use with
    	              	%T in --format string.
    	              	指定时间格式，如（“%”后面的大小写代表不同的格式，如%y表示2位的年）
                        %Y-%m-%d  日期：2012-10-13
                        %H:%M:%S  时间：15:45:05
                        是否显示该参数指定的时间，取决于–format选项中是否指定了“%T”。
    	-c|--csv      	Print events in CSV format.
                        输出csv格式。
    	-t|--timeout <seconds>
    	              	When listening for a single event, time out after
    	              	waiting for an event for <seconds> seconds.
    	              	If <seconds> is 0, inotifywait will never time out.
    	              	设置超时时间，如果为0，则无限期地执行下去。
    	-e|--event <event1> [ -e|--event <event2> ... ]
                        Listen for specific event(s).  If omitted, all events are
                        listened for.
                        指定监视的事件。
    
    Exit status:
    	0  -  An event you asked to watch for was received.
    	1  -  An event you did not ask to watch for was received
    	      (usually delete_self or unmount), or some error occurred.
    	2  -  The --timeout option was given and no events occurred
    	      in the specified interval of time.
    
    Events:
    	access		file or directory contents were read
    	modify		file or directory contents were written
    	attrib		file or directory attributes changed
    	close_write	file or directory closed, after being opened in
    	           	writable mode
    	close_nowrite	file or directory closed, after being opened in
    	           	read-only mode
    	close		file or directory closed, regardless of read/write mode
    	open		file or directory opened
    	moved_to	file or directory moved to watched directory
    	moved_from	file or directory moved from watched directory
    	move		file or directory moved to or from watched directory
    	create		file or directory created within watched directory
    	delete		file or directory deleted within watched directory
    	delete_self	file or directory was deleted
    	unmount		file system containing file or directory unmounted  
我把帮助里的解释翻译成了中文，里面写的很清楚，用法也很简单，我们就可以直接开始了。  
我希望它可以监听我文件的创建、改动、删除，并且如果有这些事件，便重启我的blog服务。所以我便把命令写进了shell脚本  

    root@VM-0-11-ubuntu:~# cat watch.sh
    #!/bin/sh
    
    inotifywait -e modify,attrib,move,create,delete,delete_self,unmount -d -outfile watchlog -r content/ | while read event; do
        supervisorctl restart blog
    done
现在，我的服务器就可以自动监测`content`目录下的改动，并自动重启blog服务了。  
最后，再把我的监控程序也放进`supervisor`进行进程管理  
    
    root@VM-0-11-ubuntu:~# ls /etc/supervisor/conf.d/
    blog.conf  watch.conf
    
    root@VM-0-11-ubuntu:~# cat /etc/supervisor/conf.d/watch.conf
    [program:watch]
    command=./watch.sh
    directory=/root/test
    autorestart=true
    startsecs = 5
    user = root
然后执行`supervisorctl update`。至此，我的博客系统增加了监控重启功能，又增强了可用性。
    
