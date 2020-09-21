#### lookup 

命令模式。

每一个命令都是一个操作：请求的一方发出请求要求执行一个操作；接收的一方接收到请求，并执行操作。命令模式允许请求的一方和接收的一方独立开来，使得请求的一方不必知道接收请求的一方的接口，更不必知道请求是怎么被接收、以及操作是否被执行、何时被执行、怎么被执行的。

命令模式涉及到五个角色，他们分别是：

- **客户端角色(Client)**：创建一个具体命令`ConcreteCommand`对象并确定其接收者。
- **命令角色(Command)**： 声明一个给所有具体命令类的抽象接口。
- **具体命令角色(ConcreteCommand)**：定义一个接收者和行为之间的弱耦合；实现`execute()`方法，负责调用接收者的相应操作。`execute()`方法通常叫做执行方法。
- **请求者角色(Invoker)**：负责调用命令对象执行请求，相关的方法叫做行动方法。
- **接收者角色(Receiver)**：负责具体实施和执行一个请求。任何一个类都可以成为接收者，实施和执行请求的方法叫做行动方法。



protocol那边的execute函数接受和处理命令。

客户端那边通过command，生成具体的命令。然后通过nsqlookupd生成具体的函数发送出去。



作者：步积
链接：https://www.jianshu.com/p/5901e76a6348
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

把不同的

```go
case <-n.optsNotificationChan:
   var tmpPeers []*lookupPeer
   var tmpAddrs []string
   for _, lp := range lookupPeers {
      if in(lp.addr, n.getOpts().NSQLookupdTCPAddresses) {
         tmpPeers = append(tmpPeers, lp)
         tmpAddrs = append(tmpAddrs, lp.addr)
         continue
      }
      n.logf(LOG_INFO, "LOOKUP(%s): removing peer", lp)
      lp.Close()
   }
   lookupPeers = tmpPeers
   lookupAddrs = tmpAddrs
   connect = true

func in(s string, lst []string) bool {
	for _, v := range lst {
		if s == v {
			return true
		}
	}
	return false
}
```

- 当nsqd更新了lookupd的地址的时候，通过chan通知lookuploop
- 我们收到后，保存有效的地址。失效的关闭

```go
if connect {
			for _, host := range n.getOpts().NSQLookupdTCPAddresses {
				if in(host, lookupAddrs) {
					continue
				}
				n.logf(LOG_INFO, "LOOKUP(%s): adding peer", host)
				lookupPeer := newLookupPeer(host, n.getOpts().MaxBodySize, n.logf,
					connectCallback(n, hostname))
				lookupPeer.Command(nil) // start the connection
				lookupPeers = append(lookupPeers, lookupPeer)
				lookupAddrs = append(lookupAddrs, host)
			}
			n.lookupPeers.Store(lookupPeers)
			connect = false
		}
```

- 而connect标志位，则表示对于新的lookupd要建立连接



- 这里拆分成两段，主要是为了解藕，一段专门处理关闭，一段处理新建连接。

#### channel和topic的创建和拆除

新建channel或者topic。都会调用nsqd的notify函数。

- 通知nsqdlookup那边新的topic或者channel的创建。
- 如果不是正在加载元数据，那么就持久化元数据到内存。

```go

func (n *NSQD) Notify(v interface{}) {
	// since the in-memory metadata is incomplete,
	// should not persist metadata while loading it.
	// nsqd will call `PersistMetadata` it after loading
	persist := atomic.LoadInt32(&n.isLoading) == 0
	n.waitGroup.Wrap(func() {
		// by selecting on exitChan we guarantee that
		// we do not block exit, see issue #123
		select {
		case <-n.exitChan:
		case n.notifyChan <- v:
			if !persist {
				return
			}
			n.Lock()
			err := n.PersistMetadata()
			if err != nil {
				n.logf(LOG_ERROR, "failed to persist metadata - %s", err)
			}
			n.Unlock()
		}
	})
}
```

isloading用来判断nsqd是否正在加载元数据。

loadmetadata会在启动nsqd的时候调用。

去请求所有的topic和channel。

