---
title: nginx配置
categories: 服务器
date: 2018-12-22 09:55:07
tags:
    - nginx
    - 网关
---
# nginx配置

## nginx常用编译参数

对于nginx，如果使用源码安装，在进行./configure编译的时候，需要为其指定一些参数。

- –prefix=PATH ： 指定nginx的安装目录。默认 /usr/local/nginx
- –conf-path=PATH ： 设置nginx.conf配置文件的路径。nginx允许使用不同的配置文件启动，通过命令行中的-c选项。默认为*prefix/conf/nginx.conf*
- –user=name： 设置nginx工作进程的用户。安装完成后，可以随时在nginx.conf配置文件更改user指令。默认的用户名是nobody。–group=name类似
- –with-pcre ： 设置PCRE库的源码路径，如果已通过yum方式安装，使用–with-pcre自动找到库文件。使用–with-pcre=PATH时，需要从PCRE网站下载pcre库的源码（版本4.4 – 8.30）并解压，剩下的就交给Nginx的./configure和make来完成。perl正则表达式使用在location指令和 ngx_http_rewrite_module模块中。
- –with-zlib=PATH ： 指定 zlib（版本1.1.3 – 1.2.5）的源码解压目录。在默认就启用的网络传输压缩模块ngx_http_gzip_module时需要使用zlib 。
- –with-http_ssl_module ： 使用https协议模块。默认情况下，该模块没有被构建。前提是openssl与openssl-devel已安装
- –with-http_stub_status_module ： 用来监控 Nginx 的当前状态
- –with-http_realip_module ： 通过这个模块允许我们改变客户端请求头中客户端IP地址值(例如X-Real-IP 或 X-Forwarded-For)，意义在于能够使得后台服务器记录原始客户端的IP地址
- –add-module=PATH ： 添加第三方外部模块，如nginx-sticky-module-ng或缓存模块。每次添加新的模块都要重新编译（Tengine可以在新加入module时无需重新编译）

 ## config基本配置概览

Nginx的配置文件默认在Nginx程序安装目录的conf目录下，其中核心文件为nginx.conf，当然你也可以自己写配置文件，然后在nginx.conf中引用。

nginx.conf配置具体信息如下：

