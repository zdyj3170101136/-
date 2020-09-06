#### channel

```go
func (c *Channel) StartInFlightTimeout(msg *Message, clientID int64, timeout time.Duration) error {
   now := time.Now()
   msg.clientID = clientID
   msg.deliveryTS = now
   msg.pri = now.Add(timeout).UnixNano()
   err := c.pushInFlightMessage(msg)
   if err != nil {
      return err
   }
   c.addToInFlightPQ(msg)
   return nil
}
```

channel维护inflightMap和inflightpeerQueue优先队列。

使用当前时间作为优先权。

则越早加入的优先权越大。



并且使用diskqueue对消息进行持久化。

维护和客户端之间的连接。

#### queuescanworker

开启一定数量的协程。

对传来的channel调用processinflight和processdefered。

将是否成功的结果返回。

```go
// queueScanWorker receives work (in the form of a channel) from queueScanLoop
// and processes the deferred and in-flight queues
func (n *NSQD) queueScanWorker(workCh chan *Channel, responseCh chan bool, closeCh chan int) {
   for {
      select {
      case c := <-workCh:
         now := time.Now().UnixNano()
         dirty := false
         if c.processInFlightQueue(now) {
            dirty = true
         }
         if c.processDeferredQueue(now) {
            dirty = true
         }
         responseCh <- dirty
      case <-closeCh:
         return
      }
   }
}
```



- processmessage会用当前时间去peerqueue中取，如果其中的message在之前就从queue中取出。
- 然后加入memorymsgchan。

#### 作用

清理内存中因为超时没有发送的信息，以及defer时间到了之后的信息。

放入memorymsgchan中，保证消息的可达性。

https://www.jianshu.com/p/2f1c12a368a4

#### channel putmessage and putdefered

- 普通的信息加入memorymsgchan，然后通过backend存储

```go
func (c *Channel) put(m *Message) error {
   select {
   case c.memoryMsgChan <- m:
   default:
      b := bufferPoolGet()
      err := writeMessageToBackend(b, m, c.backend)
      bufferPoolPut(b)
      c.ctx.nsqd.SetHealth(err)
      if err != nil {
         c.ctx.nsqd.logf(LOG_ERROR, "CHANNEL(%s): failed to write message to backend - %s",
            c.name, err)
         return err
      }
   }
   return nil
}
```

- defermessage直接加进priorityqueue

- ```go
  func (c *Channel) StartDeferredTimeout(msg *Message, timeout time.Duration) error {
     absTs := time.Now().Add(timeout).UnixNano()
     item := &pqueue.Item{Value: msg, Priority: absTs}
     err := c.pushDeferredMessage(item)
     if err != nil {
        return err
     }
     c.addToDeferredPQ(item)
     return nil
  }
  ```

- putmessagedefer被dpub，putmessage被pub触发调用直接加入momorymsgchan
- 而inflightmessage被protocol_v2的messagepump触发。
- 然后最终inflightmessage和defermessage里头的数据都会被加入memorymsgChan

```go
func (c *Channel) processDeferredQueue(t int64) bool {
   c.exitMutex.RLock()
   defer c.exitMutex.RUnlock()

   if c.Exiting() {
      return false
   }

   dirty := false
   for {
      c.deferredMutex.Lock()
      item, _ := c.deferredPQ.PeekAndShift(t)
      c.deferredMutex.Unlock()

      if item == nil {
         goto exit
      }
      dirty = true

      msg := item.Value.(*Message)
      _, err := c.popDeferredMessage(msg.ID)
      if err != nil {
         goto exit
      }
      c.put(msg)
   }

exit:
   return dirty
}
```

#### resizepool

- 动态调整queuescanloop这个协程的并发数量
- 最多四个
- 传进来的num为channel的数量，默认每四个channel分配一个queuescanloop协程去处理。
- 很有意思的事都有最小值以及最大值的设计

```go
// resizePool adjusts the size of the pool of queueScanWorker goroutines
//
//     1 <= pool <= min(num * 0.25, QueueScanWorkerPoolMax)
//
func (n *NSQD) resizePool(num int, workCh chan *Channel, responseCh chan bool, closeCh chan int) {
   idealPoolSize := int(float64(num) * 0.25)
   if idealPoolSize < 1 {
      idealPoolSize = 1
   } else if idealPoolSize > n.getOpts().QueueScanWorkerPoolMax {
      idealPoolSize = n.getOpts().QueueScanWorkerPoolMax
   }
   for {
      if idealPoolSize == n.poolSize {
         break
      } else if idealPoolSize < n.poolSize {
         // contract
         closeCh <- 1
         n.poolSize--
      } else {
         // expand
         n.waitGroup.Wrap(func() {
            n.queueScanWorker(workCh, responseCh, closeCh)
         })
         n.poolSize++
      }
   }
}
```

#### queuescanloop

- 由于channel的数量会动态改变，所有我们会每隔5s获取当前channel的数量，然后调用resizePool控制并发queuescanworker的数量
- 然后每100ms我们醒来，选择默认20个随机的channel
- 触发这些channel的queuescanworker的工作
- 如果超过25%的被选中的channel都有dirty数据需要处理，那么不断循环处理



- 控制并发worker的数量
- 控制一定数目的channel更新
- channel具有高水位



- 注意，基本上任何都有一个默认值，并且当数目小于或者大于默认值的时候，采用客观的数目

