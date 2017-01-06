# nginx for Windows

> [nginx for Windows](http://nginx.org/en/docs/windows.html)
>

Windows 版的 nginx 使用本地 Win32 API。当前只使用 `select()` 连接处理方法，因此不应该期待高性能和可扩展性。由于这个和一些别的已知的问题存在，Windows 版的 nginx 被当作是 beta 版。当前，它提供和 UNIX 版的 nginx 绝大部分相同的功能，除了 XSLT 过滤器、图片过滤器、GeoIP 模块和嵌入式的 Perl 语言。

要安装 nginx/Windows ，[下载](http://nginx.org/en/download.html) 最近的 mainline 版本（1.11.6）的分发文件，是因为 mainline 分支包含了所有已知的修复。然后解压分发文件，进入解压后的路径，运行 `nginx` 。

运行 `tasklist` 命令行工具来查看 nginx 的进程。 (`tasklist /fi "imagename eq nginx.exe"`)

其中的一个进行是主进程，另外一个是工作进程。如果 nginx 没有启动，在错误日志文件 `logs\error.log` 里查找原因。如果日志文件没有被创建，原因大概是被报告给了 Windows Event Log 。如果显示的是错误的页面而不是预料的页面，也去日志文件里去查找原因。

nginx/Windows 使用它已经运行所在的路径作为配置中的相对路径的前缀。在上面的示例中，前缀是 `C:\nginx-1.11.6\` 。配置文件中的路径必须指定为 UNIX 风格的斜杠。


```
access_log    logs/site.log;
root          C:/web/html;
```

nginx/Windows 作为一个标准的控制台程序运行（而不是一个服务），它可以使用下面的命令来管理：

```shell
nginx -s stop    快速关闭
nginx -s quit    优雅关闭
nginx -s reload  改变配置，使用新配置开始新的工作进程，优雅关闭旧的工作进程
nginx -s reopen  重新打开日志文件
```

## 已知的问题

* 尽管可以开启几个工作进程，实际上只有其中一个在工作。

* 一个工作进程只能处理 1024 个同步连接。

* 缓存和其他需要共享内存支持的模块不能在 Windows Vista 和其后的版本中工作，是在这些 Windows 版本中启用了地址空间布局不规则分布。

* 不支持 UDP 代理功能。

## 可能的未来改进

* 作为一项服务运行

* 使用 I/O 完成端口作为连接处理方法

* 在一个独立的工作进行中使用多个工作线程
