# How nginx processes a request

> [How nginx processes a request](http://nginx.org/en/docs/http/request_processing.html)
>

## 基于名称的虚拟服务器

nginx 首先决定哪个 `server` 应该处理请求。 让我们开始一个简单的配置，其中所有三个虚拟服务器侦听端口 `*:80` ：

```conf
server {
    listen      80;
    server_name example.org www.example.org;
    # ...
}
server {
    listen      80;
    server_name example.net www.example.net;
    # ...
}
server {
    listen      80;
    server_name example.com www.example.com;
    # ...
}
```

在此配置中， nginx 仅测试请求的头字段 "Host" ，以确定请求路由到哪个服务器。如果它的值不匹配任何服务器名称，或者请求根本不包含这个头字段，那么 nginx 会将请求路由到此端口的默认服务器。在上面的配置中，默认服务器是第一个——这是 nginx 的标准默认行为。也可以明确设置哪个服务器应该是默认的，在 [listen](http://nginx.org/en/docs/http/ngx_http_core_module.html#listen) 指令中使用 `default_server` 参数：

```conf
server {
    listen      80 default_server;
    server_name example.net www.example.net;
    # ...
}
```

请注意，默认服务器是监听端口的属性，而不是服务器名称的属性。后面还有介绍。

## 如何防止使用未定义的服务器名称处理请求

如果不允许没有 "Host" 头字段的请求，则可以定义丢弃请求的服务器：

```conf
server {
    listen      80;
    server_name "";
    return      444;
}
```

这里，服务器名称设置为空字符串，其将匹配没有 "Host" 头字段的请求，并且返回关闭连接的特殊 nginx 的非标准码 444 。

## 基于名称和基于 IP 的混合型虚拟服务器

让我们看一个更复杂的配置，其中一些虚拟服务器侦听不同的地址：

```conf
server {
    listen      192.168.1.1:80;
    server_name example.org www.example.org;
    # ...
}
server {
    listen      192.168.1.1:80;
    server_name example.net www.example.net;
    # ...
}
server {
    listen      192.168.1.2:80;
    server_name example.com www.example.com;
    # ...
}
```

在此配置中， nginx 首先根据 [`server`](http://nginx.org/en/docs/http/ngx_http_core_module.html#server) 块的 [`listen`](http://nginx.org/en/docs/http/ngx_http_core_module.html#listen) 指令测试请求的 IP 地址和端口。然后，它针对与 IP 地址和端口匹配的 [`server`](http://nginx.org/en/docs/http/ngx_http_core_module.html#server) 块的 [`server_name`](http://nginx.org/en/docs/http/ngx_http_core_module.html#server_name) 条目测试请求的 "Host" 头字段。如果未找到服务器名称，则请求将由默认服务器处理。例如，在 192.168.1.1:80 端口上收到的对 `www.example.com` 的请求将由 192.168.1.1:80 端口的默认服务器处理，即由第一个服务器处理，因为没有为此端口定义 `www.example.com` 。

如前所述，默认服务器是监听端口的属性，并且可以为不同端口定义不同的默认服务器：

```conf
server {
    listen 192.168.1.1:80;
    server_name example.org www.example.org;
    # ...
}
server {
    listen 192.168.1.1:80 default_server;
    server_name example.net www.example.net;
    # ...
}
server {
    listen 192.168.1.2:80 default_server;
    server_name example.com www.example.com;
    # ...
}
```

## 一个简单的 PHP 网站配置

现在让我们看看 nginx 如何选择一个 `location` 来处理一个典型的简单的 PHP 网站的请求：

```conf
server {
    listen      80;
    server_name example.org www.example.org;
    root        /data/www;
    
    location / {
        index index.html index.php;
    }
    
    location ~* \(gif|jpg|png).$ {
        expires 30d;
    }
    
    location ~ \.php$ {
        fastcgi_pass  localhost:9000;
        fastcgi_param SCRIPT_FILENAME
                      $document_root$fastcgi_script_name;
        include       fastcgi_params;
    }
}
```

nginx 首先搜索由字符串给出的最具体的前缀 location ，而不管列出的顺序如何。在上面的配置中，唯一的前缀 location 是 "`/`" ，因为它匹配任何请求，它将被用作最后的手段。然后 nginx 按照配置文件中列出的顺序检查由正则表达式给出的 location 。第一个匹配表达式停止搜索， nginx 将使用此 location 。如果没有正则表达式匹配请求，那么 nginx 使用最早找到的最特定的前缀的 location 。

请注意，所有类型的 location 只测试请求行中没有参数的 URI 部分。这是因为查询字符串中的参数可以以多种方式给出，例如：

>/index.php?user=john&page=1
>/index.php?page=1&user=john

此外，任何人都可以请求查询字段中的任何内容：

> /index.php?page=1&something+else&user=john

现在让我们看看如何在上面的配置中处理请求：

* 请求 "`/logo.gif`" 首先由前缀 location "`/`" 匹配，然后由正则表达式 "`\.(gif|jpg|png)$`" 匹配，因此，它由后者处理。使用指令 "`root /data/www`" ，请求映射到文件 "`/data/www/logo.gif`" ，并将文件发送到客户端。
* ​