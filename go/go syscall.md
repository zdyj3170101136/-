#### 系统调用，创建M

```
cloneFlags = _CLONE_VM | /* share memory */
   _CLONE_FS | /* share cwd, etc */
   _CLONE_FILES | /* share fd table */
   _CLONE_SIGHAND | /* share sig handler table */
   _CLONE_SYSVSEM | /* share SysV semaphore undo lists (see issue #20763) */
   _CLONE_THREAD /* revisit - okay for now */
```

https://blog.csdn.net/anonymalias/article/details/9238705

[https://root1iu.github.io/2019/02/02/athread%E7%9A%84%E5%88%9B%E5%BB%BA%E5%92%8C%E9%80%80%E5%87%BA%EF%BC%88%E4%B8%80%EF%BC%89/](https://root1iu.github.io/2019/02/02/athread的创建和退出（一）/)

##### CLONE_FILES

设置了CLONE_FILES后，进程和clone出来的子进程共享相同的文件描述符表。父进程或者子进程创建的文件描述符在对方进程中都是合法的。如果共享文件描述符表的进程调用execve，它的文件描述符表是重复的(非共享的)

如果CLONE_FILES不被设置，子进程在**clone(2)**执行时，继承调用进程中所有打开的的文件描述符。(子进程中复制过来的文件描述符和调用进程对应文件描述符一样)。接下来对文件描述符的操作不会影响另一个进程。

##### CLONE_FS

如果CLONE_FS被设置，调用者和子进程共享相同的文件系统信息。包括文件系统的根目录，当前工作目录等。对这些信息的修改会影响另外的进程。

和CLONE_FILES一样，如果CLONE_FS没有被设置，子进程只有有文件系统信息的副本，而不会影响到另外一个进程。

##### CLONE_SIGHAND

如果CLONE_SIGHAND被设置，调用进程和子进程共享信号处理器表。如果某个进程修改了信号的行为(通过**sigaction(2)**)，会影响到另外的一个进程。但调用进程和子进程仍然拥有独立的信号掩码和信号处理集。所以他们可以使用**sigprocmask(2)**来阻塞和释放信号而不影响另外一个。

如果CLONE_SIGHAND没有被设置，子进程只会继承调用进程的信号处理函数副本，两个进程修改信号的行为不会影响另一个进程。

从Linux2.6.0开始，如果CLONE_SIGHAND被设置了，那么CLONE_VM也需要被设置。

##### CLONE_VM

如果CLONE_VM被设置了，那么调用进程和子进程共享内存空间。特别的，两个进程的内存写操作对另一个进程是透明的。更进一步，任何内存映射和释放映射都会影响另一个进程。

如果CLONE_VM没有被设置，那么子进程和调用进程运行在不同的内存空间中，即不共享内存空间。对进程中的内存操作不会影响另外一个进程。

##### CLONE_THREAD

CLONE_THREAD如果被设置，那么这个clone出来的子进程会和调用clone的进程进一个线程组，这里的”线程”指的是线程组里的进程

线程组是Linux 2.4版本加入的，用来支持POSIX thread的一个概念，即一个线程组共享一个PID。在内部，这个共享PID被叫做线程组ID，即TGID。从Linux 2.4开始，**getpid(2)**会返回调用者的TGID。

对于CLONE_THREAD，它被赋予了新的TID，但是TGID(因此报告的进程ID)与父代保持相同，因此它们实际上具有相同的PID。但是，他们可以通过从gettid()获取TID来区分自己。

如果**clone(2)**没有指定CLONE_THREAD，那么新线程将会自己一个作为一个新的线程组，其中，TGID = TID。这个线程是新线程的头儿。

带CLONE_THREAD新建的线程和**clone(2)**的调用者有着一样的父进程。表现为**getppid(2)**返回相同的值。当一个CLONE_THREAD线程终止时，创建它的线程不会发送SIGCHLD信号以及其他终止信号；终止线程的状态也不会被**wait(2)**捕捉。(线程称为分离的)。只有当线程组中的线程都终止时，父进程会收到SIGCHLD信号。



如果线程组中的线程执行了**execve(2)**，那么除了线程头外的所有线程会终止，并且这个execve执行的新程序会在线程组头执行。

如果线程组中的一个线程使用**fork(2)**，那么线程组中的所有线程能够**wait(2)**这个子进程。

从Linux2.5.35开始，如果CLONE_THREAD被设置，那么CLONE_SIGHAND也需要被设置。需要注意的是，从Linux 2.6.0开始，CLONE_VM也需要跟着CLONE_SIGHAND一起被设置。

使用**kill(2)**来向线程组所有线程发送信号，使用**tgkill()**来向特定的线程发送信号。

信号的处理和行为是进程范围的，比如一个未处理的信号发送给一个线程，那么这个信号会影响到线程组中的所有线程。

在调度器创建M的时候，会调用`runtime.newosproc`，制服在Linux上会调用`runtime.clone`来创建一个新的线程：



Linux中，每个进程有一个pid，类型pid_t，由getpid()取得。Linux下的POSIX线程也有一个id，类型pthread_t，由pthread_self()取得，该id由线程维护，其id空间是各个进程独立的（即不同进程中的线程可能有相同的id）。你可能知道，Linux中的POSIX线程库实现的线程其实也是一个进程（LWP），只是该进程与主进程（启动线程的进程）共享一些资源而已，比如代码段，数据段等。
　　有时候我们可能需要知道线程的真实pid。比如进程P1要向另外一个进程P2中的某个线程发送信号时，既不能使用P2的pid，更不能使用线程的pthread id，而只能使用该线程的真实pid，称为tid。

https://blog.csdn.net/u012398613/article/details/52183708?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase



Linux通过进程查看线程的方法 1).`htop`按t(显示进程线程嵌套关系)和H(显示线程) ，然后F4过滤进程名。

