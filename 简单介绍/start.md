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

