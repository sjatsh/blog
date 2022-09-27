---
layout: post
title: "go平滑重启选型和项目实践"
date: "2019-09-17 21:38:00 +0800"
tag: "golang"
categories: "技术"
---

## 什么是平滑重启

当线上代码需要更新时,我们平时一般的做法需要先关闭服务然后再重启服务. 这时线上可能存在大量正在处理的请求, 这时如果我们直接关闭服务会造成请求全部 中断, 影响用户体验; 在重启重新提供服务之前, 新请求进来也会502. 这时就出现两个需要解决的问题:

*   老服务正在处理的请求必须处理完才能退出(优雅退出)
*   新进来的请求需要正常处理,服务不能中断(平滑重启)

本文主要结合linux和Golang中相关实现来介绍如何选型与实践过程.

## 优雅退出

在实现优雅重启之前首先需要解决的一个问题是如何优雅退出：  
我们知道在go 1.8.x后，golang在http里加入了shutdown方法，用来控制优雅退出。  
社区里不少http graceful动态重启，平滑重启的库，大多是基于http.shutdown做的。

### http shutdown 源码分析

先来看下http shutdown的主方法实现逻辑。用atomic来做退出标记的状态，然后关闭各种的资源，然后一直阻塞的等待无空闲连接，每500ms轮询一次。
```
var shutdownPollInterval = 500 * time.Millisecond

func (srv *Server) Shutdown(ctx context.Context) error {
    // 标记退出的状态
    atomic.StoreInt32(&srv.inShutdown, 1)
    srv.mu.Lock()
    // 关闭listen fd，新连接无法建立。
    lnerr := srv.closeListenersLocked()
    
    // 把server.go的done chan给close掉，通知等待的worekr退出
    srv.closeDoneChanLocked()

    // 执行回调方法，我们可以注册shutdown的回调方法
    for _, f := range srv.onShutdown {
        go f()
    }

    // 每500ms来检查下，是否没有空闲的连接了，或者监听上游传递的ctx上下文。
    ticker := time.NewTicker(shutdownPollInterval)
    defer ticker.Stop()
    for {
        if srv.closeIdleConns() {
            return lnerr
        }
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-ticker.C:
        }
    }
}
…

是否没有空闲的连接
func (s *Server) closeIdleConns() bool {
	s.mu.Lock()
	defer s.mu.Unlock()
	quiescent := true
	for c := range s.activeConn {
		st, unixSec := c.getState()
		if st == StateNew && unixSec < time.Now().Unix()-5 {
			st = StateIdle
		}
		if st != StateIdle || unixSec == 0 {
			quiescent = false
			continue
		}
		c.rwc.Close()
		delete(s.activeConn, c)
	}
	return quiescent
}
```
#### 关闭server.doneChan和监听的文件描述符
```
// 关闭doen chan
func (s *Server) closeDoneChanLocked() {
    ch := s.getDoneChanLocked()
    select {
    case <-ch:
        // Already closed. Don't close again.
    default:
        // Safe to close here. We're the only closer, guarded
        // by s.mu.
        close(ch)
    }
}

// 关闭监听的fd
func (s *Server) closeListenersLocked() error {
    var err error
    for ln := range s.listeners {
        if cerr := (*ln).Close(); cerr != nil && err == nil {
            err = cerr
        }
        delete(s.listeners, ln)
    }
    return err
}

// 关闭连接
func (c *conn) Close() error {
    if !c.ok() {
        return syscall.EINVAL
    }
    err := c.fd.Close()
    if err != nil {
        err = &OpError{Op: "close", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
    }
    return err
}
```

#### 这么一系列的操作后，server.go的serv主监听方法也就退出了。
```
func (srv *Server) Serve(l net.Listener) error {
    ...
    for {
        rw, e := l.Accept()
        if e != nil {
            select {
             // 退出
            case <-srv.getDoneChan():
                return ErrServerClosed
            default:
            }
            ...
            return e
        }
        tempDelay = 0
        c := srv.newConn(rw)
        c.setState(c.rwc, StateNew) // before Serve can return
        go c.serve(ctx)
    }
}
```

