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

## 旋转日志文件

为了旋转日志文件，首先需要重命名它们。之后应该发送 USR1 信号到主进程。然后主进程将重新打开所有当前打开的日志文件，并为其分配工作进程正在为所有者运行的非特权用户。成功重新打开后，主进程关闭所有打开的文件，并将消息发送到工作进程，要求它们重新打开文件。因此，旧文件几乎立即可用于后处理，例如压缩。

## 联机升级可执行文件

为了升级服务器可执行文件，应首先使用新的可执行文件替换旧文件。之后应该发送 USR2 信号到主进程。主进程首先将具有进程 ID 的文件重命名为具有 `.oldbin` 后缀的新文件，例如 `/usr/local/nginx/logs/nginx.pid.oldbin` ，然后启动一个新的可执行文件，启动新的工作进程：

```
  PID  PPID USER    %CPU   VSZ WCHAN  COMMAND
33126     1 root     0.0  1164 pause  nginx: master process /usr/local/nginx/sbin/nginx
33134 33126 nobody   0.0  1368 kqread nginx: worker process (nginx)
33135 33126 nobody   0.0  1380 kqread nginx: worker process (nginx)
33136 33126 nobody   0.0  1368 kqread nginx: worker process (nginx)
33264 33126 root     0.0  1148 pause  nginx: master process /usr/local/nginx/sbin/nginx
33265 33264 nobody   0.0  1364 kqread nginx: worker process (nginx)
33266 33264 nobody   0.0  1364 kqread nginx: worker process (nginx)
33267 33264 nobody   0.0  1364 kqread nginx: worker process (nginx)
```

之后，所有工作进程（旧的和新的）继续接受请求。如果 WINCH 信号被发送到第一主进程，它将向其工作进程发送消息，请求它们正常关闭，并且它们将开始退出：

```
  PID  PPID USER    %CPU   VSZ WCHAN  COMMAND
33126     1 root     0.0  1164 pause  nginx: master process /usr/local/nginx/sbin/nginx
33135 33126 nobody   0.0  1380 kqread nginx: worker process is shutting down (nginx)
36264 33126 root     0.0  1148 pause  nginx: master process /usr/local/nginx/sbin/nginx
36265 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
36266 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
36267 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
```

一段时间后，只有新的工作进程将处理请求：

```
  PID  PPID USER    %CPU   VSZ WCHAN  COMMAND
33126     1 root     0.0  1164 pause  nginx: master process /usr/local/nginx/sbin/nginx
36264 33126 root     0.0  1148 pause  nginx: master process /usr/local/nginx/sbin/nginx
36265 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
36266 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
36267 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
```

应当注意，旧主进程不关闭其监听套接字，并且可以被管理以在其需要时再次开始其工作进程。如果由于某种原因新的可执行文件工作不可接受，可以执行以下操作之一：

* 将 HUP 信号发送到旧的主进程。旧的工作进程将启动新的工作进程，而不重新读取配置。之后，可以通过向新的主进程发送 QUIT 信号来正常关闭所有新进程。
* 将 TERM 信号发送到新的主进程。然后它将向其工作进程发送一条消息，请求它们立即退出，并且它们将几乎立即退出。（如果新进程由于某种原因不退出，则应向其发送 KILL 信号以强制它们退出。）当新的主进程退出时，旧的主进程将自动启动新的工作进程。

如果新的主进程退出，则旧主进程从带有进程 ID 的文件名中丢弃 `.oldbin` 后缀。

如果升级成功，则应向旧主进程发送 QUIT 信号，并且只有新进程将保留：

```
  PID  PPID USER    %CPU   VSZ WCHAN  COMMAND
36264     1 root     0.0  1148 pause  nginx: master process /usr/local/nginx/sbin/nginx
36265 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
36266 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
36267 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
```

