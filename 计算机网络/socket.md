#### 可以用来回答发送http请求时发生了什么

#### socket编程原理

```go
var resolveTCPAddrTests = []resolveTCPAddrTest{
   {"tcp", "127.0.0.1:0", &TCPAddr{IP: IPv4(127, 0, 0, 1), Port: 0}, nil},
  	{"tcp", "127.0.0.1:http", &TCPAddr{IP: ParseIP("127.0.0.1"), Port: 80}, nil},
```

- 首先通过解析函数
- 把地址解析成ip地址以及端口。

```go
// DialTCP acts like Dial for TCP networks.
//
// The network must be a TCP network name; see func Dial for details.
//
// If laddr is nil, a local address is automatically chosen.
// If the IP field of raddr is nil or an unspecified IP address, the
// local system is assumed.
func DialTCP(network string, laddr, raddr *TCPAddr) (*TCPConn, error) {
   switch network {
   case "tcp", "tcp4", "tcp6":
   default:
      return nil, &OpError{Op: "dial", Net: network, Source: laddr.opAddr(), Addr: raddr.opAddr(), Err: UnknownNetworkError(network)}
   }
   if raddr == nil {
      return nil, &OpError{Op: "dial", Net: network, Source: laddr.opAddr(), Addr: nil, Err: errMissingAddress}
   }
   sd := &sysDialer{network: network, address: raddr.String()}
   c, err := sd.dialTCP(context.Background(), laddr, raddr)
   if err != nil {
      return nil, &OpError{Op: "dial", Net: network, Source: laddr.opAddr(), Addr: raddr.opAddr(), Err: err}
   }
   return c, nil
}
```

然后我们dialtcp得到一个conn。

- 使用socket打开一个连接，注册为nonblock和closeonexec
- 注册epoll

- 处理错误。由于tcp能够支持同时打开对吧。然后如果我们不设置本地的地址，那么linux 核心通常会随机挑选一个本地端口不管这个目标端口是什么；而且linux挑选地址也有可能挑选到已经使用的地址。
- 我们会对以上两种错误进行重试。

- 设置为nodelay（禁止nagle算法）

