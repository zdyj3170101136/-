#### client

```go
bodyLen, err := readLen(client.Reader, client.lenSlice)
	messageBody := make([]byte, bodyLen)
	_, err = io.ReadFull(client.Reader, messageBody)
// 然后publishmessage
func (c *clientV2) PublishedMessage(topic string, count uint64) {
	c.metaLock.Lock()
	c.pubCounts[topic] += count
	c.metaLock.Unlock()
}
```

上层protocol读取数据时，先读取长度，再通过长度分配固定大小的切片，然后直接读取全部内容。

通过一个map增加对应的topic以及message。

#### client

```go
Reader: bufio.NewReaderSize(conn, defaultBufferSize),
```

client的reader是一个大小为16kb的缓存读取器。



- 如果切片（要读取的量小于buf），则先读进buf，确保至少读取16kb，然后通过copy拷贝到切片中
- 如果大于则直接拷贝进用户切片。防止多余copy。

```go
// Read reads data into p.
// It returns the number of bytes read into p.
// The bytes are taken from at most one Read on the underlying Reader,
// hence n may be less than len(p).
// To read exactly len(p) bytes, use io.ReadFull(b, p).
// At EOF, the count will be zero and err will be io.EOF.
func (b *Reader) Read(p []byte) (n int, err error) {
   n = len(p)
   if n == 0 {
      if b.Buffered() > 0 {
         return 0, nil
      }
      return 0, b.readErr()
   }
   if b.r == b.w {
      if b.err != nil {
         return 0, b.readErr()
      }
      if len(p) >= len(b.buf) {
         // Large read, empty buffer.
         // Read directly into p to avoid copy.
         n, b.err = b.rd.Read(p)
         if n < 0 {
            panic(errNegativeRead)
         }
         if n > 0 {
            b.lastByte = int(p[n-1])
            b.lastRuneSize = -1
         }
         return n, b.readErr()
      }
      // One read.
      // Do not use b.fill, which will loop.
      b.r = 0
      b.w = 0
      n, b.err = b.rd.Read(b.buf)
      if n < 0 {
         panic(errNegativeRead)
      }
      if n == 0 {
         return 0, b.readErr()
      }
      b.w += n
   }

   // copy as much as we can
   n = copy(p, b.buf[b.r:b.w])
   b.r += n
   b.lastByte = int(b.buf[b.r-1])
   b.lastRuneSize = -1
   return n, nil
}
```





#### upgrade

reader还提供Upgrade语意。

使用snappy，tls等算法。

```go
c.Reader = bufio.NewReaderSize(snappy.NewReader(conn), defaultBufferSize)
```

#### sub

一个客户端第一次连接首先要通过sub订阅到具体的channel。

这里主要是考虑到不能加入一个正在exciting的channel或者topic。

因为addclient不会检查exitflag，即这个topic是否退出。

然后往subeventchan中传递一个channel。

会在messagepump中使用到。

```go
// This retry-loop is a work-around for a race condition, where the
// last client can leave the channel between GetChannel() and AddClient().
// Avoid adding a client to an ephemeral channel / topic which has started exiting.
var channel *Channel
for {
   topic := p.ctx.nsqd.GetTopic(topicName)
   channel = topic.GetChannel(channelName)
   if err := channel.AddClient(client.ID, client); err != nil {
      return nil, protocol.NewFatalClientErr(nil, "E_TOO_MANY_CHANNEL_CONSUMERS",
         fmt.Sprintf("channel consumers for %s:%s exceeds limit of %d",
            topicName, channelName, p.ctx.nsqd.getOpts().MaxChannelConsumers))
   }

   if (channel.ephemeral && channel.Exiting()) || (topic.ephemeral && topic.Exiting()) {
      channel.RemoveClient(client.ID)
      time.Sleep(1 * time.Millisecond)
      continue
   }
   break
}
atomic.StoreInt32(&client.State, stateSubscribed)
client.Channel = channel
// update message pump
client.SubEventChan <- channel
```

#### messagepump