![截屏2020-08-12 上午10.41.54](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-08-12 上午10.41.54.png)

- 相同的虚拟内存
- 相同的文件系统
- 相同的文件描述符
- 相同的信号处理器
- 相同的线程群
- _CLONE_SYSVSEM

![截屏2020-07-31 下午2.04.25](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-31 下午2.04.25.png)



#### 系统调用

```
func osinit() {
   ncpu = getproccount()
   physHugePageSize = getHugePageSize()
}
```

![截屏2020-07-31 下午2.15.33](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-31 下午2.15.33.png)



open和read读取一些文件。

比如说从这儿得到一些随机数。

```go
func getRandomData(r []byte) {
   if startupRandomData != nil {
      n := copy(r, startupRandomData)
      extendRandom(r, n)
      return
   }
   fd := open(&urandom_dev[0], 0 /* O_RDONLY */, 0)
   n := read(fd, unsafe.Pointer(&r[0]), int32(len(r)))
   closefd(fd)
   extendRandom(r, int(n))
}
```

```go
var urandom_dev = []byte("/dev/urandom\x00")
```





#### entrysyscall

当每个g陷入系统调用的时候。

```go
// systemstack runs fn on a system stack.
// If systemstack is called from the per-OS-thread (g0) stack, or
// if systemstack is called from the signal handling (gsignal) stack,
// systemstack calls fn directly and returns.
// Otherwise, systemstack is being called from the limited stack
// of an ordinary goroutine. In this case, systemstack switches
// to the per-OS-thread stack, calls fn, and switches back.
// It is common to use a func literal as the argument, in order
// to share inputs and outputs with the code around the call
// to system stack:
//
// ... set up y ...
// systemstack(func() {
//    x = bigcall(y)
// })
// ... use x ...
//
//go:noescape
func systemstack(fn func())
```



```go
osyield()
```

睡眠当前线程。

我们在讨论调度器的初始化过程和 `cgo` 实现时就已经提到过 这两个调用的作用：它们会将一个处于 `_Grunning` 状态的 G 切换到 `_Gsyscall` 状态，并在系统调用时间过长 后，主动放弃 P，供他人使用