```go
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

```go
func (sd *sysDialer) doDialTCP(ctx context.Context, laddr, raddr *TCPAddr) (*TCPConn, error) {
   fd, err := internetSocket(ctx, sd.network, laddr, raddr, syscall.SOCK_STREAM, 0, "dial", sd.Dialer.Control)

   // TCP has a rarely used mechanism called a 'simultaneous connection' in
   // which Dial("tcp", addr1, addr2) run on the machine at addr1 can
   // connect to a simultaneous Dial("tcp", addr2, addr1) run on the machine
   // at addr2, without either machine executing Listen. If laddr == nil,
   // it means we want the kernel to pick an appropriate originating local
   // address. Some Linux kernels cycle blindly through a fixed range of
   // local ports, regardless of destination port. If a kernel happens to
   // pick local port 50001 as the source for a Dial("tcp", "", "localhost:50001"),
   // then the Dial will succeed, having simultaneously connected to itself.
   // This can only happen when we are letting the kernel pick a port (laddr == nil)
   // and when there is no listener for the destination address.
   // It's hard to argue this is anything other than a kernel bug. If we
   // see this happen, rather than expose the buggy effect to users, we
   // close the fd and try again. If it happens twice more, we relent and
   // use the result. See also:
   // https://golang.org/issue/2690
   // https://stackoverflow.com/questions/4949858/
   //
   // The opposite can also happen: if we ask the kernel to pick an appropriate
   // originating local address, sometimes it picks one that is already in use.
   // So if the error is EADDRNOTAVAIL, we have to try again too, just for
   // a different reason.
   //
   // The kernel socket code is no doubt enjoying watching us squirm.
   for i := 0; i < 2 && (laddr == nil || laddr.Port == 0) && (selfConnect(fd, err) || spuriousENOTAVAIL(err)); i++ {
      if err == nil {
         fd.Close()
      }
      fd, err = internetSocket(ctx, sd.network, laddr, raddr, syscall.SOCK_STREAM, 0, "dial", sd.Dialer.Control)
   }

   if err != nil {
      return nil, err
   }
   return newTCPConn(fd), nil
}
```



#### 服务器端

- 服务器端没啥区别

- 就是socket的mode变成了listen状态，用于生成famil。

- 新生成一个socket套接字。

- 和之前不同的是还会调用bind和listen函数。

- ```go
  func (fd *netFD) listenStream(laddr sockaddr, backlog int, ctrlFn func(string, string, syscall.RawConn) error) error {
     var err error
     if err = setDefaultListenerSockopts(fd.pfd.Sysfd); err != nil {
        return err
     }
     var lsa syscall.Sockaddr
     if lsa, err = laddr.sockaddr(fd.family); err != nil {
        return err
     }
     if ctrlFn != nil {
        c, err := newRawConn(fd)
        if err != nil {
           return err
        }
        if err := ctrlFn(fd.ctrlNetwork(), laddr.String(), c); err != nil {
           return err
        }
     }
     if err = syscall.Bind(fd.pfd.Sysfd, lsa); err != nil {
        return os.NewSyscallError("bind", err)
     }
     if err = listenFunc(fd.pfd.Sysfd, backlog); err != nil {
        return os.NewSyscallError("listen", err)
     }
     if err = fd.init(); err != nil {
        return err
     }
     lsa, _ = syscall.Getsockname(fd.pfd.Sysfd)
     fd.setAddr(fd.addrFunc()(lsa), nil)
     return nil
  }
  ```

```go
func (sl *sysListener) listenTCP(ctx context.Context, laddr *TCPAddr) (*TCPListener, error) {
   fd, err := internetSocket(ctx, sl.network, laddr, nil, syscall.SOCK_STREAM, 0, "listen", sl.ListenConfig.Control)
   if err != nil {
      return nil, err
   }
   return &TCPListener{fd: fd, lc: sl.ListenConfig}, nil
}
```



- 通过for循环进行accept
- accept是视同epoll进行accept
- 对接受到的每一个连接我们再包装成具有epoll的那种状态/

```go
func (ln *TCPListener) accept() (*TCPConn, error) {
   fd, err := ln.fd.accept()
   if err != nil {
      return nil, err
   }
   tc := newTCPConn(fd)
   if ln.lc.KeepAlive >= 0 {
      setKeepAlive(fd, true)
      ka := ln.lc.KeepAlive
      if ln.lc.KeepAlive == 0 {
         ka = defaultTCPKeepAlive
      }
      setKeepAlivePeriod(fd, ka)
   }
   return tc, nil
}
```



- accept主要就是通过线accept，然后如果返回EAGAIN的错误，表示暂时没有数据。
- 那么就通过epoll wait
- 如果是connectionARORTED的错误，就会重试。
- 设置keepalive为默认的15s（TCP_KEEPINTVL 更改keepalive间隔。SO_KEEPALIVE开启keepalive）
- （http://www.blogjava.net/yongboy/archive/2015/04/14/424413.html）

```go
// Accept wraps the accept network call.
func (fd *FD) Accept() (int, syscall.Sockaddr, string, error) {
   if err := fd.readLock(); err != nil {
      return -1, nil, "", err
   }
   defer fd.readUnlock()

   if err := fd.pd.prepareRead(fd.isFile); err != nil {
      return -1, nil, "", err
   }
   for {
      s, rsa, errcall, err := accept(fd.Sysfd)
      if err == nil {
         return s, rsa, "", err
      }
      switch err {
      case syscall.EAGAIN:
         if fd.pd.pollable() {
            if err = fd.pd.waitRead(fd.isFile); err == nil {
               continue
            }
         }
      case syscall.ECONNABORTED:
         // This means that a socket on the listen
         // queue was closed before we Accept()ed it;
         // it's a silly error, so try again.
         continue
      }
      return -1, nil, errcall, err
   }
}
```

```go
func (fd *netFD) accept() (netfd *netFD, err error) {
   d, rsa, errcall, err := fd.pfd.Accept()
   if err != nil {
      if errcall != "" {
         err = wrapSyscallError(errcall, err)
      }
      return nil, err
   }

   if netfd, err = newFD(d, fd.family, fd.sotype, fd.net); err != nil {
      poll.CloseFunc(d)
      return nil, err
   }
   if err = netfd.init(); err != nil {
      fd.Close()
      return nil, err
   }
   lsa, _ := syscall.Getsockname(netfd.pfd.Sysfd)
   netfd.setAddr(netfd.addrFunc()(lsa), netfd.addrFunc()(rsa))
   return netfd, nil
}
```

## ECONNABORTED

​     该错误被描述为“software caused connection abort”，即“软件引起的连接中止”。原因在于当服务和客户进程在完成用于 TCP 连接的“三次握手”后，客户 TCP 却发送了一个 RST （复位）分节，在服务进程看来，就在该连接已由 TCP 排队，等着服务进程调用 accept 的时候 RST 却到达了。POSIX 规定此时的 errno 值必须 ECONNABORTED。源自 Berkeley 的实现完全在内核中处理中止的连接，服务进程将永远不知道该中止的发生。服务器进程一般可以忽略该错误，直接再次调用accept



#### 类型

domain：即协议域，又称为协议族（family）。常用的协议族有，AF_INET、AF_INET6、AF_LOCAL（或称AF_UNIX，Unix域socket）、AF_ROUTE等等。协议族决定了socket的地址类型，在通信中必须采用对应的地址，

如

#### AF_INET决定了要用ipv4地址（32位的）与端口号（16位的）的组合

AF_UNIX决定了要用一个绝对路径名作为地址。
type：指定socket类型。常用的socket类型有，SOCK_STREAM、SOCK_DGRAM、SOCK_RAW、SOCK_PACKET、SOCK_SEQPACKET等等（socket的类型有哪些？）。
protocol：故名思意，就是指定协议。常用的协议有，IPPROTO_TCP、IPPTOTO_UDP、IPPROTO_SCTP、IPPROTO_TIPC等，它们分别对应TCP传输协议、UDP传输协议、STCP传输协议、TIPC传输协议（这个协议我将会单独开篇讨论！）。
注意：并不是上面的type和protocol可以随意组合的，如SOCK_STREAM不可以跟IPPROTO_UDP组合。当protocol为0时，会自动选择type类型对应的默认协议。



https://blog.csdn.net/weibo1230123/article/details/79975574

我们使用tcp_srtam和IPPROTO_TCP。

https://blog.csdn.net/xc_tsao/article/details/44123331

#### 系统调用



```
// THIS FILE IS GENERATED BY THE COMMAND AT THE TOP; DO NOT EDIT