```go
// queueScanLoop runs in a single goroutine to process in-flight and deferred
// priority queues. It manages a pool of queueScanWorker (configurable max of
// QueueScanWorkerPoolMax (default: 4)) that process channels concurrently.
//
// It copies Redis's probabilistic expiration algorithm: it wakes up every
// QueueScanInterval (default: 100ms) to select a random QueueScanSelectionCount
// (default: 20) channels from a locally cached list (refreshed every
// QueueScanRefreshInterval (default: 5s)).
//
// If either of the queues had work to do the channel is considered "dirty".
//
// If QueueScanDirtyPercent (default: 25%) of the selected channels were dirty,
// the loop continues without sleep.
func (n *NSQD) queueScanLoop() {
   workCh := make(chan *Channel, n.getOpts().QueueScanSelectionCount)
   responseCh := make(chan bool, n.getOpts().QueueScanSelectionCount)
   closeCh := make(chan int)

   workTicker := time.NewTicker(n.getOpts().QueueScanInterval)
   refreshTicker := time.NewTicker(n.getOpts().QueueScanRefreshInterval)

   channels := n.channels()
   n.resizePool(len(channels), workCh, responseCh, closeCh)

   for {
      select {
      case <-workTicker.C:
         if len(channels) == 0 {
            continue
         }
      case <-refreshTicker.C:
         channels = n.channels()
         n.resizePool(len(channels), workCh, responseCh, closeCh)
         continue
      case <-n.exitChan:
         goto exit
      }

      num := n.getOpts().QueueScanSelectionCount
      if num > len(channels) {
         num = len(channels)
      }

   loop:
      for _, i := range util.UniqRands(num, len(channels)) {
         workCh <- channels[i]
      }

      numDirty := 0
      for i := 0; i < num; i++ {
         if <-responseCh {
            numDirty++
         }
      }

      if float64(numDirty)/float64(num) > n.getOpts().QueueScanDirtyPercent {
         goto loop
      }
   }

exit:
   n.logf(LOG_INFO, "QUEUESCAN: closing")
   close(closeCh)
   workTicker.Stop()
   refreshTicker.Stop()
}
```

#### tcpserver

```go
func TCPServer(listener net.Listener, handler TCPHandler, logf lg.AppLogFunc) error {
   logf(lg.INFO, "TCP: listening on %s", listener.Addr())

   for {
      clientConn, err := listener.Accept()
      if err != nil {
         if nerr, ok := err.(net.Error); ok && nerr.Temporary() {
            logf(lg.WARN, "temporary Accept() failure - %s", err)
            runtime.Gosched()
            continue
         }
         // theres no direct way to detect this error because it is not exposed
         if !strings.Contains(err.Error(), "use of closed network connection") {
            return fmt.Errorf("listener.Accept() error - %s", err)
         }
         break
      }
      go handler.Handle(clientConn)
   }

   logf(lg.INFO, "TCP: closing %s", listener.Addr())

   return nil
}
```

- 对每个连接判断是否是临时错误，如果是临时错误就睡眠当前goroutine等待下次运行。

- 对每一个连接开启一个协程取处理，主要包括protocol的校验等等。

- ```go
  err = prot.IOLoop(clientConn)
  ```

- 然后运行对应的协议。



nsqd 使用一个sync.once加chan确保任意一个出现错误之后的就不会运行。

```go
exitCh := make(chan error)
var once sync.Once
exitFunc := func(err error) {
   once.Do(func() {
      if err != nil {
         n.logf(LOG_FATAL, "%s", err)
      }
      exitCh <- err
   })
}
```



#### protocol_v2

在ioLoop中通过exec对每个消息，执行对应的消息。

```go
func (p *protocolV2) Exec(client *clientV2, params [][]byte) ([]byte, error) {
   if bytes.Equal(params[0], []byte("IDENTIFY")) {
      return p.IDENTIFY(client, params)
   }
   err := enforceTLSPolicy(client, p, params[0])
   if err != nil {
      return nil, err
   }
   switch {
   case bytes.Equal(params[0], []byte("FIN")):
      return p.FIN(client, params)
   case bytes.Equal(params[0], []byte("RDY")):
      return p.RDY(client, params)
   case bytes.Equal(params[0], []byte("REQ")):
      return p.REQ(client, params)
   case bytes.Equal(params[0], []byte("PUB")):
      return p.PUB(client, params)
```



#### DPUB具体处理逻辑

这里我觉得有意思的是，每一个操作由一个到client的conn以及param参数全部描述。

其中param[0]表示请求的类型。

param[1]表示topic名字。

param[2]表示timeout设置。

采用[][]byte流而不是struct结构体，可能是为了避免分配不必要的结构体？毕竟结构体字段类型会比较繁多。



```go
func (p *protocolV2) DPUB(client *clientV2, params [][]byte) ([]byte, error) {
   var err error

   if len(params) < 3 {
      return nil, protocol.NewFatalClientErr(nil, "E_INVALID", "DPUB insufficient number of parameters")
   }
```

对于每个消息，我们都会判断各种body，topicname，duration是否在最大最小值范围内，param参数长度是否不符合范围。

从reader中读取bodylen。

checkAuth检查client是否有权限在对应的topic上创建channel。

然后改变client的元数据，记录dpub了一条消息。



只有检查成功后，才会通过message的长度，分配对应大小的切片，然后使用readFull语意一次读取完毕。





