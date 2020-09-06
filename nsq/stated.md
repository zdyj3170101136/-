#### state

state会开启一个60s的定时器。

然后我们使用udp开启开启一个时长为1s的连接。



```go
    case <-ticker.C:
      addr := n.getOpts().StatsdAddress
      prefix := n.getOpts().StatsdPrefix
      conn, err := net.DialTimeout("udp", addr, time.Second)
      if err != nil {
         n.logf(LOG_ERROR, "failed to create UDP socket to statsd(%s)", addr)
         continue
      }
      sw := writers.NewSpreadWriter(conn, interval-time.Second, n.exitChan)
      bw := writers.NewBoundaryBufferedWriter(sw, n.getOpts().StatsdUDPPacketSize)
      client := statsd.NewClient(bw, prefix)

      n.logf(LOG_INFO, "STATSD: pushing stats to %s", addr)

      stats := n.GetStats("", "", false)
      for _, topic := range stats {
         // try to find the topic in the last collection
         lastTopic := TopicStats{}
         for _, checkTopic := range lastStats {
            if topic.TopicName == checkTopic.TopicName {
               lastTopic = checkTopic
               break
            }
         }
         diff := topic.MessageCount - lastTopic.MessageCount
         stat := fmt.Sprintf("topic.%s.message_count", topic.TopicName)
         client.Incr(stat, int64(diff))

         diff = topic.MessageBytes - lastTopic.MessageBytes
         stat = fmt.Sprintf("topic.%s.message_bytes", topic.TopicName)
         client.Incr(stat, int64(diff))

         stat = fmt.Sprintf("topic.%s.depth", topic.TopicName)
         client.Gauge(stat, topic.Depth)

         stat = fmt.Sprintf("topic.%s.backend_depth", topic.TopicName)
         client.Gauge(stat, topic.BackendDepth)

         for _, item := range topic.E2eProcessingLatency.Percentiles {
            stat = fmt.Sprintf("topic.%s.e2e_processing_latency_%.0f", topic.TopicName, item["quantile"]*100.0)
            // We can cast the value to int64 since a value of 1 is the
            // minimum resolution we will have, so there is no loss of
            // accuracy
            client.Gauge(stat, int64(item["value"]))
         }

         for _, channel := range topic.Channels {
            // try to find the channel in the last collection
            lastChannel := ChannelStats{}
            for _, checkChannel := range lastTopic.Channels {
               if channel.ChannelName == checkChannel.ChannelName {
                  lastChannel = checkChannel
                  break
               }
            }
            diff := channel.MessageCount - lastChannel.MessageCount
            stat := fmt.Sprintf("topic.%s.channel.%s.message_count", topic.TopicName, channel.ChannelName)
            client.Incr(stat, int64(diff))

            stat = fmt.Sprintf("topic.%s.channel.%s.depth", topic.TopicName, channel.ChannelName)
            client.Gauge(stat, channel.Depth)

            stat = fmt.Sprintf("topic.%s.channel.%s.backend_depth", topic.TopicName, channel.ChannelName)
            client.Gauge(stat, channel.BackendDepth)

            stat = fmt.Sprintf("topic.%s.channel.%s.in_flight_count", topic.TopicName, channel.ChannelName)
            client.Gauge(stat, int64(channel.InFlightCount))

            stat = fmt.Sprintf("topic.%s.channel.%s.deferred_count", topic.TopicName, channel.ChannelName)
            client.Gauge(stat, int64(channel.DeferredCount))

            diff = channel.RequeueCount - lastChannel.RequeueCount
            stat = fmt.Sprintf("topic.%s.channel.%s.requeue_count", topic.TopicName, channel.ChannelName)
            client.Incr(stat, int64(diff))

            diff = channel.TimeoutCount - lastChannel.TimeoutCount
            stat = fmt.Sprintf("topic.%s.channel.%s.timeout_count", topic.TopicName, channel.ChannelName)
            client.Incr(stat, int64(diff))

            stat = fmt.Sprintf("topic.%s.channel.%s.clients", topic.TopicName, channel.ChannelName)
            client.Gauge(stat, int64(channel.ClientCount))

            for _, item := range channel.E2eProcessingLatency.Percentiles {
               stat = fmt.Sprintf("topic.%s.channel.%s.e2e_processing_latency_%.0f", topic.TopicName, channel.ChannelName, item["quantile"]*100.0)
               client.Gauge(stat, int64(item["value"]))
            }
         }
      }
      lastStats = stats

      if n.getOpts().StatsdMemStats {
         ms := getMemStats()

         client.Gauge("mem.heap_objects", int64(ms.HeapObjects))
         client.Gauge("mem.heap_idle_bytes", int64(ms.HeapIdleBytes))
         client.Gauge("mem.heap_in_use_bytes", int64(ms.HeapInUseBytes))
         client.Gauge("mem.heap_released_bytes", int64(ms.HeapReleasedBytes))
         client.Gauge("mem.gc_pause_usec_100", int64(ms.GCPauseUsec100))
         client.Gauge("mem.gc_pause_usec_99", int64(ms.GCPauseUsec99))
         client.Gauge("mem.gc_pause_usec_95", int64(ms.GCPauseUsec95))
         client.Gauge("mem.next_gc_bytes", int64(ms.NextGCBytes))
         client.Incr("mem.gc_runs", int64(ms.GCTotalRuns-lastMemStats.GCTotalRuns))

         lastMemStats = ms
      }

      bw.Flush()
      sw.Flush()
      conn.Close()
   }
}
```

