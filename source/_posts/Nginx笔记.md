---
title: Nginx简单使用
tag: 环境部署
categories: 运维技术
date: 2022/7/13 20:46:25
# index_img:  https://s2.loli.net/2023/11/26/GyIdrAsehtBv1ug.png
---
# Nginx 笔记

Nginx 是一款高性能的 web 反向代理服务器，常用与配置集群以及负载均衡

## 安装配置 Nginx

> 下载安装

**1.** 需要访问 [nginx官网](http://nginx.org/)下载最新压缩包，也可以到[历史版本](http://nginx.org/download/)中选择下载，这里选择的是`nginx-1.19.0.tar.gz`

**2.** 想要安装 nginx 需要一些依赖，远程连接你的服务器后：

~~~shell
# 安装 nginx 的依赖
yum -y install pcre-devel zlib-devel
# 如果你的服务器已安装 gcc 环境，那么这步就可以省略
yum -y install gcc automake autoconf libtool make
~~~

**3.** 环境到这里就准备完毕了，接下来准备安装 nginx，将之前下载好的压缩文件上传至服务器中，然后解压，上传步骤这里就不赘述了，直接开始解压安装

~~~shell
# 移动目录至local下，然后上传nginx安装包
cd /usr/local
# 解压上传的压缩文件
tar -zxvf nginx-1.19.0.tar.gz
# 移动目录到解压好的文件中
cd nginx-1.19.0
~~~

**4.** 接下来准备编译安装，在解压好的`nginx-1.19.0`目录中执行

~~~shell
# 检查压缩包，生成Makefile文件
./configure
# 执行安装方法
make && make install
~~~

**5.** 执行结束后移动目录到`/usr/local`会发现目录中多了一个`nginx`文件夹，文件夹内是这样的：

```shell
[root@iZbp1abebnnglcdjsmbh8tZ /]# cd /usr/local/nginx
[root@iZbp1abebnnglcdjsmbh8tZ nginx]# ls
conf  html  logs  sbin
```

这样 nginx 就安装就完成了，我们可以看到 nginx 目录下有四个文件夹：

- `conf`：存放 nginx 的配置文件
- `html`：基本的 html 页面，比如 nginx 的欢迎页 `index.html` 就在这个目录下
- `logs`：看名字就知道是存放运行日志的文件夹了
- `sbin`：该目录下只有一个`nginx`文件，我们所有的命令都是靠这个文件运行的



> 运行 nginx

运行 nginx 需要进入到 sbin 目录下执行 nginx 命令：

~~~shell
# 进入sbin目录
[root@iZbp1abebnnglcdjsmbh8tZ nginx]# cd sbin
# 启动 nginx
[root@iZbp1abebnnglcdjsmbh8tZ sbin]# ./nginx
~~~

如果没有任何返回就代表运行成功了，nginx 默认在 80 端口运行，访问一下就可以看到效果了



> 配置环境变量

我们每次操作 nginx 都要移动到 sbin 目录下然后还要加`./`才能调用命令，比较麻烦，这里将 nginx 配置到环境变量中方便日后操作：

~~~shell
# 编辑profile配置文件
[root@iZbp1abebnnglcdjsmbh8tZ sbin]# vim /etc/profile
# 在文件的最下面添加Nginx的环境变量，添加完成后 :wq 退出即可
export PATH=$PATH:/usr/local/nginx/sbin
# 刷新环境变量
[root@iZbp1abebnnglcdjsmbh8tZ sbin]# souece /etc/profile
~~~

操作完成后就可以在任何地方都可以调用 nginx 命令了



>Nginx 常用命令

~~~shell
# 启动nginx
nginx
# 快速停止nginx
nginx -s stop
# 完整有序的停止nginx
nginx -s quit
# 查看nginx是否运行
ps -ef | grep nginx
# 查看 nginx 版本
nginx -v
# 查看配置文件是否有误
nginx -t
# 重新载入配置
nginx -s reload
~~~



## 配置文件预览

在 nginx 目录下的 conf 目录中，有一个`nginx.conf`文件，它就是 nginx 的配置文件：

```shell
# 全局配置模块
user nobody;        # 配置用户或者组，默认为nobody
worker_processes 1; # 允许生成的进程数，默认为1
# 指定nginx进程运行文件存放地址
# pid /nginx/pid/nginx.pid;
# 指定日志路径，级别。全局、http、server都可以使用，级别以此为：
# debug|info|notice|warn|error|crit|alert|emerg
error_log log/error.log debug;
events {
    # events模块
    #use epoll;  # 事件驱动模型，select|poll|kqueue|epoll|resig|/dev/poll|eventport
    accept_mutex on;  # 设置网路连接序列化，防止惊群现象发生，默认为on
    multi_accept on;  # 设置一个进程是否同时接受多个网络连接，默认为off
    worker_connections 1024;  # 最大连接数，默认为512
}
http {
    include       mime.types;   #文件扩展名与文件类型映射表
    default_type  application/octet-stream; #默认文件类型，默认为text/plain
    #access_log off; #取消服务日志    
    log_format myFormat '$remote_addr–$remote_user [$time_local] $request $status $body_bytes_sent $http_referer $http_user_agent $http_x_forwarded_for'; #自定义格式
    access_log log/access.log myFormat;  #combined为日志格式的默认值
    sendfile on;   #允许sendfile方式传输文件，默认为off，可以在http块，server块，location块。
    sendfile_max_chunk 100k;  #每个进程每次调用传输数量不能大于设定的值，默认为0，即不设上限。
    keepalive_timeout 65;  #连接超时时间，默认为75s，可以在http，server，location块。

    upstream mysvr {   
      server 127.0.0.1:7878;
      server 192.168.10.121:3333 backup;  #热备
    }
    error_page 404 https://www.baidu.com; #错误页
    server {
        keepalive_requests 120; #单连接请求上限次数。
        listen       4545;   #监听端口
        server_name  127.0.0.1;   #监听地址       
        location  ~*^.+$ {       #请求的url过滤，正则匹配，~为区分大小写，~*为不区分大小写。
           #root path;  #根目录
           #index vv.txt;  #设置默认页
           proxy_pass  http://mysvr;  #请求转向mysvr 定义的服务器列表
           deny 127.0.0.1;  #拒绝的ip
           allow 172.18.5.54; #允许的ip           
        } 
    }
}
```





## 反向代理

在了解反向代理之前，首先要了解代理模式，最常用的就是科学上网，这里用两张图说明：

 ![](/img/nginx-01.jpg)

类似上面就是我们正常访问一个网站的请求，如图访问 Google 肯定是访问不了的，我们就需要挂代理：

![](/img/nginx-02.jpg)

这种代理服务器是安放在我们客户端这一方的，服务器并不知道我们的请求经过了代理，反向代理则刚好相反：

![](/img/nginx-03.jpg)

反向代理我们请求的仅仅是代理服务器，至于请求最后究竟发送到哪里客户端不知道



> 如何使用反向代理

在 nginx 的安装目录下，有一个`conf`目录，改目录下的`nginx.conf`配置文件中的 http 模块下，可以通过 server 设置反向代理，他的标准写法是这个样子的：

~~~shell
server {
	listen       80;
	server_name  www.baidu.com;
	location / {
	    proxy_pass  http://220.181.38.148/;
	    index       index.html;
	}
}
~~~

- server 表示一个监听的目标
  - listen [ 监听条件1 ] 可以是端口号，可以是 ip 地址，也可以是 ip 地址+端口号，也可以使用通配符 *
  - server_name [ 监听条件2 ] 可以是一个域名，也可以是多个域名 ( 逗号分隔 )，也可以使用通配符 * 
  - location [ 监听条件3 ] 后面跟着一个空格，接着是一个url，例如 /blog/，/shop/ 等等区分项目路径，可以使用通配符，也可以使用正则表达式
    - proxy_pass [ 目标地址 ]  符合监听条件后跳转的目标地址
    - index [ 默认首页 ] 跳转的目标项目路径，可以通过这里设置默认首页，一般不常用

- 关于配置具体参考网站：`https://www.cnblogs.com/ysocean/p/9392908.html`

在配置文件中可以有多个 server 出现，nginx 会从上到下依次监听，当符合某一个的时候就不会继续往下走。



> 反向代理可以干些什么

我在服务器上部署了两个应用要求必须都通过 80 端口直接访问，可是 80 端口只有一个，这个时候就可以通过反向代理来配置，例如

blog.hanzhe.site -> 访问博客网站 ( 博客网站在本地的8081端口 )

~~~
server {
	listen       localhost:80;
	server_name  blog.hanzhe.site;
	location / {
	    proxy_pass http://localhost:8081/;
	}
}
~~~

zhang.hanzhe.site -> 访问个人网站 ( 个人网站在本地的8082端口 )

~~~
server {
	listen       localhost:80;
	server_name  zhang.hanzhe.site;
	location / {
	    proxy_pass http://localhost:8082/;
	}
}
~~~

这样一来，当我访问 blog.hanzhe.site 的时候，访问的是博客网站，访问 zhang.hanzhe.site 的时候访问的是个人网站，都是 80 端口，但是对应本地确实两个不一样的端口



## 负载均衡

### 负载均衡的实现

> 什么是负载均衡

反向代理技术是 nginx 的特点之一，那么`负载均衡`就是第二大特点，什么是负载均衡？顾名思义，负载均衡就是讲服务器接收的请求分散开来，从而达到减缓服务器压力的一门技术

既然将请求分散开来，那么就肯定还要有服务器来接受这些分散出去的请求，所以就需要多台服务器进行配合，按照要求轮流来负责处理请求，那么将这些请求合理的分发出去的技术就叫做 负载均衡



> 实现负载均衡

还是老位置，nginx 的安装目录下的 conf 目录下的 nginx.conf 配置文件中使用 upstream 来完成负载均衡

upstream 后面跟着标识符作为名称，然后下面可书写 n 多个 server 指向目标地址

~~~shell
upstream blog {
	server 127.0.0.1:8080;
	server 127.0.0.1:8081;
	server 127.0.0.1:8082;
}
~~~

这里匹配到 blog.hanzhe.com:80 域名之后，直接移交给上面 blog 对应的负载均衡进行处理，会按照特定的规则来分发请求。

~~~
server {
	listen      80;
	server_name blog.hanzhe.com;
	location / {
		proxy_pass  http://blog;
	}
}
~~~



### 集中方式说明

在上面提到了负载均衡会按照特定的规则来向各个服务器中分发请求，那么这个规则究竟是什么，现在就来简单了解一下。

> 1 - 轮询规则

轮询规则是 nginx 默认的负载规则，当我们书写的格式像上面的代码一样，没有多余的修饰，那么他默认就是轮询规则

轮询规则，顾名思义，就是所有请求都按照时间顺序分发到不同的服务器上，期间如果有服务器突然死机，那么就会被踢出轮询队伍中。

> 2 - 权重规则 ( weight )

轮询规则是按照时间来依次分发请求的，而权重规则是按照服务器的权重比例来进行请求分发的。

~~~shell
# 实例配置
upstream blog {
	server  localhost:8080 weight=2;
	server  localhost:8081 weight=6;
	server  localhost:8082 weight=3;
}
~~~

在目标服务器地址后面跟上空格，追加1weight`关键字就可以设置权重比例了，对应的数字越大，被访问的频率也就越高，例如上面的配置，如果访问量较高的话，8080 和 8082 两个端口加一起的访问量可能还没有一个 8081 多，可以让高性能服务器拿到较高的值，适合多个服务器配置不一致时起到的均衡作用，这就是权重规则。

> 3 - 哈希规则 ( ip_hash )

该规则会根据客户端的 ip 地址进行 hash 计算后的结果进行分类，这样的好处是可以让客户端自始至终访问的都是同一台服务器，可以解决 session 的问题。

配置方法也十分简单，在 upstream 中添加关键字即可。

~~~shell
# 实例配置
upstream blog {
	ip_hash;
	server  localhost:8080;
	....
}
~~~

- ip_hash 还可以配合 weight 权重一起使用。

> 4 - 最少规则 ( least_conn )

看名字就可以轻松的看出来，该规则的特点就是将请求交给连接数最少的服务器进行处理，也就是所谓的均摊。

代码就不粘了，使用方法同上面一致，将关键字敲上即可。

> 5 - 延迟规则 ( fair )

按照服务器的响应时间来分配，两台服务器等待的情况下，延迟低的服务器优先分配，使用方法同上

---

*这里需要注意无论是 server 还是 upstream 都是在 http 模块下的，而且 upstream 要在 server 的前面，不然不起作用*



## 动静分离





## 高可用