```go

func (n *NSQD) LoadMetadata() error {
	atomic.StoreInt32(&n.isLoading, 1)
	defer atomic.StoreInt32(&n.isLoading, 0)

	fn := newMetadataFile(n.getOpts())

	data, err := readOrEmpty(fn)
	if err != nil {
		return err
	}
	if data == nil {
		return nil // fresh start
	}

	var m meta
	err = json.Unmarshal(data, &m)
	if err != nil {
		return fmt.Errorf("failed to parse metadata in %s - %s", fn, err)
	}

	for _, t := range m.Topics {
		if !protocol.IsValidTopicName(t.Name) {
			n.logf(LOG_WARN, "skipping creation of invalid topic %s", t.Name)
			continue
		}
		topic := n.GetTopic(t.Name)
		if t.Paused {
			topic.Pause()
		}
		for _, c := range t.Channels {
			if !protocol.IsValidChannelName(c.Name) {
				n.logf(LOG_WARN, "skipping creation of invalid channel %s", c.Name)
				continue
			}
			channel := topic.GetChannel(c.Name)
			if c.Paused {
				channel.Pause()
			}
		}
		topic.Start()
	}
	return nil
}

```



而在nsqlookupd那边则会通过type类型判断调用nsq去unregister对应的topic。

告诉所有的nsqlookupd对应的topic或者channel已经消失。

有趣的是这里采用的是cmd包装的形式。

```go
case val := <-n.notifyChan:
   var cmd *nsq.Command
   var branch string

   switch val.(type) {
   case *Channel:
      // notify all nsqlookupds that a new channel exists, or that it's removed
      branch = "channel"
      channel := val.(*Channel)
      if channel.Exiting() == true {
         cmd = nsq.UnRegister(channel.topicName, channel.name)
      } else {
         cmd = nsq.Register(channel.topicName, channel.name)
      }
   case *Topic:
      // notify all nsqlookupds that a new topic exists, or that it's removed
      branch = "topic"
      topic := val.(*Topic)
      if topic.Exiting() == true {
         cmd = nsq.UnRegister(topic.name, "")
      } else {
         cmd = nsq.Register(topic.name, "")
      }
   }

   for _, lookupPeer := range lookupPeers {
      n.logf(LOG_INFO, "LOOKUPD(%s): %s %s", lookupPeer, branch, cmd)
      _, err := lookupPeer.Command(cmd)
      if err != nil {
         n.logf(LOG_ERROR, "LOOKUPD(%s): %s - %s", lookupPeer, cmd, err)
      }
   }
```

- command语句通过一个状态机来判断。
- 如果不是已连接状态则重新连接（1s的连接timeout）
- 如果是从未连接变为已连接，那么则调用callback函数。（callback主要处理向lookupd询问信息，然后通知它所有的topic和channel）
- 然后写入cmd的数据到连接中，当然啦write和read都是有timeout的。
- 如果失败，简单的从connected状态变为disconnected，等待下一次调用。



- 上层可以简单的轮询对应的command



- 这里其实只有statedisconnect和stateconnect两种状态给nsqlookupd。
- 所以这里就很奇怪。。。

```go

// Command performs a round-trip for the specified Command.
//
// It will lazily connect to nsqlookupd and gracefully handle
// reconnecting in the event of a failure.
//
// It returns the response from nsqlookupd as []byte
func (lp *lookupPeer) Command(cmd *nsq.Command) ([]byte, error) {
	initialState := lp.state
	if lp.state != stateConnected {
		err := lp.Connect()
		if err != nil {
			return nil, err
		}
		lp.state = stateConnected
		_, err = lp.Write(nsq.MagicV1)
		if err != nil {
			lp.Close()
			return nil, err
		}
		if initialState == stateDisconnected {
			lp.connectCallback(lp)
		}
		if lp.state != stateConnected {
			return nil, fmt.Errorf("lookupPeer connectCallback() failed")
		}
	}
	if cmd == nil {
		return nil, nil
	}
	_, err := cmd.WriteTo(lp)
	if err != nil {
		lp.Close()
		return nil, err
	}
	resp, err := readResponseBounded(lp, lp.maxBodySize)
	if err != nil {
		lp.Close()
		return nil, err
	}
	return resp, nil
}

```

#### 心跳

没隔15s，将会发送一个心跳包，维护topic和channel的连接的持久性。

```go
case <-ticker:
			// send a heartbeat and read a response (read detects closed conns)
			for _, lookupPeer := range lookupPeers {
				n.logf(LOG_DEBUG, "LOOKUPD(%s): sending heartbeat", lookupPeer)
				cmd := nsq.Ping()
				_, err := lookupPeer.Command(cmd)
				if err != nil {
					n.logf(LOG_ERROR, "LOOKUPD(%s): %s - %s", lookupPeer, cmd, err)
				}
			}
```