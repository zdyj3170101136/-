#### 阻塞 I /o

分为两个部分

- 数据从设备缓冲区复制到内核缓冲区
- 数据从内核缓冲区复制到用户空间![截屏2020-07-15 下午1.12.30](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-15 下午1.12.30.png)

#### 非阻塞 I /O

![截屏2020-07-15 下午1.12.44](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-15 下午1.12.44.png)

如果内核缓冲区没有数据，会直接返回。

#### 同步io

**两者的区别就在于synchronous IO做”IO operation”的时候会将process阻塞。**按照这个定义，之前所述的blocking IO，non-blocking IO，IO multiplexing都属于synchronous IO。注意到non-blocking IO会一直轮询(polling)，这个过程是没有阻塞的，但是recvfrom阶段blocking IO,non-blocking IO和IO multiplexing都是阻塞的。 而asynchronous IO则不一样，当进程发起IO 操作之后，就直接返回再也不理睬了，直到kernel发送一个信号，告诉进程说IO完成。在这整个过程中，进程完全没有被block。


链接：https://juejin.im/post/5c725dbe51882575e37ef9ed
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

#### 异步io

#### ![截屏2020-07-15 下午1.14.34](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-15 下午1.14.34.png)

#### 信号驱动io



![截屏2020-07-15 下午1.14.00](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-15 下午1.14.00.png)

#### select（io复用）

#### ![截屏2020-07-15 下午1.13.07](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-15 下午1.13.07.png)

select传入文件描述符；

- 调用fd后会阻塞，遍历文件描述符，调用对应的poll（将进程插入对应设备的等待队列，然后告诉select哪些资源可用）
- 而如果没有描述符可以用，那么进程就会睡眠
- 当设备发现自身资源可用，并且有进程在等待队列上时，就会唤醒等待的队列。因此当任意一个进程可用时，select就会被唤醒。
- select会返回一个bitmap，用来标示文件描述符是否可用，而调用方还需要遍历bitmap获得可用的文件描述符。

缺点

- 需要把fd集合传进内存空间
- select会遍历fd
- bitmap需要拷贝回用户空间，让用户自己去查询哪个可用

https://blog.csdn.net/zhougb3/article/details/79792089



#### epoll

- epoll_create：在内核建立一个结构返回一个epfd（文件描述符）
- erpoll_ctl:将fd添加到epoll中。
- epoll_wait:阻塞等待内核返回的可读写事件。

而epfd指向一个红黑树。

每个fd等待的事件都会有一个回调函数，当事件发生时，回调函数会被触发。将对应的加入一个就绪队列。

而epoll_wait只是看就绪队列有没有而已。更快。







#### LT 和 ET

LT模式将事件返回给用户后，如果没有被处理下次还会返回；

ET模式不会。

- LT读写，一般来说有读那么就读；但是一般发送缓冲区都是不满的。因此fd回一直通知写事件；所以得保证没有数据要发送的时候，把fd从epoll列表移除，需要再加回去。
- ET模式只有socket变化才会通知；就是读缓冲区从无数据到有数据；发送缓冲区从满变成未满。

#### 优化



- epoll只会在epoll_ctl的时候传入fd，不需要在wait的时候重复拷贝

https://www.cnblogs.com/aspirant/p/9166944.html

- epoll只需要在epoll_ctl的时候把fd加入就绪队列。（在os.open打开文件的时候,提前初始化）

而不像select每次都会加入所有的就绪队列

- 从go 语言write操作来看。go使用ET

https://www.cnblogs.com/Me1onRind/articles/10671741.html

写可以直接调用系统操作写，如果返回EAGAIN错误再加入epoll。

加入epoll前会gopark睡眠当前goroutine，直到文件描述符可用

当文件描述符可用，则会被唤醒.



#### epoll唤醒被阻塞的goruntine

    阻塞在read() accept() wait()上的协程，会在netpoll函数中被唤醒，注册在epoll里的event都是边缘触发模式(ET),所以在有数据可读的时候，触发可读事件, 而仅当fd从不可写变成可写的时候才会触发可写事件(连接时,写缓冲由满变空闲,对端读取了一些数据) 。

    go的runtime线程在启动时,在有使用网路库的情况才会调用netpoll函数，在linux环境下实际上调用的就是一个for循环不断进行epoll_wait(阻塞调用), 响应事件的回调函数就是调用goready唤醒该fd对应的goruntine(置为runnnable)。每一次epoll_wait最多返回128个事件。



而go会有一个netpoll函数，不断运行epollwait（）获取可用的程序。

然后唤醒对应的goroutine