####

 大部分这种问题都能够解决，在文章的最后，提到了一种特殊情况，就是父子进程中的端口占用情况。父进程监听一个端口后，fork出一个子进程，然后kill掉父进程，再重启父进程，这个时候提示端口占用，用netstat查看，子进程占用了父进程监听的端口。

 原理其实很简单，子进程在fork出来的时候，使用了写时复制（COW，Copy-On-Write）方式获得父进程的数据空间、 堆和栈副本，这其中也包括文件描述符。刚刚fork成功时，父子进程中相同的文件描述符指向系统文件表中的同一项（这也意味着他们共享同一文件偏移量）。这其中当然也包含父进程创建的socket。

 接着，一般我们会调用exec执行另一个程序，此时会用全新的程序替换子进程的正文，数据，堆和栈等。此时保存文件描述符的变量当然也不存在了，我们就无法关闭无用的文件描述符了。所以通常我们会fork子进程后在子进程中直接执行close关掉无用的文件描述符，然后再执行exec。

 但是在复杂系统中，有时我们fork子进程时已经不知道打开了多少个文件描述符（包括socket句柄等），这此时进行逐一清理确实有很大难度。我们期望的是能在fork子进程前打开某个文件句柄时就指定好：“这个句柄我在fork子进程后执行exec时就关闭”。其实时有这样的方法的：即所谓 的 close-on-exec。

 回到我们的应用场景中来，只要我们在创建socket的时候加上SOCK_CLOEXEC标志，就能够达到我们要求的效果，在fork子进程中执行exec的时候，会清理掉父进程创建的socket。

```javascript
#ifdef WIN32
```

https://cloud.tencent.com/developer/article/1177094

```
syscall.Open(name, flag|syscall.O_CLOEXEC, syscallMode(perm))
```



#### 打开log文件时

```go
logw, err = os.OpenFile(filepath.Join(path, "LOG"), os.O_WRONLY|os.O_CREATE, 0644)
```

创建目录时

```
os.MkdirAll(path, 0755)
```

open for sync

```go
func writeFileSynced(filename string, data []byte, perm os.FileMode) error {
   f, err := os.OpenFile(filename, os.O_WRONLY|os.O_CREATE|os.O_TRUNC, perm)
```

```go
// Flags to OpenFile wrapping those of the underlying system. Not all
// flags may be implemented on a given system.
const (
   // Exactly one of O_RDONLY, O_WRONLY, or O_RDWR must be specified.
   O_RDONLY int = syscall.O_RDONLY // open the file read-only.
   O_WRONLY int = syscall.O_WRONLY // open the file write-only.
   O_RDWR   int = syscall.O_RDWR   // open the file read-write.
   // The remaining values may be or'ed in to control behavior.
   O_APPEND int = syscall.O_APPEND // append data to the file when writing.
   O_CREATE int = syscall.O_CREAT  // create a new file if none exists.
   O_EXCL   int = syscall.O_EXCL   // used with O_CREATE, file must not exist.
   O_SYNC   int = syscall.O_SYNC   // open for synchronous I/O.
   O_TRUNC  int = syscall.O_TRUNC  // truncate regular writable file when opened.
)
```



```go
// Open opens the named file for reading. If successful, methods on
// the returned file can be used for reading; the associated file
// descriptor has mode O_RDONLY.
// If there is an error, it will be of type *PathError.
func Open(name string) (*File, error) {
   return OpenFile(name, O_RDONLY, 0)
}

// Create creates or truncates the named file. If the file already exists,
// it is truncated. If the file does not exist, it is created with mode 0666
// (before umask). If successful, methods on the returned File can
// be used for I/O; the associated file descriptor has mode O_RDWR.
// If there is an error, it will be of type *PathError.
func Create(name string) (*File, error) {
   return OpenFile(name, O_RDWR|O_CREATE|O_TRUNC, 0666)
}
```



- 通过open获得fd
- init pfd（epoll）
- setnonblock
- 注册finalizer

#### 设置文件non blocking

当第二个参数***cmd=F_GETFL***时，它的作用是取得文件描述符***filedes\***的文件状态标志。

当第二个参数***cmd=F_SETFL***时，它的作用是设置文件描述符***filedes\***的文件状态标志，这时第三个参数为新的状态标志。

```go
if err := f.pfd.Init("file", pollable); err != nil {
   // An error here indicates a failure to register
   // with the netpoll system. That can happen for
   // a file descriptor that is not supported by
   // epoll/kqueue; for example, disk files on
   // GNU/Linux systems. We assume that any real error
   // will show up in later I/O.
} else if pollable {
   // We successfully registered with netpoll, so put
   // the file into nonblocking mode.
   if err := syscall.SetNonblock(fdi, true); err == nil {
      f.nonblock = true
   }
}

runtime.SetFinalizer(f.file, (*file).close)
```

