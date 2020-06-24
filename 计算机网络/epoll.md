#### 阻塞 I /o

分为两个部分

- 数据从设备缓冲区复制到内核缓冲区
- 数据从内核缓冲区复制到用户空间

在触发系统调用后，内核等待两个阶段完成后才返回。

#### 非阻塞 I /O

如果内核缓冲区没有数据，会直接返回。

#### select

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