```
#定义Nginx运行的用户和用户组
user www www;
#nginx进程数，建议设置为等于CPU总核心数。
worker_processes 8;
#全局错误日志定义类型，[ debug | info | notice | warn | error | crit ]
error_log /var/log/nginx/error.log info;
#进程文件
pid /var/run/nginx.pid;
#一个nginx进程打开的最多文件描述符数目，理论值应该是最多打开文件数（系统的值ulimit -n）与nginx进程数相除，但是nginx分配请求并不均匀，所以建议与ulimit -n的值保持一致。
worker_rlimit_nofile 65535;
#工作模式与连接数上限
events
{
    #参考事件模型，use [ kqueue | rtsig | epoll | /dev/poll | select | poll ]; epoll模型是Linux 2.6以上版本内核中的高性能网络I/O模型，如果跑在FreeBSD上面，就用kqueue模型。
	use epoll;
	#单个进程最大连接数（最大连接数=连接数*进程数）
	worker_connections 65535;
}

#设定http服务器
http
{
	include mime.types; #文件扩展名与文件类型映射表
	default_type application/octet-stream; #默认文件类型
	#charset utf-8; #默认编码
	server_names_hash_bucket_size 128; #服务器名字的hash表大小
	client_header_buffer_size 32k; #上传文件大小限制
	large_client_header_buffers 4 64k; #设定请求缓
	client_max_body_size 8m; #设定请求缓
	sendfile on; #开启高效文件传输模式，sendfile指令指定nginx是否调用sendfile函数来输出文件，对于普通应用设为 on，如果用来进行下载等应用磁盘IO重负载应用，可设置为off，以平衡磁盘与网络I/O处理速度，降低系统的负载。注意：如果图片显示不正常把这个改成off。
	autoindex on; #开启目录列表访问，合适下载服务器，默认关闭。
	tcp_nopush on; #防止网络阻塞
	tcp_nodelay on; #防止网络阻塞
	keepalive_timeout 120; #长连接超时时间，单位是秒
	
	#FastCGI相关参数是为了改善网站的性能：减少资源占用，提高访问速度。下面参数看字面意思都能理解。
	fastcgi_connect_timeout 300;
	fastcgi_send_timeout 300;
	fastcgi_read_timeout 300;
	fastcgi_buffer_size 64k;
	fastcgi_buffers 4 64k;
	fastcgi_busy_buffers_size 128k;
	fastcgi_temp_file_write_size 128k;

	#gzip模块设置
	gzip on; #开启gzip压缩输出
	gzip_min_length 1k; #最小压缩文件大小
	gzip_buffers 4 16k; #压缩缓冲区
	gzip_http_version 1.0; #压缩版本（默认1.1，前端如果是squid2.5请使用1.0）
	gzip_comp_level 2; #压缩等级
	gzip_types text/plain application/x-javascript text/css application/xml;
	#压缩类型，默认就已经包含text/html，所以下面就不用再写了，写上去也不会有问题，但是会有一个warn。
	gzip_vary on;
	#limit_zone crawler $binary_remote_addr 10m; #开启限制IP连接数的时候需要使用

	upstream blog.ha97.com {
	#upstream的负载均衡，weight是权重，可以根据机器配置定义权重。weigth参数表示权值，权值越高被分配到的几率越大。
	server 192.168.80.121:80 weight=3;
	server 192.168.80.122:80 weight=2;
	server 192.168.80.123:80 weight=3;
	}

	#虚拟主机的配置
	server
	{
		#监听端口
		listen 80;
		#域名可以有多个，用空格隔开
		server_name www.ha97.com ha97.com;
		index index.html index.htm index.php;
		root /data/www/ha97;
		location ~ .*.(php|php5)?$
		{
			fastcgi_pass 127.0.0.1:9000;
			fastcgi_index index.php;
			include fastcgi.conf;
		}
		#图片缓存时间设置
		location ~ .*.(gif|jpg|jpeg|png|bmp|swf)$
		{
			expires 10d;
		}
		#JS和CSS缓存时间设置
		location ~ .*.(js|css)?$
		{
			expires 1h;
		}
		#日志格式设定
		log_format access '$remote_addr - $remote_user [$time_local] "$request" '
		'$status $body_bytes_sent "$http_referer" '
		'"$http_user_agent" $http_x_forwarded_for';
		#定义本虚拟主机的访问日志
		access_log /var/log/nginx/ha97access.log access;

		#对 "/" 启用反向代理
		location / {
            proxy_pass http://127.0.0.1:88;
            proxy_redirect off;
            proxy_set_header X-Real-IP $remote_addr;
            #后端的Web服务器可以通过X-Forwarded-For获取用户真实IP
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            #以下是一些反向代理的配置，可选。
            proxy_set_header Host $host;
            client_max_body_size 10m; #允许客户端请求的最大单文件字节数
            client_body_buffer_size 128k; #缓冲区代理缓冲用户端请求的最大字节数，
            proxy_connect_timeout 90; #nginx跟后端服务器连接超时时间(代理连接超时)
            proxy_send_timeout 90; #后端服务器数据回传时间(代理发送超时)
            proxy_read_timeout 90; #连接成功后，后端服务器响应时间(代理接收超时)
            proxy_buffer_size 4k; #设置代理服务器（nginx）保存用户头信息的缓冲区大小
            proxy_buffers 4 32k; #proxy_buffers缓冲区，网页平均在32k以下的设置
            proxy_busy_buffers_size 64k; #高负荷下缓冲大小（proxy_buffers*2）
            proxy_temp_file_write_size 64k;
			#设定缓存文件夹大小，大于这个值，将从upstream服务器传
		}

		#设定查看Nginx状态的地址
		location /NginxStatus {
            stub_status on;
            access_log on;
            auth_basic "NginxStatus";
            auth_basic_user_file conf/htpasswd;
            #htpasswd文件的内容可以用apache提供的htpasswd工具来产生。
		}

		#本地动静分离反向代理配置
        #所有jsp的页面均交由tomcat或resin处理
        location ~ .(jsp|jspx|do)?$ {
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://127.0.0.1:8080;
        }
		#所有静态文件由nginx直接读取不经过tomcat或resin
		location ~ .*.	(htm|html|gif|jpg|jpeg|png|bmp|swf|ioc|rar|zip|txt|flv|mid|doc|ppt|pdf|xls|mp3|wma)$
		{ expires 15d; }
		location ~ .*.(js|css)?$
		{ expires 1h; }
		}
}
```