#### spreadwriter

这个spreadwriter，传递的是多个byte切片。

flush数据的流程有所不同。

这里interval为60 - 1等于59s。

说明flush的时候比方说如果buf的大小为10.

那么就创建一个为5.9s的定时器。

然后可以说是每5.9s写入一部分byte流数据。



这样数据就会在59s内平缓的流动。

而spreadwriter的write仅仅是简单的内存copy。

```go
func (s *SpreadWriter) Write(p []byte) (int, error) {
   b := make([]byte, len(p))
   copy(b, p)
   s.buf = append(s.buf, b)
   return len(p), nil
}
```

```go
type SpreadWriter struct {
   w        io.Writer
   interval time.Duration
   buf      [][]byte
   exitCh   chan int
}
```

```go
func (s *SpreadWriter) Flush() {
   sleep := s.interval / time.Duration(len(s.buf))
   ticker := time.NewTicker(sleep)
   for _, b := range s.buf {
      s.w.Write(b)
      select {
      case <-ticker.C:
      case <-s.exitCh: // skip sleeps finish writes
      }
   }
   ticker.Stop()
   s.buf = s.buf[:0]
}
```

#### boundarybuffer

boundarybuffer是写入数据的时候有一个缓冲区。

但是缓冲区是一个大小为508字节的buffer。

写入的时候先写入这个508字节的缓冲区，如果缓冲区大小超过了508字节那么就会flush。

但是此时的flush仅仅是把底层buffer的数据写入spreadwriter。



而在stateloop中，则是在最末尾的时候才flush spreadwriter。

```go

func NewBoundaryBufferedWriter(w io.Writer, size int) *BoundaryBufferedWriter {
	return &BoundaryBufferedWriter{
		bw: bufio.NewWriterSize(w, size),
	}
}

func (b *BoundaryBufferedWriter) Write(p []byte) (int, error) {
	if len(p) > b.bw.Available() {
		err := b.bw.Flush()
		if err != nil {
			return 0, err
		}
	}
	return b.bw.Write(p)
}
```

#### stase client

这里的解藕也很值得学习。

send仅仅处理数据的发送。

而发送不同类型的数据则由上层负责。

```go

func (c *Client) Timing(stat string, delta int64) error {
	return c.send(stat, "%d|ms", delta)
}

func (c *Client) Gauge(stat string, value int64) error {
	return c.send(stat, "%d|g", value)
}

func (c *Client) send(stat string, format string, value int64) error {
	format = fmt.Sprintf("%s%s:%s\n", c.prefix, stat, format)
	_, err := fmt.Fprintf(c.w, format, value)
	return err
}

```

#### topicstates

首先我们得到stats为topicstates的切片，包括的所有topic的topicState。

然后我们把增量写入。

```go
diff = topic.MessageBytes - lastTopic.MessageBytes
				stat = fmt.Sprintf("topic.%s.message_bytes", topic.TopicName)
				client.Incr(stat, int64(diff))
```



我们还会保存上一轮循环的state 叫做lastStats。

然后获取lasttopic。

这里命名挺有意思的，checktopic。

```go
for _, checkTopic := range lastStats {
   if topic.TopicName == checkTopic.TopicName {
      lastTopic = checkTopic
      break
   }
}
```

