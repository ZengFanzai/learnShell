# Linux 守护进程

- [Linux 守护进程](#linux-%e5%ae%88%e6%8a%a4%e8%bf%9b%e7%a8%8b)
  - [前台任务和后台任务](#%e5%89%8d%e5%8f%b0%e4%bb%bb%e5%8a%a1%e5%92%8c%e5%90%8e%e5%8f%b0%e4%bb%bb%e5%8a%a1)
  - [SIGHUP 信号](#sighup-%e4%bf%a1%e5%8f%b7)
    - [Linux 系统的任务退出的设计](#linux-%e7%b3%bb%e7%bb%9f%e7%9a%84%e4%bb%bb%e5%8a%a1%e9%80%80%e5%87%ba%e7%9a%84%e8%ae%be%e8%ae%a1)
  - [disown 命令](#disown-%e5%91%bd%e4%bb%a4)
  - [标准 I/O](#%e6%a0%87%e5%87%86-io)
  - [nohup 命令](#nohup-%e5%91%bd%e4%bb%a4)
  - [Screen 命令与 Tmux 命令](#screen-%e5%91%bd%e4%bb%a4%e4%b8%8e-tmux-%e5%91%bd%e4%bb%a4)
  - [Systemd](#systemd)

## 前台任务和后台任务

- **守护进程**（daemon）：就是一直在后台运行的进程。

- "**前台任务**"(foreground job), 它会独占命令行窗口，只有运行完了或手动终止，才能执行其他命令
- 变成守护进程的第一步，就是把它改成"**后台任务**"(background job)

```sh
COMMAND <Script> &
```

"后台任务"有两个特点

1. 继承当前会话(session)的标准输出(stdout)和标准错误(stderr)。因此，后台任务的所有输出依然会同步地在命令行下显示
2. 不再继承当前会话(session)的标准输入(stdin)。你无法向这个任务输入指令了。如果它试图读取标准输入，就会暂停执行(halt)。

因此，“后台任务”与“与前台任务”的本质区别只有一个：是否继承标准输入。

## SIGHUP 信号

### Linux 系统的任务退出的设计

1. 用户准备退出 session
2. 系统向该 session 发出 SIGHUP 信号
3. session 将 SIGHUP 信号发给所有子进程
4. 子进程收到 SIGHUP 信号后，自动退出

所有前台任务退出时，会收到`SIGHUP`信号，后台任务是否能收到`SIGHUP`信号，由 Shell 的`huponexit`参数决定

```sh
shopt | grep huponexit
```

大多数 Linux 系统，该参数是默认关闭(off)。因此，session 退出时，不会把 SIGHUP 信号发给“后台任务”，及“后台任务”不会随着 session 一起退出。

## disown 命令

通过“后台任务”启动“守护进程”并不保险，更保险的方法是使用`disown`命令。`disown`命令可以将指定任务从“后台任务”列表(`jobs`命令返回的结果)之中移除。一个“后台任务”只要不在这个列表中，session 就不会向它发送`SIGHUP`信号

`disown`命令用法：

```sh
# 移出最近一个正在执行的后台任务
disown

# 移出所有正在执行的后台任务
disown -r

# 移出所有后台任务
disown -a

# 不移出后台任务，但是让它们不会收到SIGHUP信号
disown -h

# 根据jobId，移出指定的后台任务
disown %2
disown -h %2
```

## 标准 I/O

使用`disown`命令之后，如果退出 session 以后，如果后台进程与标准 I/O 有交互，它还是会挂掉。

例：

```js
var http = require("http");

http
  .createServer(function(req, res) {
    console.log("server starts..."); // 加入此行
    res.writeHead(200, { "Content-Type": "text/plain" });
    res.end("Hello World");
  })
  .listen(5000);
```

启动脚本，然后执行`disown`命令。

```sh
node server.js &
disown
```

接着，如果退出 session，访问 5000 端口，就会发现无法连接，因为“后台任务”的标准 I/O 继承至当前 session，`disown`命令并没有改变这一点。一旦“后台任务”读写标准 I/O，就会发现他已经不存在了，使用就报错终止执行。

## nohup 命令

比`disown`更方便的命令，就是`nohup`。

```sh
nohup node server.js &
```

`nohup`命令对`server.js`进程做了三件事。

- 阻止 SIGHUP 信号发到这个进程。
- 关闭标准输入。该进程不再能够接收任何输入，即使运行在前台。
- 重定向标准输出标准错误到文件 nohup.out。

`nohup`命令实际上将子进程与它所在的 session 分离了。
注意，`nohup`命令不会自动把进程变为“后台任务”，使用必须加上`&`符号

## Screen 命令与 Tmux 命令

另一种思路是使用 `terminal multiplexer` （终端复用器：在同一个终端里面，管理多个 session），典型的就是 `Screen` 命令和 `Tmux` 命令。

它们可以在当前 session 里面，新建另一个 session。这样的话，当前 session 一旦结束，不影响其他 session。而且，以后重新登录，还可以再连上早先新建的 session。

Screen 的用法如下。

```sh
# 新建一个 session
screen
node server.js
```

然后，按下 `ctrl + A` 和 `ctrl + D`，回到原来的 session，从那里退出登录。下次登录时，再切回去。

```sh
screen -r
```

如果新建多个后台 session，就需要为它们指定名字。

```sh
screen -S name

# 切回指定 session
screen -r name
screen -r pid_number

# 列出所有 session
screen -ls
```

如果要停掉某个 session，可以先切回它，然后按下`ctrl + c`和`ctrl + d`。

Tmux 比 Screen 功能更多、更强大，它的基本用法如下。

```sh
tmux
node server.js

# 返回原来的session
tmux detach

```

除了 tmux detach，另一种方法是按下 `Ctrl + B` 和 `d` ，也可以回到原来的 session。

```sh
# 下次登录时，返回后台正在运行服务 session
tmux attach
```

如果新建多个 session，就需要为每个 session 指定名字。

```sh
# 新建 session
tmux new -s session_name

# 切换到指定 session
tmux attach -t session_name

# 列出所有 session
tmux list-sessions

# 退出当前 session，返回前一个 session
tmux detach

# 杀死指定 session
tmux kill-session -t session-name
```

## Systemd

Linux 系统有自己的守护进程管理工具 Systemd

> 摘自1：[Linux 守护进程的启动方法](http://www.ruanyifeng.com/blog/2016/02/linux-daemon.html)
