# Command-line parameters

> [Command-line parameters](http://nginx.org/en/docs/switches.html)
>

nginx 支持以下命令行参数：

- `-?` | `-h` — 打印命令行参数帮助。

- `-c file` — 使用可选配置 *`file`* 而不是默认文件。

- `-g directives` — 设置 [全局配置指令](http://nginx.org/en/docs/ngx_core_module.html) ，例如 

  > ````
  >  nginx -g "pid /var/run/nginx.pid; worker_processes `sysctl -n hw.ncpu`;"
  > ````

- `-p `*`prefix`* — 设置 nginx 路径前缀，即 将保留服务器文件的目录（默认值是 *`/usr/local/nginx`* ）。

- `-q` — 在配置测试期间抑制非错误消息。

- `-s signal` — 向主进程发送 *信号* 。

  参数 *信号* 可以是下列之一：

  * `stop` — 快速关闭
  * `quit` — 正常关闭
  * `reload` — 重新加载配置，使用新配置启动新的工作进程，正常关闭旧工作进程。
  * `reopen` — 重新打开日志文件

- `-t` — 测试配置文件：nginx 检查配置是否正确的语法，然后尝试打开配置中引用的文件。

- `-T` — 与 `-t` 相同，但另外将配置文件转储到标准输出（1.9.2）。

- `-v` — 打印 nginx 版本。

- `-V` —打印 nginx 版本，编译器版本和配置参数。