```go
func (p *protocolV2) messagePump(client *clientV2, startedChan chan bool) {
   var err error
   var memoryMsgChan chan *Message
   var backendMsgChan chan []byte
   var subChannel *Channel
   // NOTE: `flusherChan` is used to bound message latency for
   // the pathological case of a channel on a low volume topic
   // with >1 clients having >1 RDY counts
   var flusherChan <-chan time.Time
   var sampleRate int32

   subEventChan := client.SubEventChan
   identifyEventChan := client.IdentifyEventChan
   outputBufferTicker := time.NewTicker(client.OutputBufferTimeout)
   heartbeatTicker := time.NewTicker(client.HeartbeatInterval)
   heartbeatChan := heartbeatTicker.C
   msgTimeout := client.MsgTimeout

   // v2 opportunistically buffers data to clients to reduce write system calls
   // we force flush in two cases:
   //    1. when the client is not ready to receive messages
   //    2. we're buffered and the channel has nothing left to send us
   //       (ie. we would block in this loop anyway)
   //
   flushed := true

   // signal to the goroutine that started the messagePump
   // that we've started up
   close(startedChan)

   for {
      if subChannel == nil || !client.IsReadyForMessages() {
         // the client is not ready to receive messages...
         memoryMsgChan = nil
         backendMsgChan = nil
         flusherChan = nil
         // force flush
         client.writeLock.Lock()
         err = client.Flush()
         client.writeLock.Unlock()
         if err != nil {
            goto exit
         }
         flushed = true
      } else if flushed {
         // last iteration we flushed...
         // do not select on the flusher ticker channel
         memoryMsgChan = subChannel.memoryMsgChan
         backendMsgChan = subChannel.backend.ReadChan()
         flusherChan = nil
      } else {
         // we're buffered (if there isn't any more data we should flush)...
         // select on the flusher ticker channel, too
         memoryMsgChan = subChannel.memoryMsgChan
         backendMsgChan = subChannel.backend.ReadChan()
         flusherChan = outputBufferTicker.C
      }

      select {
      case <-flusherChan:
         // if this case wins, we're either starved
         // or we won the race between other channels...
         // in either case, force flush
         client.writeLock.Lock()
         err = client.Flush()
         client.writeLock.Unlock()
         if err != nil {
            goto exit
         }
         flushed = true
      case <-client.ReadyStateChan:
      case subChannel = <-subEventChan:
         // you can't SUB anymore
         subEventChan = nil
      case identifyData := <-identifyEventChan:
         // you can't IDENTIFY anymore
         identifyEventChan = nil

         outputBufferTicker.Stop()
         if identifyData.OutputBufferTimeout > 0 {
            outputBufferTicker = time.NewTicker(identifyData.OutputBufferTimeout)
         }

         heartbeatTicker.Stop()
         heartbeatChan = nil
         if identifyData.HeartbeatInterval > 0 {
            heartbeatTicker = time.NewTicker(identifyData.HeartbeatInterval)
            heartbeatChan = heartbeatTicker.C
         }

         if identifyData.SampleRate > 0 {
            sampleRate = identifyData.SampleRate
         }

         msgTimeout = identifyData.MsgTimeout
      case <-heartbeatChan:
         err = p.Send(client, frameTypeResponse, heartbeatBytes)
         if err != nil {
            goto exit
         }
      case b := <-backendMsgChan:
         if sampleRate > 0 && rand.Int31n(100) > sampleRate {
            continue
         }

         msg, err := decodeMessage(b)
         if err != nil {
            p.ctx.nsqd.logf(LOG_ERROR, "failed to decode message - %s", err)
            continue
         }
         msg.Attempts++

         subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout)
         client.SendingMessage()
         err = p.SendMessage(client, msg)
         if err != nil {
            goto exit
         }
         flushed = false
      case msg := <-memoryMsgChan:
         if sampleRate > 0 && rand.Int31n(100) > sampleRate {
            continue
         }
         msg.Attempts++

         subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout)
         client.SendingMessage()
         err = p.SendMessage(client, msg)
         if err != nil {
            goto exit
         }
         flushed = false
      case <-client.ExitChan:
         goto exit
      }
   }

exit:
   p.ctx.nsqd.logf(LOG_INFO, "PROTOCOL(V2): [%s] exiting messagePump", client)
   heartbeatTicker.Stop()
   outputBufferTicker.Stop()
   if err != nil {
      p.ctx.nsqd.logf(LOG_ERROR, "PROTOCOL(V2): [%s] messagePump error - %s", client, err)
   }
}
```

- 这里通过一个startchan传递同步信息



- subeventchannel用来传递这个客户端所订阅的channel，收到之后就会被请为nil，所以一个client只能连接一个channel的连接。



