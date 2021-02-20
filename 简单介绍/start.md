#### 1.语法：nginx  -s  signal

##### 可执行文件支持的signal信号

- stop  快速停止
- reload  重新加载配置文件
- quit  平滑停止
- reopen  重新打开日志文件

```
nginx -s quit

nginx -s reload

```

##### signal信号也能通过unix工具kill发送给nginx master进程

```
kill -s QUIT 1245(nginx master pid)
```

#### 2.配置文件结构

- simple directives  简单指令
- block directives  块指令
- context  上下文（events, http, server, location)  *server in http, and location in server*
- main context  主上下文（events 和 http指令）

#### 3.static content

```
http {
    server {
        location / {
            root /data/www;
        }
        location /images/ {
            root /data;
        }
    }
}
/data/images/1.png 响应请求 http://FQDN:PORT/images/1.png
/data/www/test/1.html 响应请求 http://FQDN:PORT/test/1.html
```

如果有多个location，则优先选择前缀较长的location(/images/优先于/）

#### 4.proxy server

```
#应用服务器
server {
    listen 8080;
    root /data/up1; #默认root
    location / { 
    		#此处无root指令，默认使用上级root
    }
}
#代理服务器
server {
    location / {
        proxy_pass http://localhost:8080;
    }
    location /images/ {
        root /data;
    }
}
```

使用本地目录/data/images/中的文件处理图像请求，将所有其他请求发送到代理服务器 localhost:8080

```
server {
    location / {
        proxy_pass http://localhost:8080/;
    }
    location ~ \.(gif|jpg|png)$ {
        root /data/images;
    }
}
```

nginx首先检查指定前缀的location指令，记住最长的前缀对应的location，然后检查正则表达式。 如果网络请求与正则表达式匹配，则nginx会选择此location，否则，它将选择之前记住的location

#### 5.FastCGI proxy

```
server {
    location / {
        fastcgi_pass  localhost:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param QUERY_STRING    $query_string;
    }

    location ~ \.(gif|jpg|png)$ {
        root /data/images;
    }
}
```

在 PHP中，SCRIPT_FILENAME:参数用于确定脚本名称，QUERY_STRING参数用于传递请求参数

#### 6.Connection processing methods

nginx将自动选择处理连接请求的最高效的方法methods:

- select
- poll
- kqueue  ( FreeBSD 4.1+, OpenBSD 2.9+, NetBSD 2.0, and macOS)
- epoll  (Linux 2.6+)
- /dev/poll  (Solaris 7 11/99+, HP/UX 11.22+ (eventport), IRIX 6.5.15+, and Tru64 UNIX 5.1A+)
- eventport  (Solaris 10+ (due to known issues, it is recommended using the `/dev/poll` method instead))

```
Syntax:	use method;
Default:	—
Context:	events
```

#### 7.debugging log

验证nginx编译时配置是否支持debug，执行命令：

```
nginx -V
输出结果：configure arguments: --with-debug ...
```

重新定义日志而不指定调试级别将禁用调试日志

```
error_log /path/to/log debug;
http {
    server {
        error_log /path/to/log;
        ...
```

启用调试功能需要：

```
error_log /path/to/log debug;
http {
    server {
        error_log /path/to/log debug; #添加debug调试级别，或者注释此行
        ...
```

#### 8.nginx日志输出到syslog

```
error_log syslog:server=192.168.1.1 debug;
access_log syslog:server=unix:/var/log/nginx.sock,nohostname;
access_log syslog:server=[2001:db8::1]:12345,facility=local7,tag=nginx,severity=info combined;
```

参数解释：相关链接[RFC 3164](https://tools.ietf.org/html/rfc3164#section-4.1)

```
facility=string  #设置系统日志消息功能，参考RFC 3164 
nohostname  #禁用将“hostname”字段添加到系统日志消息头中
tag=string  #设置syslog消息的tag
severity=string  #设置日志级别，参考RFC 3164
```

#### 9.load balancer

负载均衡机制：

```
round-robin  #简单轮询 默认模式
least-connected  #最少活跃连接数
ip-hash  #基于client ip的会话保持
```

Nginx中的反向代理实现包括HTTP，HTTPS，FastCGI，uwsgi，SCGI，memcached和gRPC的负载平衡

##### 简单轮询

```
http {
    upstream myapp1 {
        server srv1.example.com;
        server srv2.example.com;
        server srv3.example.com;
    }
    server {
        listen 80;
        location / {
            proxy_pass http://myapp1;
        }
    }
}
```

##### 最少连接数

```
    upstream myapp1 {
        least_conn;
        server srv1.example.com;
        server srv2.example.com;
        server srv3.example.com;
    }
```

##### ip-hash

```
    upstream myapp1 {
        ip_hash;
        server srv1.example.com;
        server srv2.example.com;
        server srv3.example.com;
    }
```

##### 加权负载均衡

```
    upstream myapp1 {
        server srv1.example.com weight=3; #3
        server srv2.example.com;  #1
        server srv3.example.com;  #1
    }
```

##### 健康检查

```
upstream dynamic {
    server backend1.example.com      weight=5;
    server backend2.example.com:8080 fail_timeout=5s slow_start=30s;
    server 192.0.2.1                 max_fails=3;
    server backend3.example.com      resolve;
    server backend4.example.com      service=http resolve;
    server backup1.example.com:8080  backup;
    server backup2.example.com:8080  backup;
}
server {
    location / {
        proxy_pass http://dynamic;
        health_check;
    }
}
```