func socket(domain int, typ int, proto int) (fd int, err error) {
   r0, _, e1 := RawSyscall(SYS_SOCKET, uintptr(domain), uintptr(typ), uintptr(proto))
   fd = int(r0)
   if e1 != 0 {
      err = errnoErr(e1)
   }
   return
}
```

```
// THIS FILE IS GENERATED BY THE COMMAND AT THE TOP; DO NOT EDIT

func bind(s int, addr unsafe.Pointer, addrlen _Socklen) (err error) {
   _, _, e1 := Syscall(SYS_BIND, uintptr(s), uintptr(addr), uintptr(addrlen))
   if e1 != 0 {
      err = errnoErr(e1)
   }
   return
}

// THIS FILE IS GENERATED BY THE COMMAND AT THE TOP; DO NOT EDIT

func connect(s int, addr unsafe.Pointer, addrlen _Socklen) (err error) {
   _, _, e1 := Syscall(SYS_CONNECT, uintptr(s), uintptr(addr), uintptr(addrlen))
   if e1 != 0 {
      err = errnoErr(e1)
   }
   return
}
```

```
// THIS FILE IS GENERATED BY THE COMMAND AT THE TOP; DO NOT EDIT

func accept(s int, rsa *RawSockaddrAny, addrlen *_Socklen) (fd int, err error) {
   r0, _, e1 := Syscall(SYS_ACCEPT, uintptr(s), uintptr(unsafe.Pointer(rsa)), uintptr(unsafe.Pointer(addrlen)))
   fd = int(r0)
   if e1 != 0 {
      err = errnoErr(e1)
   }
   return
}
```

```
// THIS FILE IS GENERATED BY THE COMMAND AT THE TOP; DO NOT EDIT

