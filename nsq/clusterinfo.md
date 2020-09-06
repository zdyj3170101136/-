#### clusterinfo

nsqd会使用http开启一个新的客户

```go
httpcli := http_api.NewClient(nil, opts.HTTPClientConnectTimeout, opts.HTTPClientRequestTimeout)
n.ci = clusterinfo.New(n.logf, httpcli)
```





// 这里设置http的参数还是蛮有意思的。

比如通过transport设置http发起连接请求的最大时间2s。

以及写完数据后，最小的收到响应的时间5s/

```
// A custom http.Transport with support for deadline timeouts
func NewDeadlineTransport(connectTimeout time.Duration, requestTimeout time.Duration) *http.Transport {
   // arbitrary values copied from http.DefaultTransport
   transport := &http.Transport{
      DialContext: (&net.Dialer{
         Timeout:   connectTimeout,
         KeepAlive: 30 * time.Second,
         DualStack: true,
      }).DialContext,
      ResponseHeaderTimeout: requestTimeout,
      MaxIdleConns:          100,
      IdleConnTimeout:       90 * time.Second,
      TLSHandshakeTimeout:   10 * time.Second,
   }
   return transport
}


func NewClient(tlsConfig *tls.Config, connectTimeout time.Duration, requestTimeout time.Duration) *Client {
	transport := NewDeadlineTransport(connectTimeout, requestTimeout)
	transport.TLSClientConfig = tlsConfig
	return &Client{
		c: &http.Client{
			Transport: transport,
			Timeout:   requestTimeout,
		},
	}
}
```

而client通过newrequest生成get请求。

添加header。

读取body以及通过json解析数据。

```go
// GETV1 is a helper function to perform a V1 HTTP request
// and parse our NSQ daemon's expected response format, with deadlines.
func (c *Client) GETV1(endpoint string, v interface{}) error {
retry:
   req, err := http.NewRequest("GET", endpoint, nil)
   if err != nil {
      return err
   }

   req.Header.Add("Accept", "application/vnd.nsq; version=1.0")

   resp, err := c.c.Do(req)
   if err != nil {
      return err
   }

   body, err := ioutil.ReadAll(resp.Body)
   resp.Body.Close()
   if err != nil {
      return err
   }
   if resp.StatusCode != 200 {
      if resp.StatusCode == 403 && !strings.HasPrefix(endpoint, "https") {
         endpoint, err = httpsEndpoint(endpoint, body)
         if err != nil {
            return err
         }
         goto retry
      }
      return fmt.Errorf("got response %s %q", resp.Status, body)
   }
   err = json.Unmarshal(body, &v)
   if err != nil {
      return err
   }

   return nil
}
```

而clusterinfo。是建立在client之上的。

client包装了http的get和post连接，让我们只需要提供参数就行。

而cluterinfo表示获取信息。

其实就是一个witgroup向所有的lookupd询问。

如果没有一个询问道，那么就说明出错了。

通过mutex锁限制对errs的并发访问。



然后对tpoic进行uniq去重，以及排序。

去除重复也很简单，两轮遍历。

但是很有意思的是代码的规范性。

比如常用的entry，返回结果用r 表示result，输入为s 表示一串string。

用exiting表示已经存在的。

```go
func Uniq(s []string) (r []string) {
   for _, entry := range s {
      found := false
      for _, existing := range r {
         if existing == entry {
            found = true
            break
         }
      }
      if !found {
         r = append(r, entry)
      }
   }
   return
}
```

```go
// GetLookupdTopics returns a []string containing a union of all the topics
// from all the given nsqlookupd
func (c *ClusterInfo) GetLookupdTopics(lookupdHTTPAddrs []string) ([]string, error) {
   var topics []string
   var lock sync.Mutex
   var wg sync.WaitGroup
   var errs []error

   type respType struct {
      Topics []string `json:"topics"`
   }

   for _, addr := range lookupdHTTPAddrs {
      wg.Add(1)
      go func(addr string) {
         defer wg.Done()

         endpoint := fmt.Sprintf("http://%s/topics", addr)
         c.logf("CI: querying nsqlookupd %s", endpoint)

         var resp respType
         err := c.client.GETV1(endpoint, &resp)
         if err != nil {
            lock.Lock()
            errs = append(errs, err)
            lock.Unlock()
            return
         }

         lock.Lock()
         defer lock.Unlock()
         topics = append(topics, resp.Topics...)
      }(addr)
   }
   wg.Wait()

   if len(errs) == len(lookupdHTTPAddrs) {
      return nil, fmt.Errorf("Failed to query any nsqlookupd: %s", ErrList(errs))
   }

   topics = stringy.Uniq(topics)
   sort.Strings(topics)

   if len(errs) > 0 {
      return topics, ErrList(errs)
   }
   return topics, nil
}
```