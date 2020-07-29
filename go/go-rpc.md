```
// Call represents an active RPC.
type Call struct {
   ServiceMethod string      // The name of the service and method to call.
   Args          interface{} // The argument to the function (*struct).
   Reply         interface{} // The reply from the function (*struct).
   Error         error       // After completion, the error status.
   Done          chan *Call  // Strobes when call is complete.
}
```

#### call结构体

- string，表示要调用的服务以及对应的方法
  - 服务表示一个type a int，然后实现了很多方法。
- args，参数的指针（struct）
- done 是一个channel，当完成了之后会返回call这个结构体
- reply指向返回的指针（struct）



#### client

client会有一个自增的seq。

对于每一个请求。

在map里头包存 seq - call，

```
seq := client.seq
client.seq++
client.pending[seq] = call
```



#### send

seng使用序列化的方式发送：

- seq
- servicemethod：string
- 调用的参数

```go
// Encode and send the request.
client.request.Seq = seq
client.request.ServiceMethod = call.ServiceMethod
err := client.codec.WriteRequest(&client.request, call.Args)
```

#### client

client启动的时候开启一个协程，处理输入请求。



client会往一个ip地址建立http链接。

- 如何知道是rpc请求呢

```
io.WriteString(conn, "CONNECT "+path+" HTTP/1.0\n\n")
```

- ```
  
  DefaultRPCPath   = "/_goRPC_"
  ```

  - 收到response后，

  - 从response的header中解出sequence，找到对应的map里头的call结构体。

  - 然后把resonse返回值解码到对应的call。

  - 最后往call里头的done 这个chan发送call自己

```
func (client *Client) input() {
   var err error
   var response Response
   for err == nil {
      response = Response{}
      err = client.codec.ReadResponseHeader(&response)
      if err != nil {
         break
      }
      seq := response.Seq
      client.mutex.Lock()
      call := client.pending[seq]
      delete(client.pending, seq)
      client.mutex.Unlock()

      switch {
      case call == nil:
         // We've got no pending call. That usually means that
         // WriteRequest partially failed, and call was already
         // removed; response is a server telling us about an
         // error reading request body. We should still attempt
         // to read error body, but there's no one to give it to.
         err = client.codec.ReadResponseBody(nil)
         if err != nil {
            err = errors.New("reading error body: " + err.Error())
         }
      case response.Error != "":
         // We've got an error response. Give this to the request;
         // any subsequent requests will get the ReadResponseBody
         // error if there is one.
         call.Error = ServerError(response.Error)
         err = client.codec.ReadResponseBody(nil)
         if err != nil {
            err = errors.New("reading error body: " + err.Error())
         }
         call.done()
      default:
         err = client.codec.ReadResponseBody(call.Reply)
         if err != nil {
            call.Error = errors.New("reading body " + err.Error())
         }
         call.done()
      }
   }
   // Terminate pending calls.
   client.reqMutex.Lock()
   client.mutex.Lock()
   client.shutdown = true
   closing := client.closing
   if err == io.EOF {
      if closing {
         err = ErrShutdown
      } else {
         err = io.ErrUnexpectedEOF
      }
   }
   for _, call := range client.pending {
      call.Error = err
      call.done()
   }
   client.mutex.Unlock()
   client.reqMutex.Unlock()
   if debugLog && err != io.EOF && !closing {
      log.Println("rpc: client protocol error:", err)
   }
}
```



#### server

**func** (t *Arith) **Multiply**(args *Args, reply ***int**) **error** {    *reply = args.A * args.B    **return** nil }

- 服务实现的方法，也就是函数，必须要有两个参数，而且都是指针

        - 一个是函数入参
        - 一个是返回值
        - 函数本身返回error



#### register

服务首先要注册，我们传入的是一个服务的指针。

但是调用方它是用这个string对吧。



- name，表示服务的名字，通过reflect得到
- rcvr，方法的receiver，也就是服务的指针。
- 然后一个map通过reflect方法得到所有的方法的名字，以及对应的methodType
- type，传进来的指针的reflect.type，用于获取这个指针对应的所有方法。（这就也包括结构体和指针都实现的方法）



```
type service struct {
   name   string                 // name of service
   rcvr   reflect.Value          // receiver of methods for the service
   typ    reflect.Type           // type of the receiver
   method map[string]*methodType // registered methods
}
```



#### servicemap

```
serviceMap sync.Map   // map[string]*service
```

service那边会用服务的名字保存对应的这个service结构体。



#### 收到request之后

- ```
  dot := strings.LastIndex(req.ServiceMethod, ".")
  if dot < 0 {
     err = errors.New("rpc: service/method request ill-formed: " + req.ServiceMethod)
     return
  }
  serviceName := req.ServiceMethod[:dot]
  methodName := req.ServiceMethod[dot+1:]
  ```



- 以.作为分割符，得到这个service的名字，以及method的名字。
- 然后通过servicemap得到对应的service结构体
- 通过service结构体得到对应的method结构体

```
// Look up the request.
svci, ok := server.serviceMap.Load(serviceName)
if !ok {
   err = errors.New("rpc: can't find service " + req.ServiceMethod)
   return
}
svc = svci.(*service)
mtype = svc.method[methodName]
```



- requestHeader中
- 从reuqestBOdy里头得到参数，然后通过对应的methodtype，调用函数



https://juejin.im/post/5d1760455188255cfc1a019f#heading-2