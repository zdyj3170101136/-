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



本质上来讲tcp原始的有很多缺点。

比如说序列号只有32位会快速回绕（timestamp解决）。

累积确认机制的问题（sack）

重传时太慢（cubic用一个三次函数解决）。

以及他的基于重传的算法本身就有问题（基于bbr解决）。



但是，如果我们要实现一种基于udp的协议，就不得不实现tcp的那些接受窗口这些机制。

所以协议本身得实现流量控制，拥塞控制等算法。



不同的事，我们可以在应用层解决这些问题。

就是现在看来，tcp是操作系统层，更新太麻烦了。这种复杂的协议不应该放在传输层。



- uuid和队头阻塞

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



1.6.5 更多的 ACK 块

一般来说，接收方收到发送方的消息后都应该发送一个 ACK 回复，表示收到了数据。但每收到一个数据就返回一个 ACK 回复太麻烦，所以一般不会立即回复，而是接收到多个数据后再回复，TCP SACK 最多提供 3 个 ACK block。但有些场景下，比如下载，只需要服务器返回数据就好，但按照 TCP 的设计，每收到 3 个数据包就要 “礼貌性” 地返回一个 ACK。而 QUIC 最多可以捎带 256 个 ACK block。在丢包率比较严重的网络下，更多的 ACK block 可以减少重传量，提升网络效率。



在早期的TCP拥塞控制中，通过收到duplicate ack（默认为3个），连续三次ack某一个包来告诉sender某个丢包了，然后进入fast retransmition state，sender重传这个包，当receiver收到这个重传包后，便会发一个ack（ack新的包）给sender，告诉sender下一个要发送的包是哪一个。可以看到，通过duplicate ack每次只能重传一个包，如果有多个丢包，在等待重传过程中，很容易timeout，造成带宽利用率下降（underutilized）。

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



QUIC是一种新的传输 方式，与TCP相比，它可以减少延迟。 从表面上看，QUIC与在UDP上实现的TCP + TLS + HTTP / 2非常相似。由于TCP是在操作系统内核和中间盒固件中实现的，因此对TCP进行重大更改几乎是不可能的。但是，由于QUIC建立在UDP之上，因此不受任何限制。

![截屏2020-07-14 下午6.23.54](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-14 下午6.23.54.png)

![截屏2020-07-15 上午12.03.13](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-07-15 上午12.03.13.png)

#### 连接

 （例如，响应头中 "alt-svc: quic=":443"; ma=2592000; v="44,43,39,35" 告诉 Chrome 服务端支持端口443上的QUIC，且支持的版本号是 44，43，39，35，max-age 为 2592000 秒）。 现在 Chrome 知道服务端支持 QUIC，于是尝试使用 QUIC 来进行下一个请求。发出请求后，Chrome 将采取 QUIC 和 TCP 竞争的方式与服务端建立连接。（建立这些连接，但是不发送请求）如果第一个请求通过 TCP 发出，TCP 赢得竞争，第二个请求将通过 TCP 发出。 在随后的某个时刻，QUIC 如果一旦连接成功，将来所有请求都将通过 QUIC 连接发送。



#### quic流量控制

https://blog.csdn.net/Dreamandpassion/article/details/84951322

接受端设定初始的接受窗口。

当接受端已经消耗了过半的数据的时候，才会重新发送调整流量控制的值。

又使得窗口移动。



整个packet的窗口之和为stream的窗口之和。



Chromium currently sets:

  CFCW:15728640 // 15 MB
  SFCW:6291456 // 6 MB

 

Google’s servers currently set:

  CFCW:1572864 // 1.5 MB
  SFCW:1048576 // 1 MB



一般设置在m级别。