## nginx常用配置说明

### nginx日志文件配置

关于日志文件的配置主要有两个属性log_format、access_log、error_log。

#### log_format

log_format为日志格式，其语法为：

```
log_format name format {format ... }
```

其中，name表示日志名字，format表示定义的格式样式。

log_format有一个默认的、无需配置的combined日志格式：

```
log_format combined '$remote_addr-$remote_user [$time_local]'
"$request"$status $body_bytes_sent
 "$http_referer" "$http_user_agent"
```

在日志格式中的变量主要有：

| 参数                    | 说明                                                 | 示例                                                         |
| ----------------------- | ---------------------------------------------------- | ------------------------------------------------------------ |
| $remote_addr            | 反向代理服务器的IP地址                               | 127.0.0.1                                                    |
| $remote_user            | 远程客户端用户名称                                   | --                                                           |
| $time_local             | 访问时间与时区                                       | 18/Jul/2012:17:00:01 +0800                                   |
| $request                | 请求URL与HTTP协议                                    | GET /article-10000.html HTTP/1.1                             |
| $status                 | 请求状态，例如成功时状态为200，页面找不到时状态为404 | 200                                                          |
| $body_bytes_sent        | 发送客户端的文件主体内容大小                         | 1000                                                         |
| $upstream_status        | upstream状态                                         | 200                                                          |
| $http_referer           | 访问来源                                             | https://www.baidu.com/                                       |
| $http_user_agent        | 客户浏览器的相关信息                                 | Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 5.1; Trident/4.0; SV1; GTB7.0; .NET4.0C; |
| $ssl_protocol           | SSL协议版本                                          | TLSv1                                                        |
| $ssl_cipher             | 交换数据中的算法                                     | RC4-SHA                                                      |
| $upstream_addr          | 后台upstream的地址，即真正提供服务的主机地址         | 10.10.10.100:80                                              |
| $request_time           | 整个请求的总时间                                     | 0.205                                                        |
| $upstream_response_time | 请求过程中，upstream响应时间                         | 0.002                                                        |
| $http_host              | 请求地址，即浏览器中你输入的地址（IP或域名）         | www.baidu.com 192.168.1.1                                    |

#### access_log

access_log主要用于记录nginx访问日志。其语法如下：

```
access_log path [foramt {buffer-size | off} 
```

关闭日志方法：access_log off。

使用默认combined格式记录日志：access_log path。

使用自定义格式记录日志，首先定义一个log_format，如log_format customformat '具体配置'。

access_log path customformat  buffer=32k。

结尾的buffer代表缓冲区大小。

#### error_log 

与access_log不同的是，error_log主要记录nginx运行过程中的错误日志。其语法如下

```
error_log <FILE> <LEVEL>
```

其中参数含义如下：

- FILE：代表日志文件存放目录。
- LEVEL：错误日志级别。

