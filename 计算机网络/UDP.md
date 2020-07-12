## UDP

UDP由于不保存连接状态，立即发送（TCP会把数据保存在缓冲区，时不时取一些出来发送），无连接状态（发送和接收缓冲各种参数），首部开销小（8字节）。



更适合实时应用。

用于DNS中



16 位源端口号 ｜ 16位目的端口号

16位（首部+数据）长度 ｜ 检验和

#### 区别

1. TCP面向连接，UDP面向非连接即发送数据前不需要建立链接
2. udp立即发送，tcp保存在缓冲区合适的时候再发送。
3. TCP提供可靠的服务（数据传输），UDP无法保证
4. udp首部开销小，只有8个字节。
5. TCP面向字节流，UDP面向报文（tcp会对报文的字节计数会拆分合并报文，udp不会）
6. TCP数据传输比UDP慢
7. 在一个TCP连接中，仅有两方进行彼此通信，因此广播和多播不能用于TCP
8. TCP使用校正和，确认和重传机制，累积确认，来保证可靠传输，使用滑动窗口机制来实现流量控制，udp校验和可选

#### quic

由于tcp的滑动窗口特性，这种有着前后顺序的包之间。

将会存在队头阻塞特性。

https://xiaozhuanlan.com/topic/2083674195

#### 避免队头阻塞

quic通过RAID5.

当发送五个数据包时在发送前五个包的异或结果。

这样任意一个包丢失都能够重新恢复数据。

通过这种形式基本避免了重发数据的情况。



丢了两个包以上还是会重传。

#### 0 - rtt

当发送sync报文的时候同时发送一个sessionid。

而sessionid是上次握手成功后，服务器返回的上下文标记。

这样在tcp阶段就可以实现0-rtt延迟。



![截屏2020-07-09 下午7.42.31](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-09 下午7.42.31.png)

#### 为什么要基于udp

- tcp由操作系统实现，改进困难

- 很多路由器只识别tcp和udp

  

#### 第一次连接

客户端发送一个hello。

服务端返回一个syncookie。

一次rtt

https://www.jianshu.com/p/bb3eeb36b479

#### 拥塞控制

在应用层实现了cubic，bbr等拥塞控制算法。

改变容易。

#### 可靠性

使用单调递增的packet number。



QUIC 同样是一个可靠的协议，它使用 Packet Number 代替了 TCP 的 sequence number，并且每个 Packet Number 都严格递增，也就是说就算 Packet N 丢失了，重传的 Packet N 的 Packet Number 已经不是 N，而是一个比 N 大的值。而 TCP 呢，重传 segment 的 sequence number 和原始的 segment 的 Sequence Number 保持不变，也正是由于这个特性，引入了 Tcp 重传的歧义问题。



对于接受到的报文如果本应该是重传请求的响应，单计算成原始请求，那么rt过大。

![截屏2020-07-10 下午3.10.38](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-10 下午3.10.38.png)

#### packet通过重传stream offset来保持数据的有序性

#### ![截屏2020-07-10 下午3.11.39](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-10 下午3.11.39.png)

#### 不能丢弃已经接受的护具

在tcp中，接受方丢弃已经接受的数据，考虑缓存溢出。

但是quic不能。

#### 更多的选择确认

tcp通过sack实现了选择确认。

但是由于首部option只有40个字节。



但是范围小，最多四个段。

quic就可以提供256个段。

#### timestamp

udp计算rtt的时候会减去acp delay。



#### 没有对头阻塞问题。

http2一个tcp多个连接。

如果一个stream的阻塞了，全会阻塞。

![截屏2020-07-10 下午3.49.11](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-10 下午3.49.11.png)但是quic不会。

#### 速重启会话

手机端的时候，如果网络改变会改变ip地址这时候tcp会话会重启。

但是udp使用uuid标志连接，不惜要重新握手。

任何一条 QUIC 连接不再以 IP 及端口四元组标识，而是以一个 64 位的随机数作为 ID 来标识，这样就算 IP 或者端口发生变化时，只要 ID 不变，这条连接依然维持着，上层业务逻辑感知不到变化，不会中断，也就不需要重连。

#### 报文头部也有认证

TCP 协议头部没有经过任何加密和认证，所以在传输过程中很容易被中间网络设备篡改，注入和窃听。比如修改序列号、滑动窗口。这些行为有可能是出于性能优化，也有可能是主动攻击。

但是 QUIC 的 packet 可以说是武装到了牙齿。除了个别报文比如 PUBLIC_RESET 和 CHLO，所有报文头部都是经过认证的，报文 Body 都是经过加密的。

https://zhuanlan.zhihu.com/p/32553477

#### 流量控制

quic的流量控制针对的是stream。

类似tcp流量控制，接受端告诉发送端自己可以接受的字节数。

但是不会因为，未ack的数据而对头阻塞。



![截屏2020-07-10 下午3.50.26](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-10 下午3.50.26.png