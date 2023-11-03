# 初学者指南

### 基础配置

- 默认工作目录：`/var/www/html/`，`/usr/share/nginx/html`（取决于安装的版本）

- 默认配置文件 `nginx.conf`，位于`/usr/local/nginx/conf`，`/etc/nginx`或`/usr/local/etc/nginx`。（取决于安装的版本）
- `nginx.conf`会在nginx启动时加载其安装目录下`sites-enabled`目录下的全部文件和`conf.d`目录下以.conf结尾的文件。
- 为了更方便的进行模块化配置，我们通常将网站的配置文件放在nginx安装目录下的`sites-available`文件夹中并将其软连接到`sites-enabled`目录下。

### 命令

`nginx -s [signal]`，nginx运行时可通过该命令向其发信，使nginx作出对应的动作。其中signal可选值如下：

- stop 停止nginx服务
- quit 关闭nginx进程
- reload 重启nginx进程并重新加载配置文件
- reopen 重新打开日志文件，主要用于日志分割

### 配置文件结构

nginx的配置文件由指令模块组成。指令分为简单指令和块指令。

- 简单指令由指令名和参数组成，以空格分割，以分号`;`结尾。
- 块指令是由中括号`{}`包裹的一组指令的集合，如果块指令中包含了如`events`，`http`，`server`和`location`等指令，则称这条块指令为上下文。
- 井号`#`后面的内容视为注释。

### 配置案例

- 路径映射

配置完成后nginx将项目根目录`/`配置于磁盘`/data/www`目录下，将项目静态资源目录配置于磁盘`/data/staticx`目录下：

```nginx
http {
  server {
    location / {
      root /data/www;
    }

    location /static/ {
      root /data/static;
    }
  }
}
```

使用该配置时，假定域名为localhost，则用户访问如<http://localhost/static/favicon.ico>时将于硬盘的`/data/static`目录下寻找favicon.ico文件，未找到时返回404。访问<http://localhost/static/js/main.js>时将于硬盘的`/data/static`目录下寻找js文件夹，并在其中寻找main.js文件。访问其他路径则在硬盘的`/data/www`目录下寻找对应文件。

location指令的路径配置中，可以使用正则表达式。它需要以`～`开头。nginx将优先选择正则表达式的配置来提供服务。如：

```nginx
location ~ \.(gif|jpg|png)$ {
  root /data/images;
}
```

上述配置将匹配以.gif .jpg .png结尾的请求。

- 正向代理

本案例在同一个nginx实例上部署两个服务。配置完成后nginx会将请求转发到设定好的服务器上。

```nginx
http {
  server {
    listen 8080;
    root /data/up1;

    location / {
    }
  }
  server {
    location / {
      proxy_pass http://localhost:8080/;
    }

    location ~ \.(gif|jpg|png)$ {
      root /data/images;
    }
  }
}
```

第一个块指令将监听8080端口，并将项目根目录`/`配置于磁盘`/data/up1`目录下。

第二个块指令未制定监听端口，故默认监听80端口。按照正则优先的规则，若有以.gif .jpg .png结尾的请求，将于本机磁盘`/data/images`目录下寻找对应文件，其他请求将被转发至<http://localhost:8080>主机。

注意：`proxy_pass`指令的参数可以制定代理服务器的协议、主机和端口，在本例中，我们制定了http协议，localhost主机，8080端口。（即<http://localhost:8080>）也可以设置为由`upstream`定义好的轮询服务器组。

- 反向代理

本案例配置完成后，访问<http://localhost>时将由127.0.0.1:8080向客户提供服务。

```nginx
http {
  upstream really_server {  
    server 127.0.0.1:8080 weight=1;  
  }
	server {  
    listen       80;  
    server_name  localhost;    
    location / {  
      proxy_pass   http://really_server;
      index  index.html index.htm;  
    }       
  }  
}
```

先通过`upstream`指令定义一个轮询服务器组，其中可通过`server`指令配置服务器，通过可选参数`weight`设置权重。在配置多个服务器的情况下，权重越高，则轮询到的概率越大。

然后通过配置`server`块，监听80端口并设置服务器名为localhost。当nginx收到客户端的请求时，若满足请求地址为localhost，请求端口为80且符合`location`块的路径匹配参数，则nginx将按其中配置的`proxy_pass`发起反向代理。

上诉配置内容配置完成后均需要使用`nginx -s reload`命令重新加载配置文件方可生效。