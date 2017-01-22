# Using nginx as HTTP load balancer

> [Using nginx as HTTP load balancer](http://nginx.org/en/docs/http/load_balancing.html)

## 引言

跨多个应用程序实例的负载均衡是用于优化资源利用率、最大化吞吐量、减少延迟并确保容错配置的常用技术。

可以使用 nginx 作为一个非常有效的 HTTP 负载均衡器分配流量到几个应用程序服务器，并提高使用 nginx 的 web 应用程序的性能、可扩展性和可靠性。

## 负载均衡方法

在 nginx 中支持以下负载均衡机制（或方法）：

* 轮询 — 以轮询方式分发应用服务器的请求；
* 最小连接 — 下一个请求分配给具有最少活动连接数的服务器；
* ip-hash — 哈希函数用于确定下一个请求选择什么服务器（基于客户端的 IP 地址）。

## 默认负载均衡配置

使用 nginx 进行负载均衡的最简单配置可能如下所示：

```conf
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

在上面的示例中，在 srv1 - srv3 上运行的同一应用程序有 3 个实例。当负载均衡方法没有特别配置时，它默认为 轮询。所有请求都被 [代理](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass) 到服务器组 myapp1 ，并且 nginx 应用 HTTP 负载均衡来分发请求。

nginx 中的反向代理实现包括 HTTP 、 HTTPS 、 FastCGI 、 uwsgi 、 SCGI 和 memcached 的负载均衡。

要为 HTTPS 而不是 HTTP 配置负载均衡，只需使用 “https” 作为协议。

当为 FastCGI 、 uwsgi 、 SCGI 或 memcached 设置负载均衡时，分别使用 [fastcgi_pass](http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_pass) 、 [uwsgi_pass](http://nginx.org/en/docs/http/ngx_http_uwsgi_module.html#uwsgi_pass) 、 [scgi_pass](http://nginx.org/en/docs/http/ngx_http_scgi_module.html#scgi_pass) 和 [memcached_pass](http://nginx.org/en/docs/http/ngx_http_memcached_module.html#memcached_pass) 指令。

## 最小连接负载均衡

另一个负载均衡规则是最小连接。最小连接允许在某些请求需要更长时间完成的情况下更公平地控制应用程序实例上的负载。

使用最小连接的负载均衡， nginx 将尽量不使用过多的请求来过载繁忙的应用服务器，而是将新的请求分发给不太忙的服务器。

当将 [least_conn](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#least_conn) 指令用作服务器组配置的一部分时，将激活 nginx 中的最小连接负载均衡：

```conf
upstream myapp1 {
    least_conn;
    server srv1.example.com;
    server srv2.example.com;
    server srv3.example.com;
}
```

## 会话持久性

请注意，使用轮询或最小连接的负载均衡，每个后续客户端的请求都可能分发到不同的服务器。不能保证同一客户端将始终定向到同一服务器。

如果需要将客户端绑定到特定的应用程序服务器 — 换句话说，使客户端的会话在总是尝试选择特定服务器方面是“粘性”或“持久”的 — 可以使用 ip-hash 负载均衡机制。

使用 ip-hash ，客户端的 IP 地址用作哈希键，以确定应为客户端请求选择服务器组中的哪个服务器。此方法确保来自同一客户端的请求将始终定向到同一服务器，除非此服务器不可用。

要配置 ip-hash 负载均衡，只需要将 [ip_hash](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#ip_hash) 指令添加到服务器配置（上游）组配置：

```conf
upstream myapp1 {
    ip_hash;
    server srv1.example.org;
    server srv2.example.org;
    server srv3.example.org;
}
```

## 权重负载均衡

还可以通过使用服务器权重进一步影响 nginx 负载均衡算法。

在上面的示例中，未配置服务器权重，这意味着所有指定的服务器被视为对特定负载均衡方法同等资格。

特别是对于轮询，它还意味着跨越服务器的请求的或多或少的平等分配 — 假如有足够的请求，并且当请求以统一的方式被处理并且足够快地完成。

当为服务器指定 [weight](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) 参数时，权重将作为负载均衡决策的一部分。

```conf
upstream myapp1 {
    server srv1.example.com weight=3;
    server srv2.example.com;
    server srv3.example.com;
}
```

使用此配置，每五个请求将分布在应用程序实例中，如下所示：三个请求将定向到 srv1 ，一个请求将进入 srv2 ，另一个请求将进入 srv3 。

类似的，可以在最近的 nginx 版本中使用具有最小连接和 ip-hash 负载均衡的权重。

## 健康检查

nginx 中的反向代理实现包括带内（或被动）服务器运行状况检查。如果特定服务器的响应失败并出现错误， nginx 会将此服务器标记为失败，并尝试避免一段时间内为随后的入站请求选择此服务器。

[max_fails](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) 指令设置在 [fail_timeout](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) 期间与服务器通信的连续不成功尝试的次数。默认情况下， [max_fails](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) 设置为 1 。当它设置为 0 时，将禁用此服务器的运行状况检查。 [fail_timeout](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) 参数还定义服务器将被标记为失败的时间。在服务器故障后的 [fail_timeout](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) 间隔后， nginx 将开始以实时客户端的请求正常地探测服务器。如果探测成功，则服务器将被标记为活动。

## 深入阅读

此外，有更多的指令和参数控制 nginx 中的服务器负载均衡，例如， [proxy_next_upstream](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_next_upstream) 、 [backup](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#server), [down](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) 和 [keepalive](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#keepalive) 。有关更多信息，请查看我们的参考文档。

最后同样重要的是，应用程序负载均衡、应用程序状况检查、活动监视和服务器组的即时重新配置都是我们付费的 NGINX Plus 订阅的一部分。

以下文章更详细地介绍了与 NGINX Plus 的负载均衡：

* [Load Balancing with NGINX and NGINX Plus](https://www.nginx.com/blog/load-balancing-with-nginx-plus/)
* [Load Balancing with NGINX and NGINX Plus part 2](https://www.nginx.com/blog/load-balancing-with-nginx-plus-part2/)

