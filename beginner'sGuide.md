# Beginner's Guide

> [Beginner's Guide](http://nginx.org/en/docs/beginners_guide.html)
>

这份指南给出了一份 nginx 的简单介绍，描述了一些简单的任务。假定 nginx 已经安装在了读者的机器上。如果没有安装，请参见 [安装 nginx](http://nginx.org/en/docs/install.html) 。这份指南描述了如何启动和停止 nginx 和重新加载配置，解释了配置文件的结构，描述了如何装配 nginx 来分发静态内容，如何配置 nginx 作为代理服务器，怎样和 FastCGI 应用连接。

nginx 有一个主进程和几个工作进程。主进程的主要作用是读取和鉴定配置，维持工作进程。工作进程处理实际请求。 nginx 采用基于事件模型和依赖于操作系统的机制，来实现高效地分发工作进程中的请求。工作进程的数量定义在配置文件，可能是给定的配置中固定的，也可能是基于可用 CPU 核心数而动态调整的（参见 [worker_processes](http://nginx.org/en/docs/ngx_core_module.html#worker_processes) ）。

配置文件决定了 nginx 和它的模块的工作方式。默认情况下，配置文件被命名为 `nginx.conf` 存放在路径 `/usr/local/nginx/conf` `/etc/nginx` 或 `/usr/local/etc/nginx` 中。

## 启动、停止 nginx 和重新加载配置

要启动 nginx ，运行指定的可执行文件。一旦 nginx 启动，可以通过带参数 `-s` 的执行命令控制。用法如下：

```shell
nginx -s signal
```

 *signal* 可选情况：

* `stop` — 快速关闭
* `quit` — 优雅关闭
* `reload` — 重新加载配置文件
* `reopen` — 重新打开日志文件

例如，要停止 nginx 工作进程直到它们完成服务的当前请求，可以执行下面的命令。

```shell
nginx -s quit # 应当在与启动 nginx 的用户环境下执行这条命令
```

更改配置文件不会立即生效直到向 nginx 发送重新加载配置的命令或者重启之前。要重新加载配置，执行命令：

```shell
nginx -s reload
```

一旦主进程接收到重载配置的信号，它会检查新的配置文件的语法有效性，并尝试应用其中的配置。如果执行成功，主进程会开启一个新的工作进程，并且发送消息给旧的工作进程，请求它们关闭。否则，主进程会回滚配置文件中的修改，并且继续以旧的配置文件工作。旧的工作进程接收到一条关闭指令，停止接受新的连接继续服务于当前的请求直到这些请求完成。然后，旧的工作进程退出。

在 Unix 实用工具的帮助下，可以向 nginx 进程发送信号，如 `kill` 。在这种情况下，信号被直接发给指定进程 ID 的进程。默认情况下， nginx 主进程的进程 ID 被写入路径 `/usr/local/nginx/logs` 或 `/var/run` 的 `nginx.pid` 文件中。例如，如果主进程的 ID 是 1628 ，要发送使 nginx 优雅关闭的 `QUIT` 信号，执行命令：

```shell
kill -s QUIT 1628
```

要得到所有运行中的 nginx 的进程的列表，可以使用 `ps` 工具，例如下面这种方式：

```shell
ps -ax | grep nginx
```

要了解更多向 nginx 发送信号的信息，请参见 [Controlling nginx](http://nginx.org/en/docs/control.html)。

## 配置文件的结构

nginx 由模块组成，通过配置文件中特定的指令控制。指定被分成简单指令和块指令。简单指令由被空格分开并以分号结尾的名称和参数组成。块指令和简单指令有着相同的结构，但不是以分号结尾而是一组被花括号包围的附加指令。如果块指令花括号内可以有其它指令，被叫做上下文（例如：[events](http://nginx.org/en/docs/ngx_core_module.html#events) [http](http://nginx.org/en/docs/http/ngx_http_core_module.html#http) [server](http://nginx.org/en/docs/http/ngx_http_core_module.html#server) [location](http://nginx.org/en/docs/http/ngx_http_core_module.html#location)）。

位于配置文件中所有上下文之外的指令被当作是 [主上下文](http://nginx.org/en/docs/ngx_core_module.html)。`events` 和 `http` 指令存在于 `main` 上下文， `server` 在 `http` 中， `location` 在 `server` 中。

一行中 `#` 之后的部分被当作是注释。

## 服务于静态内容

一个重要的 web 服务器任务是分发文件（例如 图像或静态 HTML 页面）。你会实现一个小例子，取决于请求，来自不同的本地路径（`/data/www`(可能包含 HTML 文件) `/data/images`(包含图像)）下的文件会被分发出去。这会需要编辑配置文件和设置一个 [`http`](http://nginx.org/en/docs/http/ngx_http_core_module.html#http) 块中带有两个 [`location`](http://nginx.org/en/docs/http/ngx_http_core_module.html#location) 块的 [`server`](http://nginx.org/en/docs/http/ngx_http_core_module.html#server) 块。

首先，创建 `/data/www` 路径，放入一个任意文本内容的 `index.html` 文件，创建 `/data/images` 路径，放入一些图片在里面。

接下来，打开配置文件。默认的配置文件已经包含几个 `server` 块的样例，多半是被注释掉的。现在，注释掉这些块，开始一个新的 `server` 块：

```conf
http {
    server {
    }
}
```

通常，这些配置文件可能包含几个 `server` 块，以它们 [监听](http://nginx.org/en/docs/http/ngx_http_core_module.html#listen) 的端口和 [服务名称](http://nginx.org/en/docs/http/server_names.html) 区分开。一旦 nginx 决定使用哪一个 `server` 处理请求，它会基于定义在 `server` 块里的 `location` 指令的参数检查请求头中指定的 URI 。

添加下列 `location` 块到 `server` 块中：

```conf
location / {
    root /data/www;
}
```

这个 `location` 块指定了相对来自于请求 URI 的 `/`  前缀。为了匹配请求， URI 会被添加到 [`root`](http://nginx.org/en/docs/http/ngx_http_core_module.html#root) 指令指定的路径（也就是 `/data/www` ）后面，来组成对于请求文件的本地文件系统的路径。如果有几个 `locatino` 块匹配， nginx 选择带有最长前缀的那一个。上面的 `location` 块给出了最短前缀，长度是 1 ，因此只有在其它所有 `location` 都匹配失败的情况下，这个块才会被使用。

接下来，添加第二个 `location` 块：

```conf
location /images/ {
    root /data;
}
```

它会匹配以 `/images` 开始的请求（ `location /` 也匹配类的请求，但是有着较短的前缀）。

`server` 块的最终配置应该类似于这样：

```bash
server {
    location / {
        root /data/www;
    }
    location /images/ {
        root /data;
    }
}
```

已经有一个可以工作的服务器配置，监听标准的 80 端口，可在本地机器上通过 `http://localhost/` 访问。作为以 `/images/` 开头的 URI 的请求的响应，服务器会发送来自 `/data/images` 路径下的文件。例如，响应 `http://localhost/images/example.png` 请求， nginx 会发送 `/data/images/example.png` 文件。如果这个文件不存在， nginx 会发送一个指示 404 错误的响应。 URI 不以 `/images/` 开头的请求会被映射到 `/data/www` 路径。例如，响应 `http://localhost/some/example.html` 请求， nginx 会发送 `/data/www/some/example.html` 文件。

要应用新的配置，启动 nginx 如果现在还没启动，或者想 nginx 主进程发送 `reload` 信号，执行：

```shell
nginx -s reload
```

如果没有得到预期的效果，可以尝试在路径 `/usr/local/nginx/logs` 或 `/var/log/nginx` 下的 `access.log` 和 `error.log` 文件中找出原因。

## 设置一个简单的代理服务器

nginx 经常使用的用途之一是把它设置为一个代理服务器，也就是接收请求，传递给代理的服务器，从被代理的服务器获取响应，返回给客户端。

我们将配置一个简单的代理服务器，使用本地路径的文件服务于图像请求，发送其它所有的请求给一个代理的服务器。在这个例子中，所有的服务器都被定义在一个单独的 nginx 实例上。

首先，定义一个代理的服务器，使用下面的内容添加一个或多个 `server` 块到 nginx 配置文件：

```conf
server {
    listen 8080;
    root /data/up1;
    
    location / {
    }
}
```

这就是一个简单的服务器，它监听 8080 端口（前提是 `listen` 指令还没有被指定，由于标准的 80 端口已经被占用了），映射所有的请求到 `/data/up1` 本地文件系统路径。创建这个路径，并在里面放入 `index.html` 文件。当为服务请求而选择的 `location` 块不包括自己的 `root` 指令时，使用此 `root` 指令。

接下来，使用先前的部分服务器配置，修改使它成为一个代理服务器配置。在第一个 `location` 块中，放入带协议的 [`proxy_pass`](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass)  指令，代理的服务器名称和端口在参数中指定（在我们的案例中，是	`http://localhost:8080` ）。

```conf
server {
    location / {
        proxy_pass http://localhost:8080;
    }
    
    location /images/ {
        root /data;
    }
}
```

我们将修改第二个 `location` 块，当前是映射带有 `/images/` 前缀的请求到 `/data/images` 路径下的文件，使它匹配带有标准扩展名的图像请求。修改的 `location` 块就像这样：

```conf
location ~ \.(gif|jpg|png)$ {
    root /data/images;
}
```

参数是一个正则表达式，匹配所有以 `.gif` `.jpg` 或 `.png` 结尾的 URI 。正则表达式应该用 `~` 预先处理。符合的请求会被映射到 `/data/images` 路径。

当 nginx 选择一个 `location` 块来服务于请求，它首先检查特定的 `location` 指令，记住最长前缀的 `location` ，然后检查正则表达式。如果有一条正则表达式匹配， nginx 会采用这个 `location` ，否则，就采用之前记住的那个。

一个 nginx 代理服务器的配置结果就像这样：

```conf
server {
    location / {
        proxy_pass http://localhost:8080/;
    }
    
    location ~ \.(gif|jpg|png)$ {
        root /data/images;
    }
}
```

这个服务器会过滤以 `.gif` `.jpg` `.png` 结尾的请求，将它们映射到 `/data/images` 路径（通过添加 URI 到 `root` 指令的参数），传递其他所有请求到上面配置的代理的服务器。

要应用新的配置，就像前面的部分描述的那样发送 `reload` 信号到 nginx 。

有很多 [其他](http://nginx.org/en/docs/http/ngx_http_proxy_module.html) 指令可能会在更进一步配置代理连接的时候用到。

## 设置 FastCGI 代理

nginx 可以被用来路由请求到 FastCGI 服务器，这些服务器运行着由各种框架和编程语言（例如 PHP ）构建的应用。

配合 FastCGI 工作的最基本的 nginx 配置包括，使用 [`fastcgi_pass`](http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_pass) 指令而不是 `proxy_pass` 指令，和使用 [`fastcgi_param`](http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_param) 指令来设置传递到 FastCGI 服务器的参数。假定 FastCGI 服务器可以通过 `localhost:9000` 访问。采用先前的代理配置部分作为基本配置，使用 `fastcgi_pass` 指令替换 `proxy_pass` 配置，把参数改成 `localhost:9000` 。在 PHP 中，`SCRIPT_FILENAME` 参数被用来判断脚本名称， `QUERY_STRING` 被用来传递请求参数。配置结果就是：

```conf
server {
    location / {
        fastcgi_pass localhost:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param QUERY_STRING    $query_string;
    }
    
    location ~ \.(gif|jpg|png)$ {
        root /data/images;
    }
}
```

这将设置一个服务器，将除静态图像的请求之外的的所有请求路由到 `localhost:9000` 通过 FastCGI 协议运行的代理服务器。