```go
func SetNonblock(fd int, nonblocking bool) (err error) {
   flag, err := fcntl(fd, F_GETFL, 0)
   if err != nil {
      return err
   }
   if nonblocking {
      flag |= O_NONBLOCK
   } else {
      flag &^= O_NONBLOCK
   }
   _, err = fcntl(fd, F_SETFL, flag)
   return err
}
```



#### epollctl

**EPOLLIN    连接到达；有数据来临；
The associated file is available for read(2) operations.
EPOLLOUT   有数据要写
The associated file is available for write(2) operations.
EPOLLRDHUP  这个好像有些系统检测不到，可以使用EPOLLIN，read返回0，删除掉事件，关闭close(fd);
如果有EPOLLRDHUP，检测它就可以直到是对方关闭；否则就用上面方法。
Stream socket peer closed connection, or shut down writing half
of connection. (This flag is especially useful for writing sim-
ple code to detect peer shutdown when using Edge Triggered moni-
toring.)**

在使用 epoll 时，对端正常断开连接（调用 close()），在服务器端会触发一个 epoll 事件。在低于 2.6.17 版本的内核中，这个 epoll 事件一般是 EPOLLIN，即 0x1，代表连接可读。

连接池检测到某个连接发生 EPOLLIN 事件且没有错误后，会认为有请求到来，将连接交给上层进行处理。这样一来，上层尝试在对端已经 close() 的连接上读取请求，只能读到 EOF，会认为发生异常，报告一个错误。

因此在使用 2.6.17 之前版本内核的系统中，我们无法依赖封装 epoll 的底层连接库来实现对对端关闭连接事件的检测，只能通过上层读取数据时进行区分处理。

https://blog.csdn.net/a13393665983/article/details/102183536?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.compare&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.compare

ET高速模式以及rdhup。

```go
func netpollopen(fd uintptr, pd *pollDesc) int32 {
   var ev epollevent
   ev.events = _EPOLLIN | _EPOLLOUT | _EPOLLRDHUP | _EPOLLET
   *(**pollDesc)(unsafe.Pointer(&ev.data)) = pd
   return -epollctl(epfd, _EPOLL_CTL_ADD, int32(fd), &ev)
}
```



#### write

- 通过一个读写锁加上写锁。读写并发执行

- prepareWrite，检查epoll是否能够进行写

- 在大多数操作系统上，无法一次性写入2gb的文件。

- 使用系统调用写文件，如果返回的事EAGAIN错误，说明内核缓冲区还没有准备好。

- 使用epoll waitWrite(休眠当前协程)

- check错误是不是pipe，如果是的话就触发sigpipe信号

- ```
  // epipecheck raises SIGPIPE if we get an EPIPE error on standard
  // output or standard error. See the SIGPIPE docs in os/signal, and
  // issue 11845.
  func epipecheck(file *File, e error) {
     if e == syscall.EPIPE && file.stdoutOrErr {
        sigpipe()
     }
  }
  ```

```go
// Write implements io.Writer.
func (fd *FD) Write(p []byte) (int, error) {
   if err := fd.writeLock(); err != nil {
      return 0, err
   }
   defer fd.writeUnlock()
   if err := fd.pd.prepareWrite(fd.isFile); err != nil {
      return 0, err
   }
   var nn int
   for {
      max := len(p)
      if fd.IsStream && max-nn > maxRW {
         max = nn + maxRW
      }
      n, err := syscall.Write(fd.Sysfd, p[nn:max])
      if n > 0 {
         nn += n
      }
      if nn == len(p) {
         return nn, err
      }
      if err == syscall.EAGAIN && fd.pd.pollable() {
         if err = fd.pd.waitWrite(fd.isFile); err == nil {
            continue
         }
      }
      if err != nil {
         return nn, err
      }
      if n == 0 {
         return nn, io.ErrUnexpectedEOF
      }
   }
}
```

