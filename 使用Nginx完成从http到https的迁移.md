+++
title = "使用Nginx完成从http到https的迁移"
date = 2018-09-04T18:13:52+08:00
tags = ["Nginx","http","https"]
draft = false
+++

## 前言  
以前我的站点是`http://www.zealllot.com/`,所以当用手机访问网站时，经常会被运营商流量劫持。比如用移动网打开，网页的右下角经常会出现一个移动商城的标，一个不小心便会点到它，然后就跳转到移动网页。这种情况十分讨厌，使得访问体验变得极差。也不知道法律什么时候可以对它产生约束。  
既然别人帮不了我，那就只能自己帮自己。怎样才可以不被运营商流量劫持呢？答案是使用`https`协议。
## https简介
引用wiki的话:  
>Hypertext Transfer Protocol Secure (HTTPS) is an extension of the Hypertext Transfer Protocol (HTTP) for secure communication over a computer network, and is widely used on the Internet.[1][2] In HTTPS, the communication protocol is encrypted using Transport Layer Security (TLS), or formerly, its predecessor, Secure Sockets Layer (SSL). The protocol is therefore also often referred to as HTTP over TLS,[3] or HTTP over SSL.

https和http的区别:
>HTTPS URLs begin with "https://" and use port 443 by default, where as HTTP URLs begin with "http://" and use port 80 by default.HTTP is not encrypted and is vulnerable to man-in-the-middle and eavesdropping attacks, which can let attackers gain access to website accounts and sensitive information, and modify webpages to inject malware or advertisements. HTTPS is designed to withstand such attacks and is considered secure against them (with the exception of older, deprecated versions of SSL).
    
https最主要的功能是对http的明文信息进行了加密，使得传输的信息不易被泄露、劫持。  
## https实现  
实现从http到https的迁移，只需要两部：
* 申请证书
* 配置服务器  
### 申请证书  
我是在阿里云申请的[免费证书][]，一个证书可以使用一年。品牌是`Symantec`，证书类型是`免费型DV SSL`，你可以找一下。点击立即购买，填完相关域名信息后，进度就会在进行中。我是过了三天才去查看证书是否签发成功的，实际签发时间应该更短。  
签发成功后，阿里云的证书管理界面，在你对应的证书栏点击下载。下载内容有两个文件，一个证书文件，一个key文件。

[免费证书]:https://common-buy.aliyun.com/?spm=5176.2020520163.cas.1.47d72b7afq7NL0&commodityCode=cas#/buy  
### 配置服务器  
服务器这端，如果你不使用`Nginx`等web服务器工具，那就需要改代码。  
比如go的原生`net/http`包中，使用http监听：  

    package main
    
    import (
    	"io"
    	"log"
    	"net/http"
    )
    
    func main() {
    	// Hello world, the web server
    
    	helloHandler := func(w http.ResponseWriter, req *http.Request) {
    		io.WriteString(w, "Hello, world!\n")
    	}
    
    	http.HandleFunc("/hello", helloHandler)
    	log.Fatal(http.ListenAndServe(":8080", nil))
    }
改成https：
    
    package main
    
    import (
    	"io"
    	"log"
    	"net/http"
    )
    
    func main() {
    	http.HandleFunc("/", func(w http.ResponseWriter, req *http.Request) {
    		io.WriteString(w, "Hello, TLS!\n")
    	})
    
    	// One can use generate_cert.go in crypto/tls to generate cert.pem and key.pem.
    	log.Printf("About to listen on 8443. Go to https://127.0.0.1:8443/")
    	err := http.ListenAndServeTLS(":8443", "cert.pem", "key.pem", nil)
    	log.Fatal(err)
    }  
改动代码，多少是会觉得有些麻烦的。所以，我选择用`Nginx`作反向代理。  
#### Nginx安装
>ubuntu  

    apt-get install nginx
>centos  

    yum install nginx 
只是使用了最基本的功能，对版本也没有什么要求，所以直接用包管理工具安装就行了。
#### 配置
Nginx的配置目录一般放在`/etc/nginx/`下

    root@VM-0-11-ubuntu:~# ls /etc/nginx/
    fastcgi.conf    koi-utf  mime.types  proxy_params  sites-available  snippets      win-utf
    conf.d  fastcgi_params  koi-win  nginx.conf  scgi_params   sites-enabled    uwsgi_params
可以看到，目录下有一个`conf.d`文件夹，专门用来放服务的配置。这里我们等会儿再说。  
  
首先，我们需要创建一个`cert`文件夹，专门用来放证书和key。
    
    mkdir cert
然后把证书文件和key文件放入文件夹。
然后进入服务配置文件夹配置服务。
    
    cd conf.d
创建一个你的服务的配置文件，比如我的`zealllot.conf`  
    
    touch zealllot.conf
在里面填入相关的配置信息，可以参考我的配置  
    
    root@VM-0-11-ubuntu:~# cat /etc/nginx/conf.d/zealllot.conf
    server {
        listen  80;
        server_name www.zealllot.com;
        rewrite ^(.*)$  https://$host$1 permanent;  #可以将http的请求redirect到https去
    }
    
    server {
        listen 443;
        server_name www.zealllot.com;
        ssl on;
        ssl_certificate   cert/[证书文件名].pem;
        ssl_certificate_key  cert/[key文件名].key;
        ssl_session_timeout 5m;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;
    
        location / {
            proxy_pass http://[需要反向代理的服务的IP]:[端口];  #对该IP和端口的服务进行反向代理
        }
    }
配置完毕后启动`Nginx`服务

    service nginx start
如果已经启动了，那么
    
    service nginx restart  
好了，到这里，http到https的迁移就完成了。
