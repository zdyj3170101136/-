#### infilghtPqueue

以priority建立的小顶堆。



- push的时候如果len将要大于cap，分配两倍的cap，维持len保持不变。
- pop的时候，如果len小于二分之一cap，并且cap大于25.分配二分之一cap且len不变。



- 增加了peekAndShift意思。如果最小的priority小于max，那么就pop。否则返回nil



#### topic

topic把消息复制给所有的channel

如果messageDefer不为0，则加入deferqueue队列。

（DPUB，客户端发送一个延迟的消息。）

```go
// messagePump selects over the in-memory and backend queue and
// writes messages to every channel for this topic
func (t *Topic) messagePump() {
   var msg *Message
   var buf []byte
   var err error
   var chans []*Channel
   var memoryMsgChan chan *Message
   var backendChan chan []byte

   // do not pass messages before Start(), but avoid blocking Pause() or GetChannel()
   for {
      select {
      case <-t.channelUpdateChan:
         continue
      case <-t.pauseChan:
         continue
      case <-t.exitChan:
         goto exit
      case <-t.startChan:
      }
      break
   }
   t.RLock()
   for _, c := range t.channelMap {
      chans = append(chans, c)
   }
   t.RUnlock()
   if len(chans) > 0 && !t.IsPaused() {
      memoryMsgChan = t.memoryMsgChan
      backendChan = t.backend.ReadChan()
   }

   // main message loop
   for {
      select {
      case msg = <-memoryMsgChan:
      case buf = <-backendChan:
         msg, err = decodeMessage(buf)
         if err != nil {
            t.ctx.nsqd.logf(LOG_ERROR, "failed to decode message - %s", err)
            continue
         }
      case <-t.channelUpdateChan:
         chans = chans[:0]
         t.RLock()
         for _, c := range t.channelMap {
            chans = append(chans, c)
         }
         t.RUnlock()
         if len(chans) == 0 || t.IsPaused() {
            memoryMsgChan = nil
            backendChan = nil
         } else {
            memoryMsgChan = t.memoryMsgChan
            backendChan = t.backend.ReadChan()
         }
         continue
      case <-t.pauseChan:
         if len(chans) == 0 || t.IsPaused() {
            memoryMsgChan = nil
            backendChan = nil
         } else {
            memoryMsgChan = t.memoryMsgChan
            backendChan = t.backend.ReadChan()
         }
         continue
      case <-t.exitChan:
         goto exit
      }

      for i, channel := range chans {
         chanMsg := msg
         // copy the message because each channel
         // needs a unique instance but...
         // fastpath to avoid copy if its the first channel
         // (the topic already created the first copy)
         if i > 0 {
            chanMsg = NewMessage(msg.ID, msg.Body)
            chanMsg.Timestamp = msg.Timestamp
            chanMsg.deferred = msg.deferred
         }
         if chanMsg.deferred != 0 {
            channel.PutMessageDeferred(chanMsg, chanMsg.deferred)
            continue
         }
         err := channel.PutMessage(chanMsg)
         if err != nil {
            t.ctx.nsqd.logf(LOG_ERROR,
               "TOPIC(%s) ERROR: failed to put msg(%s) to channel(%s) - %s",
               t.name, msg.ID, channel.name, err)
         }
      }
   }

exit:
   t.ctx.nsqd.logf(LOG_INFO, "TOPIC(%s): closing ... messagePump", t.name)
}
```

#### 未start前

```go
// do not pass messages before Start(), but avoid blocking Pause() or GetChannel()
```

#### start后

从memorymsgchan中获取msg或者从backendchan中decode出msg。

- channelupdate，重新从channelMap中获取chan
- 如果已经暂停，置messagechan为nil，这样发送端会阻塞。

```go
// copy the message because each channel
// needs a unique instance but...
// fastpath to avoid copy if its the first channel
// (the topic already created the first copy)
```

对于每个msg，遍历chan，发送copy后的结构体。

保证互不影响。对于第一个不用copy。

保证unique message。

#### PutMessage

- 使用messageBytes和messageCount两个原子表示。为减少损耗，增加总量。
- 使用exitFlag判断是否exit，若是则退出



- memoryMsgChan长度一般为10000，表示最多chan中有这么多的msg

```go
func (t *Topic) put(m *Message) error {
   select {
   case t.memoryMsgChan <- m:
   default:
      b := bufferPoolGet()
      err := writeMessageToBackend(b, m, t.backend)
      bufferPoolPut(b)
      t.ctx.nsqd.SetHealth(err)
      if err != nil {
         t.ctx.nsqd.logf(LOG_ERROR,
            "TOPIC(%s) ERROR: failed to write message to backend - %s",
            t.name, err)
         return err
      }
   }
   return nil
}
```