func Listen(s int, n int) (err error) {
   _, _, e1 := Syscall(SYS_LISTEN, uintptr(s), uintptr(n), 0)
   if e1 != 0 {
      err = errnoErr(e1)
   }
   return
}
```

#### 为什么为15s

当accept一条新链接，或者dial一条新链接。

都会设置keepalive。



https://github.com/golang/go/issues/23459



我试了一下，但遇到了意外情况。当前`net`仅支持设置保持活动间隔/延迟，但不支持在终止连接（`TCP_KEEPCNT`/ `tcp_keepalive_probes`）之前设置未确认的保持活动数量。
我想这意味着我们正在使用本地内核默认值`TCP_KEEPCNT`，无论它是什么（例如，在Linux `9`, `5`应该在Windows上，在Mac，FreeBSD和OpenBSD [1]上`8`）。

无法控制此参数意味着我们无法在保持活动机制认为连接断开之前精确设置目标总时间，因为总时间为（初始延迟+间隔*计数），并且初始延迟==间隔。

这里需要采取什么行动？

- 添加（私有）方法以在每个平台上设置TCP_KEEPCNT；覆盖套接字的内核默认值，并在所有平台上定位相同的总时间
- 假定本地内核默认值未更改；根据平台使用不同的keepalive间隔（例如，在Linux上为18s，在Windows上为30s，在mac / bsd上为20s），以便在“不变的内核默认值”情况下，所有平台的总时间相同（3m）
- 忽略问题，并使用相同的时间间隔，而不管平台如何；每个平台的总时间不同（例如，假设间隔为15秒，在Linux上，总时间为2m 30s，在Windows上为1m 30s，在mac / bsd上为2m 15s）

[1]作为一个侧面说明，这似乎为保持活动OpenBSD的支持[状态](https://golang.org/src/net/tcpsockopt_openbsd.go)为OpenBSD不支持保持连接，但是[这似乎并没有这样的情况](https://man.openbsd.org/NetBSD-7.1/tcp.4)。



如果您对此没有什么反对，我将尝试使用第三个也是最简单的选项（“使用相同的时间间隔，而不考虑平台”）



- 我认为使用15能够让最大的超时时间设置为2m30s。
- 然后我们http使用3min作为间隔。



- TCP_KEEPDILE 设置连接上如果没有数据发送的话，多久后发送keepalive探测分组，单位是秒
- TCP_KEEPINTVL 前后两次探测之间的时间间隔，单位是秒

```go
func setKeepAlivePeriod(fd *netFD, d time.Duration) error {
   // The kernel expects seconds so round to next highest second.
   d += (time.Second - time.Nanosecond)
   secs := int(d.Seconds())
   if err := fd.pfd.SetsockoptInt(syscall.IPPROTO_TCP, syscall.TCP_KEEPINTVL, secs); err != nil {
      return wrapSyscallError("setsockopt", err)
   }
   err := fd.pfd.SetsockoptInt(syscall.IPPROTO_TCP, syscall.TCP_KEEPIDLE, secs)
   runtime.KeepAlive(fd)
   return wrapSyscallError("setsockopt", err)
}
```

#### accept在哪个阶段

![img](https://upload-images.jianshu.io/upload_images/5822251-477f1361d732b133.JPEG?imageMogr2/auto-orient/strip|imageView2/2/w/495/format/webp)![截屏2020-08-11 上午10.53.06](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-08-11 上午10.53.06.png)

答案是：accept过程发生在三次握手之后，三次握手完成后，客户端和服务器就建立了tcp连接并可以进行数据交互了。这时可以调用accept函数获得此连接。