常见的错误日志级别有：debug | info | notice | warn | error | crit | alert | emerg ，级别越高记录的错误信息越少。对于我们而言，一般用到的为warn，error,crit。

### nginx压缩输出

nginx压缩输出使用的技术为gzip(GNU-ZIP)压缩技术。经过gzip压缩后的页面大小可以变为原来的30%以下，压缩页面后可以降低用户在浏览页面时下载资源的时间，但gzip有个明显的去诶按，就是需要服务端与客户端同步支持gzip，即服务器压缩，浏览器解压。目前IE、Chrome等主流浏览器均具备解压gzip的功能。

nginx中的gzip指令如下：

开启或关闭gzip

```
gzip on|off
```

设置系统获取几个单位的缓存用于存储gzip的压缩结果数据流

```
gzip_buffers number size
```

设置gzip压缩比例

```
gzip_comp_level 1..9
```

设置允许压缩的页面最小字节数，当页面超过该数值时才进行压缩，其默认为0，即所有页面都进行压缩

```
gzip_min_length length
```

### nginx作为静态服务器

nginx其根本上是一个HTTP服务器，可以将服务器撒花姑娘的静态资源文件（如HTML、JS、Image）通过HTTP协议展现给客户端。

```
server {
    listen 80; # 端口号
    location / {
        root /usr/share/nginx/html; # 静态文件路径
    }
}
```

### nginx负载均衡与反向代理

#### 负载均衡

> 负载均衡就是由多台服务器以对称的方式组成一个服务器集合，每台服务器都具有等价的地位，都可以单独对外提供服务而无须其他服务器的辅助。通过某种负载分担技术，将外部送来的请求均匀分配到对称结构中的某一台服务器上，而接收到请求的服务器独立地回应客户的请求。

常见的复杂均衡主要有：

- 用户手动选择：通过用户手动选择线路。
- DNS轮询：为域名添加多个解析记录。
- 四/七层负载均衡设备：硬件实现负载均衡，如F5。软件实现为LVS。
- Nginx负载均衡

#### 反向代理

> 反向代理是指以代理服务器来接受Internet上的连接请求，然后将请求发给内部网路上的服务器，并将从服务器上得到的结果返回给Internet上请求连接的客户端，此时代理服务器对外就表现为一个服务器。

#### 具体实现

在nginx中，通过Upstream可以设置一组在proxy_pass和fastcgi_pass指令中使用的代理服务器，默认的负载均衡方式为轮询，Upstream模块中的Server指令用于指定后端服务器的名称和参数，服务器的名称可以是一个域名、一个IP地址、端口号或UNIX Socket。

而在server{}虚拟主机内，可以通过proxy_pass和fastcgi_pass指令设置进行反向代理的服务器集群。

proxy_set_header指令用于在向反向代理的后端Web服务器发起请求时添加指定的Header头信息。

当后端Web服务器上有多个基于域名的虚拟主机时，要通过添加Header头信息Host，用于指定请求的域名，这样后端Web服务器才能识别该方向代理访问请求由哪一个虚拟主机来处理。

在使用方向代理后，连接通过代理服务器链接到目标服务器后，如果在目标服务器中存在获取用户真实IP的代码（比如，Java、PHP等后台语言）就会失效，这时服务器获取的是代理服务器的IP。如果要获取真实IP，需要在nginx反向代理配置里添加Header头信息：X-Forwarded-For,让目标服务器能够获取用户的真实IP。

具体配置如下：

```
upstream baidu.com {
	# weight 设置权重
      server 127.0.0.1:8881 weight=3;
      server 127.0.0.1:8882;
      server 127.0.0.1:8888;
}
server{ 
    listen 80; 
    server_name baidu.com; 
    location / { 
        proxy_pass         http://baidu.com; 
        proxy_set_header   Host             $host; 
        #获取真实IP设置
        proxy_set_header   X-Real-IP        $remote_addr; 
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for; 
    } 
}
```