```go
func writeMessageToBackend(buf *bytes.Buffer, msg *Message, bq BackendQueue) error {
   buf.Reset()
   _, err := msg.WriteTo(buf)
   if err != nil {
      return err
   }
   return bq.Put(buf.Bytes())
}
```

使用buffer将msg转换为byte流写入backend。



- 而backend又有chan可读取。
- backend如果topicname带有后缀ephemeral短暂的，则直接丢弃。
- 如果不是，则为diskqueue。





#### channelMap

```go
// DeleteExistingChannel removes a channel from the topic only if it exists
func (t *Topic) DeleteExistingChannel(channelName string) error {
   t.Lock()
   channel, ok := t.channelMap[channelName]
   if !ok {
      t.Unlock()
      return errors.New("channel does not exist")
   }
   delete(t.channelMap, channelName)
   // not defered so that we can continue while the channel async closes
   numChannels := len(t.channelMap)
   t.Unlock()

   t.ctx.nsqd.logf(LOG_INFO, "TOPIC(%s): deleting channel %s", t.name, channel.name)

   // delete empties the channel before closing
   // (so that we dont leave any messages around)
   channel.Delete()

   // update messagePump state
   select {
   case t.channelUpdateChan <- 1:
   case <-t.exitChan:
   }

   if numChannels == 0 && t.ephemeral == true {
      go t.deleter.Do(func() { t.deleteCallback(t) })
   }

   return nil
}
```

- messagePump使用读锁访问channelmap。
- 外部使用写锁修改channelMap，然后通过channelUpdateChan通知。

#### empty

```go
func (t *Topic) Empty() error {
   for {
      select {
      case <-t.memoryMsgChan:
      default:
         goto finish
      }
   }

finish:
   return t.backend.Empty()
}
```

清空memoryMsgChan,如果清空完了则finish

#### exit

```go
func (t *Topic) exit(deleted bool) error {
   if !atomic.CompareAndSwapInt32(&t.exitFlag, 0, 1) {
      return errors.New("exiting")
   }

   if deleted {
      t.ctx.nsqd.logf(LOG_INFO, "TOPIC(%s): deleting", t.name)

      // since we are explicitly deleting a topic (not just at system exit time)
      // de-register this from the lookupd
      t.ctx.nsqd.Notify(t)
   } else {
      t.ctx.nsqd.logf(LOG_INFO, "TOPIC(%s): closing", t.name)
   }

   close(t.exitChan)

   // synchronize the close of messagePump()
   t.waitGroup.Wait()

   if deleted {
      t.Lock()
      for _, channel := range t.channelMap {
         delete(t.channelMap, channel.name)
         channel.Delete()
      }
      t.Unlock()

      // empty the queue (deletes the backend files, too)
      t.Empty()
      return t.backend.Delete()
   }

   // close all the channels
   for _, channel := range t.channelMap {
      err := channel.Close()
      if err != nil {
         // we need to continue regardless of error to close all the channels
         t.ctx.nsqd.logf(LOG_ERROR, "channel(%s) close - %s", channel.name, err)
      }
   }

   // write anything leftover to disk
   t.flush()
   return t.backend.Close()
}
```

- exit通过exitflag标示位不让putmessage运行。
- 通过exitChan关闭messagePump，等待messagePUMP关闭完毕。
- 关闭channelMap
- flush磁盘
- 关闭backend



- 为什么要有关闭标志位。
- 如果直接关闭exitChan，由于执行是select随机的，无法保证。
- 而exitChan会不处理已经有的数据。



#### benchmark

一个benchamrkPut测试put到channel然后直接close的速度。

模拟的是正常运行然后关闭。



一个benchmarkTopicToChannel。

不同之处是加了一个调度器sched直到所有数据都到了memoryMsgChan。（给其他goroutine运行的机会，比如messsagePUMP）

我们可以看出来这一步的速度还是比较长的。



模拟的是正常运行时。



数据有两个流通。

一个topic里头，在put函数里进入memoryMsgchan，topic另一个协程取出来然后放进channel的memoryMsgChan中。

```go
for {
   if len(channel.memoryMsgChan) == b.N {
      break
   }
   runtime.Gosched()
}
```

BenchmarkTopicToChannelPut-8   	   34058	     41735 ns/op

BenchmarkTopicPut-8   	   37386	     29340 ns/op