#### 那么如何保证用户在请求完成后，再关闭连接的？
```
func (s *Server) doKeepAlives() bool {
	return atomic.LoadInt32(&s.disableKeepAlives) == 0 && !s.shuttingDown()
}


// Serve a new connection.
func (c *conn) serve(ctx context.Context) {
	defer func() {
                ... xiaorui.cc ...
		if !c.hijacked() {
                        // 关闭连接，并且标记退出
			c.close()
			c.setState(c.rwc, StateClosed)
		}
	}()
        ...
	ctx, cancelCtx := context.WithCancel(ctx)
	c.cancelCtx = cancelCtx
	defer cancelCtx()

	c.r = &connReader{conn: c}
	c.bufr = newBufioReader(c.r)
	c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)

	for {
                // 接收请求
		w, err := c.readRequest(ctx)
		if c.r.remain != c.server.initialReadLimitSize() {
			c.setState(c.rwc, StateActive)
		}
                ...
                ...
                // 匹配路由及回调处理方法
		serverHandler{c.server}.ServeHTTP(w, w.req)
		w.cancelCtx()
		if c.hijacked() {
			return
		}
                ...
                // 判断是否在shutdown mode, 选择退出
		if !w.conn.server.doKeepAlives() {
			return
		}
    }
    ...
```

## 优雅重启

### 方法演进

#### 从linux系统的角度

*   直接使用`exec`，把代码段替换成新的程序的代码， 废弃原有的数据段和堆栈段并为新程序分配新的数据段与堆栈段，唯一留下的就是进程号。  
   > 这样就会存在的一个问题就是老进程无法优雅退出，老进程正在处理的请求无法正常处理完成后退出。  
   > 并且新进程服务的启动并不是瞬时的，新进程在`listen`之后`accept`之前，新连接可能因为`syn queue`队列满了而被拒绝(这种情况很少, 但在并发很高的情况下是有可能出现)。这里结合下图与TCP三次握手的过程来看可能会好理解很多，个人感觉有种豁然开朗的感觉.

![image.png](/img/tcp_three_hand.jpg)

- 通过`fork`后`exec`创建新进程， `exec`前在老进程中通过`fcntl(fd, F_SETFD, 0);`清除`FD_CLOEXEC`标志，之后`exec`新进程就会继承老进程 的fd并可以直接使用。  
之后新进程和老进程`listen`相同的fd同时提供服务， 在新进程正常启动服务后发送信号给老进程, 老进程优雅退出。   
之后所有请求 都到了新进程也就完成了本次优雅重启。
 结合实际线上环境存在的问题:  这时新的子进程由于父进程的退出, 系统会把它的父进程改成1号进程,由于线上环境大多数服务都是通过 `supervisor`进行管理的,这就会存在一个问题, `supervisor`会认为服务异常退出, 会重新启动一个新进程.

*   通过给文件描述符设置`SO_REUSEPORT`标志让两个进程监听同一个端口, 这里存在的问题是这里使用的是两个不同的FD监听同一个端口，老进程退出的时候。 `syn queue`队列中还未被accept的连接会被内核kill掉。

*   通过`ancilliary data`系统调用使用UNIX域套接字在进程之间传递文件描述符， 这样也可以实现优雅重启。但是这样的实现会比较复杂， `HAProxy`中 实现了该模型。

*   直接`fork`然后`exec`调用，子进程会继承所有父进程打开的文件描述符， 子进程拿到的文件描述符从3递增， 顺序与父进程打开顺序一致。子进程通过`epoll_ctl` 注册fd并注册事件处理函数(这里以epoll模型为例)， 这样子进程就能和父进程监听同一个端口的请求了(此时父子进程同时提供服务)， 当子进程正常启动并提供服务后 发送`SIGHUP`给父进程， 父进程优雅退出此时子进程提供服务， 完成优雅重启。

#### Golang中的实现

从上面看， 相对来说比较容易的实现是直接`fork`and`exec`的方式最简单， 那么接下来讨论下在Golang中的具体实现。

