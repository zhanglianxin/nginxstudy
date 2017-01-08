# Controlling Nginx

> [Controlling Nginx](http://nginx.org/en/docs/control.html)
>

可以使用信号控制 nginx 。默认情况下主进程的进程 ID 被写入到文件 `/usr/local/nginx/logs/nginx.pid` 。这个名称可以改变在配置期间，或者使用 [pid](http://nginx.org/en/docs/ngx_core_module.html#pid) 指令。主进程支持下列信号：

```shell
TERM, INT # 快速关闭
QUIT      # 优雅关闭
HUP       # 改变配置，和时区改变保持一致（仅支持 FreeBSD 和 Linux ），
          # 使用一个新的配置开启新的工作进程，优雅关闭旧的进程
USR1      # 重新打开日志文件
USR2      # 升级一个可执行文件
WINCH     # 优雅关闭工作进程
```

独立的工作进程也可以使用信号控制，但不是必需的。支持的信号有：

```shell
TERM, INT # 快速关闭
QUIT      # 优雅关闭
USR1      # 重新打开日志文件
WINCH     # 异常终止调试（需要启用 debug_points ）
```

## 改变配置

为了让 nginx 重新读取配置文件，需要向主进程发送一个 HUP 信号。主进程首先检查语法有效性，然后尝试应用新配置，即打开日志文件和新的监听套接字。如果失败，它会回滚改变继续使用旧配置工作。如果成功，它会开启新的工作进程，向旧工作进程发送消息请求它们优雅地关闭。旧工作进程关闭监听套接字继续服务旧客户端，当服务完成所有客户端，旧工作进程关闭。

让我们通过例子来说明这一点。想象一下， nginx 运行在 FreeBSD 4.x ，这个命令

```shell
ps axw -o pid, ppid, user, %cpu, vsz, wchan, command | egrep '(nginx|PID)'
```

产生了下列输出：

```
  PID  PPID USER    %CPU   VSZ WCHAN  COMMAND
33126     1 root     0.0  1148 pause  nginx: master process /usr/local/nginx/sbin/nginx
33127 33126 nobody   0.0  1380 kqread nginx: worker process (nginx)
33128 33126 nobody   0.0  1364 kqread nginx: worker process (nginx)
33129 33126 nobody   0.0  1364 kqread nginx: worker process (nginx)
```

如果发送 HUP 给主进程，输出变成：

```
  PID  PPID USER    %CPU   VSZ WCHAN  COMMAND
33126     1 root     0.0  1164 pause  nginx: master process /usr/local/nginx/sbin/nginx
33129 33126 nobody   0.0  1380 kqread nginx: worker process is shuttint down (nginx)
33134 33126 nobody   0.0  1368 kqread nginx: worker process (nginx)
33135 33126 nobody   0.0  1368 kqread nginx: worker process (nginx)
33136 33126 nobody   0.0  1368 kqread nginx: worker process (nginx)
```

旧工作进程之一 PID 33129 仍在继续工作。一段时间后退出：

```
  PID  PPID USER    %CPU   VSZ WCHAN  COMMAND
33126     1 root     0.0  1164 pause  nginx: master process /usr/local/nginx/sbin/nginx
33134 33126 nobody   0.0  1368 kqread nginx: worker process (nginx)
33135 33126 nobody   0.0  1368 kqread nginx: worker process (nginx)
33136 33126 nobody   0.0  1368 kqread nginx: worker process (nginx)
```

