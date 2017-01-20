# Server names

> [Server names](http://nginx.org/en/docs/http/server_names.html)

服务器名称使用 [`server_name`](http://nginx.org/en/docs/http/ngx_http_core_module.html#server_name) 指令定义，并确定哪个 [`server`](http://nginx.org/en/docs/http/ngx_http_core_module.html#server) 块用于给定的请求。另请参见 “[How nginx processes a request](http://nginx.org/en/docs/http/request_processing.html)” 。它们可以使用确切名称、通配符名称或正则表达式来定义：

```conf
server {
    listen      80;
    server_name example.org www.example.org;
    # ...
}
server {
    listen      80;
    server_name *.example.org;
    # ...
}
server {
    listen      80;
    server_name mail.*;
    # ...
}
server {
    listen      80;
    server_name ~^(?<user>.+)\.example\.net$;
    # ...
}
```

当按名称搜索虚拟服务器时，如果名称匹配多个指定的变体，例如 通配符名称和正则表达式匹配，将按照以下优先级顺序选择第一个匹配变体：

0. 确切名称
1. 以星号开头的最长通配符名称，例如 “`*.example.org`”
2. 以星号结尾的最长通配符名称，例如 “`mail.*`”
3. 首先匹配正则表达式（按照配置文件中出现的顺序）

## 通配符名称

通配符名称只能在名称的开头或结尾包含星号，而且只能在点号边上。名称 “`www.*.example.org`” 和 “`w*.example.org`” 无效。但是，可以使用正则表达式指定这些名称，例如，“`~^www\..+\.example\.org$`” 和 “`~^w.*\.example\.org$`” 。星号可以匹配多个名称部分。名称 “`*.example.org`” 不仅匹配 `www.example.org` ，还匹配 `www.sub.example.org` 。

使用 “`.example.org`” 形式的特殊通配符名称可以匹配确切的名称 “`example.org`” 和通配符名称 “`*.example.org`” 。

## 正则表达式名称

nginx 使用的正则表达式与 Perl 编程语言（ PCRE ）使用的正则表达式兼容。要使用正则表达式，服务器名称必须以波形符号开头：

```conf
server_name ~^www\d+\.example\.net$;
```

否则将被视为确切的名称，或者如果表达式包含星号作为通配符名称（最有可能作为无效名称）。不要忘记设置 “`^`” 和 “`$`” 定位点。它们不是语法上的，而是逻辑上的。还要注意，域名点应该用反斜杠转义。包含字符 “`{`” 和 “`}`” 的正则表达式引用：

```conf
server_name "~^(?<name>\w\d{1, 3}+)\.example\.net$";
```

否则 nginx 将无法启动并显示错误信息：

```
directive "server_name" is not terminated by ";" in ...
```

命名的正则表达式捕获可以稍后用作变量：

```conf
server {
    server_name ~^(www\.)?(?<domain>.+)$;
    
    location / {
        root /sites/$domain;
    }
}
```

PCRE 库支持使用以下语法的命名捕获：

```
?<name>  Perl 5.10 compatible syntax, supported since PCRE-7.0
?'name'  Perl 5.10 compatible syntax, supported since PCRE-7.0
?P<name> Python compatible syntax, supported since PCRE-4.0
```

如果 nginx 启动失败，并显示错误消息：

```
pcre_compile() failed: unrecognized character after (?< in ...
```

这意味着 PCRE 库是旧的，并且应尝试语法 “`?P<*name*>`” 。捕获还可以以数字形式使用：

```conf
server {
    server_name ~^(www\.)?(.+)$;
    
    location / {
        root /sites/$2;
    }
}
```

然而，这种使用应该限于简单的情况（如上所述），因为数字引用可以容易地被覆盖。

## 其他名称

有一些服务器名称被特殊处理。

如果需要处理没有默认 [服务器](http://nginx.org/en/docs/http/ngx_http_core_module.html#server) 块中的 “Host” 头字段的请求，则应该指定名称：

```conf
server {
    listen      80;
    server_name example.org www.example.org "";
}
```

如果在 [服务器](http://nginx.org/en/docs/http/ngx_http_core_module.html#server) 块中没有定义 [server_name](http://nginx.org/en/docs/http/ngx_http_core_module.html#server_name) ，则 nginx 使用名称作为服务器名称。

如果服务器名称定义为 “`$hostname`” (0.9.4) ，则使用机器的主机名。

如果有人使用 IP 地址而不是服务器名称发出请求，则 “Host” 请求字段将包含 IP 地址，并且可以使用 IP 地址作为服务器名称来处理请求：

```conf
server {
    listen      80;
    server_name example.org www.example.org "" 192.168.1.1;
    # ...
}
```

在 catch-all 服务器示例中，可以看到奇怪的名称 “`_`” ：

```conf
server {
    listen      80 default_server;
    server_name _;
    return      444;
}
```

这个名称没有什么特别之处，它只是无数的无效域名之一，从不与任何真实名称相交。其他无效名称，如 “`--`” 和 “`!@#`” 可以同样使用。

nginx 版本高达 0.6.25 支持特殊名称 “`*`” ，这被错误地解释为一个 catch-all 的名称。它从来没有用作 catch-all 或通配符服务器名称。相反，它提供了现在由 [`server_name_in_redirect`](http://nginx.org/en/docs/http/ngx_http_core_module.html#server_name_in_redirect) 指令提供的功能。特殊名称 “`*`” 现已废弃，应使用 [`server_name_in_redirect`](http://nginx.org/en/docs/http/ngx_http_core_module.html#server_name_in_redirect) 指令。请注意，没有办法使用 [`server_name`](http://nginx.org/en/docs/http/ngx_http_core_module.html#server_name) 指令指定 catch-all 名称或默认服务器。这是 [`listen`](http://nginx.org/en/docs/http/ngx_http_core_module.html#listen) 指令的属性，而不是 [`server_name`](http://nginx.org/en/docs/http/ngx_http_core_module.html#server_name) 指令的属性。另请参见 “[nginx 如何处理请求](http://nginx.org/en/docs/http/request_processing.html)”。可以定义监听端口 *:80 和 *:8080 的服务器，并指定一个将是端口 *:8080 的默认服务器，而另一个将是端口 *:80 的默认服务器：

```conf
server {
    listen      80;
    listen      8080 default_server;
    server_name example.net;
    # ...
}

server {
    listen      80 default_server;
    listen      8080;
    server_name example.org;
    # ...
}
```

## 优化

确切名称、以星号开头的通配符名称以及星号结尾的通配符名称存储在绑定到监听端口的三个哈希表中。哈希表的大小在配置阶段进行优化，以便找到具有最少 CPU 缓存未命中的名称。设置哈希表的详细信息在单独的 [文档](http://nginx.org/en/docs/hash.html) 中提供。

首先搜索确切名称的哈希表。如果未找到名称，将搜索以星号开头的通配符名称的哈希表。如果在那里找不到名称，则搜索以星号结尾的通配符名称的哈希表。

搜索通配符名称哈希表比搜索确切名称的哈希表慢，因为名称由域部分搜索。请注意，特殊通配符格式 “`.example.org`” 存储在通配符名称哈希表中，而不是确切名称哈希表中。

正则表达式是顺序测试的，因此是最慢的方法，并且是不可扩展的。

出于这些原因，最好在可能的情况下使用确切的名称。例如，如果服务器最常请求的名称是 “`example.org`” 和 “`www.example.org`” ，则显式定义它们更有效：

```conf
server {
    listen      80;
    server_name example.org www.example.org *.example.org;
    # ...
}
```

和使用简化格式相比：

```conf
server {
    listen      80;
    server_name .example.org;
    # ...
}
```

如果定义了大量的服务器名称，或定义了异常长的服务器名称，则可能需要调整 http 级别的 [server_names_hash_max_size](http://nginx.org/en/docs/http/ngx_http_core_module.html#server_names_hash_max_size) 和 [server_names_hash_bucket_size](http://nginx.org/en/docs/http/ngx_http_core_module.html#server_names_hash_bucket_size) 指令。 [server_names_hash_bucket_size](http://nginx.org/en/docs/http/ngx_http_core_module.html#server_names_hash_bucket_size) 指令的默认值可以等于 32 ，或 64 ，另一个值，具体取决于 CPU 高速缓存行大小。如果默认值为 32 ，服务器名称定义为 “`too.long.server.name.example.org`” ，则 nginx 将无法启动并显示错误信息。

```
could not build the server_names_hash,
you should increase server_names_hash_bucket_size: 32
```

在这种情况下，指令值应该增加到 2 的下一个幂。

```conf
http {
    server_names_hash_bucket_size 64;
    # ...
}
```

如果定义了大量的服务器名称，则会出现另一条错误消息：

```
could not build the server_names_hash,
you should increase either server_names_hash_max_size: 512
or server_names_hash_bucket_size: 32
```

在这种情况下，首先尝试将 [server_names_hash_max_size](http://nginx.org/en/docs/http/ngx_http_core_module.html#server_names_hash_max_size) 设置为接近服务器名称数的数字。只有当这没有用，或者如果 nginx 启动时间非常长，尝试增加 [server_names_hash_bucket_size](http://nginx.org/en/docs/http/ngx_http_core_module.html#server_names_hash_bucket_size) 。

如果服务器是监听端口的唯一服务器，那么 nginx 将不会测试服务器名称（并且不会为监听端口构建哈希表）。但是，有一个例外。如果服务器名称是带有 [捕获](captures) 的正则表达式，那么 nginx 必须执行表达式以获取捕获。

## 兼容性

* The special server name “`$hostname`” has been supported since 0.9.4.
* A default server name value is an empty name “” since 0.8.48.
* Named regular expression server name captures have been supported since 0.8.25.
* Regular expression server name captures have been supported since 0.7.40.
* An empty server name “” has been supported since 0.7.12.
* A wildcard server name or regular expression has been supported for use as the first server name since 0.6.25.
* Regular expression server names have been supported since 0.6.7.
* Wildcard form `example.*` has been supported since 0.6.0.
* The special form `.example.org` has been supported since 0.3.18.
* Wildcard form `*.example.org` has been supported since 0.1.13.