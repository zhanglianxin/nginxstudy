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

nginx 由模块组成，通过配置文件中特定的指令控制。指定被分成简单指令和块指令。简单指令由被空格分开并以分号结尾的名字和参数组成。块指令和简单指令有着相同的结构，但不是以分号结尾而是一组被花括号包围的附加指令。如果块指令花括号内可以有其它指令，被叫做上下文（例如：[events](http://nginx.org/en/docs/ngx_core_module.html#events) [http](http://nginx.org/en/docs/http/ngx_http_core_module.html#http) [server](http://nginx.org/en/docs/http/ngx_http_core_module.html#server) [location](http://nginx.org/en/docs/http/ngx_http_core_module.html#location)）。

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

通常，这些配置文件可能包含几个 `server` 块，以它们 [监听](http://nginx.org/en/docs/http/ngx_http_core_module.html#listen) 的端口和 [服务名字](http://nginx.org/en/docs/http/server_names.html) 区分开。