我们知道Golang中socket的fd默认是设置了`FD_CLOEXEC`标志的(net/sys_cloexec.go[参考源码](https://github.com/golang/go/blob/master/src/net/sys_cloexec.go))

```
// Wrapper around the socket system call that marks the returned file
// descriptor as nonblocking and close-on-exec.
func sysSocket(family, sotype, proto int) (int, error) {
	// See ../syscall/exec_unix.go for description of ForkLock.
	syscall.ForkLock.RLock()
	s, err := socketFunc(family, sotype, proto)
	if err == nil {
		syscall.CloseOnExec(s)
	}
	syscall.ForkLock.RUnlock()
	if err != nil {
		return -1, os.NewSyscallError("socket", err)
	}
	if err = syscall.SetNonblock(s, true); err != nil {
		poll.CloseFunc(s)
		return -1, os.NewSyscallError("setnonblock", err)
	}
	return s, nil
}
```

所以在`exec`后fd会被系统关闭，但是我们可以直接通过`os.Command`来实现。  
这里有些人可能有点疑惑了不是`FD_CLOEXEC`标志的设置，新起的子进程继承的fd会被关闭。   
事实是`os.Command`启动的子进程可以继承父进程的fd并且使用, 阅读源码我们可以知道`os.Command`中通过`Stdout`,`Stdin`,`Stderr`以及`ExtraFiles` 传递的描述符默认会被Golang清除`FD_CLOEXEC`标志, 通过`Start`方法追溯进去我们可以确认我们的想法。(syscall/exec_{GOOS}.go我这里是macos的源码实现[参考源码](https://github.com/golang/go/blob/master/src/syscall/exec_darwin.go))

```
// dup2(i, i) won't clear close-on-exec flag on Linux,
// probably not elsewhere either.
_, _, err1 = rawSyscall(funcPC(libc_fcntl_trampoline), uintptr(fd[i]), F_SETFD, 0)
if err1 != 0 {
	goto childerror
}
```

#### 结合supervisor时的问题

实际项目中, 线上服务一般是被supervisor启动的, 如上所说的我们如果通过父子进程, 子进程启动后退出父进程这种方式的话存在的问题就是子进程会被1号进程接管, 导致supervisor 认为服务挂掉重启服务,为了避免这种问题我们可以使用master, worker的方式。
这种方式基本思路就是: 项目启动的时候程序作为master启动并监听端口创建socket描述符但是不对外提供服务, 然后通过`os.Command`创建子进程通过`Stdin`, `Stdout`, `Stderr`,`ExtraFiles`和`Env`传递标椎输入输出错误和文件描述符以及环境变量. 通过环境变量子进程可以知道自己是子进程并通过`os.NewFile`将fd注册到`epoll`中, 通过fd创建`TCPListener`对象, 绑定`handle`处理器之后`accept`接受请求并处理， 参考伪代码:

```
f := os.NewFile(uintptr(3+i), "")
l, err := net.FileListener(f)
if err != nil {
	return fmt.Errorf("failed to inherit file descriptor: %d", i)
}

server:=&http.Server{Handler: handler}
server.Serve(l)
```

上述过程只是启动了worker进程并提供服务, 真正的优雅重启, 可以通过接口(由于线上环境发布机器可能没有权限,只能曲线救国)或者发送信号给worker进程,worker 发送信号给master, master进程收到信号后起一个新worker, 新worker启动并正常提供服务后发送一个信号给master,master发送退出信号给老worker,老worker退出.

日志收集的问题， 如果项目本身日志是直接打到文件，可能会存在fd滚动等问题(目前没有研究透彻). 目前的解决方案是项目log全部输出到stdout由supervisor来收集到日志文件， 创建worker的时候stdout, stderr是可以继承过去的，这就解决了日志的问题， 如果有更好的方式环境一起探讨。

## 参考文章

[谈谈golang网络库的入门认识](https://www.bytelang.com/article/content/N9kv2_T4GAU=)  
[深入理解Linux TCP backlog](https://www.jianshu.com/p/7fde92785056)  
[go优雅升级/重启工具调研](https://lrita.github.io/2019/06/06/golang-graceful-upgrade/#so_reuseport-%E5%A4%9A%E8%BF%9B%E7%A8%8B)  
[记一次惊心的网站TCP队列问题排查经历](https://zhuanlan.zhihu.com/p/36731397)  
[accept和accept4的区别](http://suntus.github.io/2015/11/25/accept%E5%92%8Caccept4%E7%9A%84%E5%8C%BA%E5%88%AB/)  