- 当channel的memorymsgchan收到消息，如果samplerate不为0，那么我们通过一个随机数，决定这个消息发补发送。
- backendchan也同样处理逻辑
- 信息加入inflightchan，以一个timeout，开始发送信息



- 我们发送的信息包括一个int32表示消息类型，以及使用心跳时间设置为write超时时间。
- 我们考虑到消息会缓存到buffer里头（为了减少系统调用，每16kb才会写一次）

```go
func (p *protocolV2) Send(client *clientV2, frameType int32, data []byte) error {
   client.writeLock.Lock()

   var zeroTime time.Time
   if client.HeartbeatInterval > 0 {
      client.SetWriteDeadline(time.Now().Add(client.HeartbeatInterval))
   } else {
      client.SetWriteDeadline(zeroTime)
   }

   _, err := protocol.SendFramedResponse(client.Writer, frameType, data)
   if err != nil {
      client.writeLock.Unlock()
      return err
   }

   if frameType != frameTypeMessage {
      err = client.Flush()
   }

   client.writeLock.Unlock()

   return err
}
```

#### 调整write的缓存大小

- desiredsize为-1表示立即发送
- 0表示默认
- 以及一个上下界限校验参数合理性的判断

```go
func (c *clientV2) SetOutputBuffer(desiredSize int, desiredTimeout int) error {
   c.writeLock.Lock()
   defer c.writeLock.Unlock()

   switch {
   case desiredTimeout == -1:
      c.OutputBufferTimeout = 0
   case desiredTimeout == 0:
      // do nothing (use default)
   case true &&
      desiredTimeout >= int(c.ctx.nsqd.getOpts().MinOutputBufferTimeout/time.Millisecond) &&
      desiredTimeout <= int(c.ctx.nsqd.getOpts().MaxOutputBufferTimeout/time.Millisecond):

      c.OutputBufferTimeout = time.Duration(desiredTimeout) * time.Millisecond
   default:
      return fmt.Errorf("output buffer timeout (%d) is invalid", desiredTimeout)
   }

   switch {
   case desiredSize == -1:
      // effectively no buffer (every write will go directly to the wrapped net.Conn)
      c.OutputBufferSize = 1
      c.OutputBufferTimeout = 0
   case desiredSize == 0:
      // do nothing (use default)
   case desiredSize >= 64 && desiredSize <= int(c.ctx.nsqd.getOpts().MaxOutputBufferSize):
      c.OutputBufferSize = desiredSize
   default:
      return fmt.Errorf("output buffer size (%d) is invalid", desiredSize)
   }

   if desiredSize != 0 {
      err := c.Writer.Flush()
      if err != nil {
         return err
      }
      c.Writer = bufio.NewWriterSize(c.Conn, c.OutputBufferSize)
   }

   return nil
}
```

#### identify

传递客户端连接的元数据。

修改完后把chan变为nil，则一次传递语意。

#### flush

outputbuffer有个ticker计时器

每次时间到期就flush数据。

![截屏2020-09-04 下午12.27.30](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-04 下午12.27.30.png)



flushed标志位在flush过后变为true。

这会导致

flushchan置为nil，那么就不会flush了。



而发送数据，又会导致flushchan变为chan，flushed为false。



- 通过只在有脏数据的时候flush来减少损耗

#### readyformessage

- readycount表示愿意接受的数据量
- inflightcount表示正在发送的数据量



- 如果inflight的大于就返回flase，表示客户端还没有准备好。
- readycount只能通过客户端发送RDY消息来更新。

#### fin

通过传递fin来表示消息已经成功接受。

```go
err = client.Channel.FinishMessage(client.ID, *id)
if err != nil {
   return nil, protocol.NewClientErr(err, "E_FIN_FAILED",
      fmt.Sprintf("FIN %s failed %s", *id, err.Error()))
}

client.FinishedMessage()
```

- 这会把消息从inflightqueue中移除
- 减小inflightcount表示消息成功处理
- 增大finishcount



- 往readystatechan中发送值
- 因为messagepump可能可以发送信息了。
- 因为如果之前clientisnotready返回的是false的话，那么对应的chan就会被置为nil。
- 为了避免无穷的for循环，我们会阻塞在readystatechan中。