```go
// Write implements io.Writer.
func (fd *FD) Write(p []byte) (int, error) {
   if err := fd.writeLock(); err != nil {
      return 0, err
   }
   defer fd.writeUnlock()
   if err := fd.pd.prepareWrite(fd.isFile); err != nil {
      return 0, err
   }
   var nn int
   for {
      max := len(p)
      if fd.IsStream && max-nn > maxRW {
         max = nn + maxRW
      }
      n, err := syscall.Write(fd.Sysfd, p[nn:max])
      if n > 0 {
         nn += n
      }
      if nn == len(p) {
         return nn, err
      }
      if err == syscall.EAGAIN && fd.pd.pollable() {
         if err = fd.pd.waitWrite(fd.isFile); err == nil {
            continue
         }
      }
      if err != nil {
         return nn, err
      }
      if n == 0 {
         return nn, io.ErrUnexpectedEOF
      }
   }
}
```



#### epollopen

```
func netpollinit() {
   epfd = epollcreate1(_EPOLL_CLOEXEC)
   if epfd >= 0 {
      return
   }
   epfd = epollcreate(1024)
   if epfd >= 0 {
      closeonexec(epfd)
      return
   }
   println("runtime: epollcreate failed with", -epfd)
   throw("runtime: netpollinit failed")
}
```

#### epollwait

循环调用epollwait，获得之后。

根据events判断

- 如果参数小于 0，无限期等待文件描述符就绪；
- 如果参数等于 0，非阻塞地轮询网络；
- 如果参数大于 0，阻塞特定时间轮询网络；



![截屏2020-08-06 下午2.16.53](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-08-06 下午2.16.53.png)

- 无限期阻塞等待就位

- 如果是EPOLLIN,EPOOLLRDHUP， EPOLLHUP这三个，表示是读
- 如果是EPOLLOUT,EPOOLLHUP这两个表示是写。
- 这里其实当epoollRDHUP出现的时候，会把读的goroutine打开。即使此时已经关闭。

```go
// polls for ready network connections
// returns list of goroutines that become runnable
func netpoll(block bool) gList {
   if epfd == -1 {
      return gList{}
   }
   waitms := int32(-1)
   if !block {
      waitms = 0
   }
   var events [128]epollevent
retry:
   n := epollwait(epfd, &events[0], int32(len(events)), waitms)
   if n < 0 
      if n != -_EINTR {
         println("runtime: epollwait on fd", epfd, "failed with", -n)
         throw("runtime: netpoll failed")
      }
      goto retry
   }
   var toRun gList
   for i := int32(0); i < n; i++ {
      ev := &events[i]
      if ev.events == 0 {
         continue
      }
      var mode int32
      if ev.events&(_EPOLLIN|_EPOLLRDHUP|_EPOLLHUP|_EPOLLERR) != 0 {
         mode += 'r'
      }
      if ev.events&(_EPOLLOUT|_EPOLLHUP|_EPOLLERR) != 0 {
         mode += 'w'
      }
      if mode != 0 {
         pd := *(**pollDesc)(unsafe.Pointer(&ev.data))
         pd.everr = false
         if ev.events == _EPOLLERR {
            pd.everr = true
         }
         netpollready(&toRun, pd, mode)
      }
   }
   if block && toRun.empty() {
      goto retry
   }
   return toRun
}
```

#### go

```go
// copyBuffer is the actual implementation of Copy and CopyBuffer.
// if buf is nil, one is allocated.
func copyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
   // If the reader has a WriteTo method, use it to do the copy.
   // Avoids an allocation and a copy.
   if wt, ok := src.(WriterTo); ok {
      return wt.WriteTo(dst)
   }
   // Similarly, if the writer has a ReadFrom method, use it to do the copy.
   if rt, ok := dst.(ReaderFrom); ok {
      return rt.ReadFrom(src)
   }
  
```

注意readfrom接口。

直接目的端口调用readfrom获取源端口数据。

而writeTo也是封装了readfrom接口。

而也只有tcp实现了这种接口

```go
func (c *TCPConn) readFrom(r io.Reader) (int64, error) {
   if n, err, handled := splice(c.fd, r); handled {
      return n, err
   }
   if n, err, handled := sendFile(c.fd, r); handled {
      return n, err
   }
   return genericReadFrom(c, r)
}
```