```go
// Read implements io.Reader.
func (fd *FD) Read(p []byte) (int, error) {
   if err := fd.readLock(); err != nil {
      return 0, err
   }
   defer fd.readUnlock()
   if len(p) == 0 {
      // If the caller wanted a zero byte read, return immediately
      // without trying (but after acquiring the readLock).
      // Otherwise syscall.Read returns 0, nil which looks like
      // io.EOF.
      // TODO(bradfitz): make it wait for readability? (Issue 15735)
      return 0, nil
   }
   if err := fd.pd.prepareRead(fd.isFile); err != nil {
      return 0, err
   }
   if fd.IsStream && len(p) > maxRW {
      p = p[:maxRW]
   }
   for {
      n, err := syscall.Read(fd.Sysfd, p)
      if err != nil {
         n = 0
         if err == syscall.EAGAIN && fd.pd.pollable() {
            if err = fd.pd.waitRead(fd.isFile); err == nil {
               continue
            }
         }

         // On MacOS we can see EINTR here if the user
         // pressed ^Z.  See issue #22838.
         if runtime.GOOS == "darwin" && err == syscall.EINTR {
            continue
         }
      }
      err = fd.eofError(n, err)
      return n, err
   }
}
```

socket大部分时候时可写的，没必要等待

https://baijiahao.baidu.com/s?id=1653394764952522005&wfr=spider&for=pc

#### Epoll惊群

当多个线程调用epollwait时阻塞等待，然后事件到了，所有进程都会响应。

但是只有一个进程真实处理这个事件。

使用锁确保一个fd只会被一个进程监听。



现在linux优化，只会有一个进程被唤醒。

#### io模型比较

主要区别：
信号驱动io在从内核缓冲区复制到用户缓冲区的时候会阻塞。

阻塞io整个过程都是阻塞的。

非阻塞io内核到用户也是阻塞的。（需要不断的轮询）



select仅仅是把内核缓冲区数据拷贝到用户，和non-block，信号io一样。



而设备缓冲区可以了，不代表内核缓冲区ok。





select或poll调用之后，会阻塞进程，与blocking IO阻塞不同在于，`此时的select不是等到socket数据全部到达再处理, 而是有了一部分数据就会调用用户进程来处理`。如何知道有一部分数据到达了呢？`监视的事情交给了内核，内核负责数据到达的处理。也可以理解为"非阻塞"吧`。（重点是一个进程监听多个文件描述符）



在IO multiplexing Model中，`实际中，对于每一个socket，一般都设置成为non-blocking`，但是，如上图所示，整个用户的process其实是一直被block的。`只不过process是被select这个函数block，而不是被socket IO给block`。所以**`IO多路复用是阻塞在select，epoll这样的系统调用之上，而没有阻塞在真正的I/O系统调用如recvfrom之上。`**





系统不需要创建新的额外进程或者线程，也不需要维护这些进程和线程的运行，降底了系统的维护工作量，节省了系统资源，I/O多路复用的主要应用场景如下：



![截屏2020-07-15 下午1.17.26](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-15 下午1.17.26.png)

![截屏2020-07-15 下午1.14.53](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-15 下午1.14.53.png)

https://www.jianshu.com/p/486b0965c296



。Linux 的异步 IO 最初是为数据库设计的，`因此通过异步 IO 的读写操作不会被缓存或缓冲，这就无法利用操作系统的缓存与缓冲机制`。



**`很多人把 Linux 的 O_NONBLOCK 认为是异步方式，但事实上这是前面讲的同步非阻塞方式。`**需要指出的是，虽然 Linux 上的 IO API 略显粗糙，但每种编程框架都有封装好的异步 IO 实现。操作系统少做事，把更多的自由留给用户，正是 UNIX 的设计哲学，也是 Linux 上编程框架百花齐放的一个原因。

（从内核到用户进程还需要阻塞，但是从设备到内核不需要）。



#### epoll到底是不是异步io

更为重要的是, epoll 因为采用 mmap的机制, 使得 内核socket buffer和 用户空间的 buffer共享, 从而省去了 socket data copy, 这也意味着, 当epoll 回调上层的 callback函数来处理 socket 数据时, 数据已经从内核层 "自动" 到了用户空间, 虽然和 用poll 一样, 用户层的代码还必须要调用 read/write, 但这个函数内部实现所触发的深度不同了.

用 poll 时, poll通知用户空间的Appliation时, 数据还在内核空间, 所以Appliation调用 read API 时, 内部会做 copy socket data from kenel space to user space.

而用 epoll 时, epoll 通知用户空间的Appliation时?, 数据已经在用户空间, 所以 Appliation调用 read API 时?, 只是读取用户空间的 buffer, 没有 kernal space和 user space的switch了.

于是想了一下：

明显没有IO操作的拷贝数据到内核空间了，stevens应该在99年就挂了，2.6内核的epoll才采用mmap机制，书籍偏旧了吧。

那么epoll是异步IO了吧。