```go
func (c *clientV2) tryUpdateReadyState() {
   // you can always *try* to write to ReadyStateChan because in the cases
   // where you cannot the message pump loop would have iterated anyway.
   // the atomic integer operations guarantee correctness of the value.
   select {
   case c.ReadyStateChan <- 1:
   default:
   }
}
```

```go
func (c *clientV2) FinishedMessage() {
   atomic.AddUint64(&c.FinishCount, 1)
   atomic.AddInt64(&c.InFlightCount, -1)
   c.tryUpdateReadyState()
}
```

#### processinfightqueue

- 对于inflightmessage如果时间过期了，那么就重新加入memorymsgchan中
- 然后client的inflightcount减小
- 这个函数会被queuescanworker调用

```go
func (c *Channel) processInFlightQueue(t int64) bool {
   c.exitMutex.RLock()
   defer c.exitMutex.RUnlock()

   if c.Exiting() {
      return false
   }

   dirty := false
   for {
      c.inFlightMutex.Lock()
      msg, _ := c.inFlightPQ.PeekAndShift(t)
      c.inFlightMutex.Unlock()

      if msg == nil {
         goto exit
      }
      dirty = true

      _, err := c.popInFlightMessage(msg.clientID, msg.ID)
      if err != nil {
         goto exit
      }
      atomic.AddUint64(&c.timeoutCount, 1)
      c.RLock()
      client, ok := c.clients[msg.clientID]
      c.RUnlock()
      if ok {
         client.TimedOutMessage()
      }
      c.put(msg)
   }

exit:
   return dirty
}
```

#### req

```go
// RequeueMessage requeues a message based on `time.Duration`, ie:
//
// `timeoutMs` == 0 - requeue a message immediately
// `timeoutMs`  > 0 - asynchronously wait for the specified timeout
//     and requeue a message (aka "deferred requeue")
//
```

```go
err = client.Channel.RequeueMessage(client.ID, *id, timeoutDuration)
if err != nil {
   return nil, protocol.NewClientErr(err, "E_REQ_FAILED",
      fmt.Sprintf("REQ %s failed %s", *id, err.Error()))
}

client.RequeuedMessage()
```

- 客户端也可以主动requeue某条信息
- 加入deferqueue或者重新put

#### 超时设置

heartbeat为心跳时间

读取时间为两个心跳，写入时间为一个心跳。

而心跳返回为nop，真的就是空信息。

因此正是这儿设置了两个心跳间隔没有收到heartbeat（事实上是任何信息）就会error。

```go
client.SetReadDeadline(time.Now().Add(client.HeartbeatInterval * 2))
```



#### ioloop

- 在ioloop里头，我们除了开启一个messagepump协程用来发送信息。
- 我们还for循环不断的读取信息。
- 我们以“\n”作为分割符每次读起一行，也许会trim“\r”（考虑到不同操作系统的区别）
- 然后通过空格把byte流转换为切片，进行处理。

```go
// ReadSlice does not allocate new space for the data each request
// ie. the returned slice is only valid until the next call to it
line, err = client.Reader.ReadSlice('\n')
if err != nil {
   if err == io.EOF {
      err = nil
   } else {
      err = fmt.Errorf("failed to read command - %s", err)
   }
   break
}

// trim the '\n'
line = line[:len(line)-1]
// optionally trim the '\r'
if len(line) > 0 && line[len(line)-1] == '\r' {
   line = line[:len(line)-1]
}
params := bytes.Split(line, separatorBytes)

p.ctx.nsqd.logf(LOG_DEBUG, "PROTOCOL(V2): [%s] %s", client, params)

var response []byte
response, err = p.Exec(client, params)
if err != nil {
   ctx := ""
   if parentErr := err.(protocol.ChildErr).Parent(); parentErr != nil {
      ctx = " - " + parentErr.Error()
   }
   p.ctx.nsqd.logf(LOG_ERROR, "[%s] - %s%s", client, err, ctx)

   sendErr := p.Send(client, frameTypeError, []byte(err.Error()))
   if sendErr != nil {
      p.ctx.nsqd.logf(LOG_ERROR, "[%s] - %s%s", client, sendErr, ctx)
      break
   }

   // errors of type FatalClientErr should forceably close the connection
   if _, ok := err.(*protocol.FatalClientErr); ok {
      break
   }
   continue
}

if response != nil {
   err = p.Send(client, frameTypeResponse, response)
   if err != nil {
      err = fmt.Errorf("failed to send response - %s", err)
      break
   }
}
```