#### memstate

当然，我们还会推送一些有关gc的信息上去。

```go
if n.getOpts().StatsdMemStats {
   ms := getMemStats()

   client.Gauge("mem.heap_objects", int64(ms.HeapObjects))
   client.Gauge("mem.heap_idle_bytes", int64(ms.HeapIdleBytes))
   client.Gauge("mem.heap_in_use_bytes", int64(ms.HeapInUseBytes))
   client.Gauge("mem.heap_released_bytes", int64(ms.HeapReleasedBytes))
   client.Gauge("mem.gc_pause_usec_100", int64(ms.GCPauseUsec100))
   client.Gauge("mem.gc_pause_usec_99", int64(ms.GCPauseUsec99))
   client.Gauge("mem.gc_pause_usec_95", int64(ms.GCPauseUsec95))
   client.Gauge("mem.next_gc_bytes", int64(ms.NextGCBytes))
   client.Incr("mem.gc_runs", int64(ms.GCTotalRuns-lastMemStats.GCTotalRuns))

   lastMemStats = ms
}
```

#### Getstats

其实就是加上读锁。

从topicmap里头取出realtopic。

从realtopic里头取出所有的realchannel。

收集每一个realchannel的clients。

生成channelstats。



一个topic的所有channel的channelstats收集成为topicstats。

```go
channels = append(channels, NewChannelStats(c, clients, clientCount))
```

```go
topics = append(topics, NewTopicStats(t, channels))
```

#### channelstate

通过锁和原子操作获取真实值。

除了一个很有意思的result结构体。



这个结构体会在finishmessage的时候插入对应message的时间戳。

```go
// FinishMessage successfully discards an in-flight message
func (c *Channel) FinishMessage(clientID int64, id MessageID) error {
   msg, err := c.popInFlightMessage(clientID, id)
   if err != nil {
      return err
   }
   c.removeFromInFlightPQ(msg)
   if c.e2eProcessingLatencyStream != nil {
      c.e2eProcessingLatencyStream.Insert(msg.Timestamp)
   }
   return nil
}
```

```go

func NewChannelStats(c *Channel, clients []ClientStats, clientCount int) ChannelStats {
	c.inFlightMutex.Lock()
	inflight := len(c.inFlightMessages)
	c.inFlightMutex.Unlock()
	c.deferredMutex.Lock()
	deferred := len(c.deferredMessages)
	c.deferredMutex.Unlock()

	return ChannelStats{
		ChannelName:   c.name,
		Depth:         c.Depth(),
		BackendDepth:  c.backend.Depth(),
		InFlightCount: inflight,
		DeferredCount: deferred,
		MessageCount:  atomic.LoadUint64(&c.messageCount),
		RequeueCount:  atomic.LoadUint64(&c.requeueCount),
		TimeoutCount:  atomic.LoadUint64(&c.timeoutCount),
		ClientCount:   clientCount,
		Clients:       clients,
		Paused:        c.IsPaused(),

		E2eProcessingLatency: c.e2eProcessingLatencyStream.Result(),
	}
}

```

#### quantile

```h g
// e2e message latency 10min
	E2EProcessingLatencyWindowTime  time.Duration `flag:"e2e-processing-latency-window-time"`
	E2EProcessingLatencyPercentiles []float64     `flag:"e2e-processing-latency-percentile" cfg:"e2e_processing_latency_percentiles"`

```

```go
func (q *Quantile) IsDataStale(now time.Time) bool {
   return now.After(q.lastMoveWindow.Add(q.MoveWindowTime))
}
```

```go
func (q *Quantile) Insert(msgStartTime int64) {
   q.Lock()

   now := time.Now()
   for q.IsDataStale(now) {
      q.moveWindow()
   }

   q.currentStream.Insert(float64(now.UnixNano() - msgStartTime))
   q.Unlock()
}
```

insert的时候，检查现在是否相对于上一次movewindow的时间大于五分钟。如果大于的话则movewindow，移动到另一个stream里头去。

然后往currentstream插入当前时间与msgstarttime的时间差。

这表示发送这个message的时间使用量为多少。

```go
func (q *Quantile) moveWindow() {
   q.currentIndex ^= 0x1
   q.currentStream = &q.streams[q.currentIndex]
   q.lastMoveWindow = q.lastMoveWindow.Add(q.MoveWindowTime)
   q.currentStream.Reset()
}
```