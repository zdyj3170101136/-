#### http + 装饰模式

```go

type Decorator func(APIHandler) APIHandler

type APIHandler func(http.ResponseWriter, *http.Request, httprouter.Params) (interface{}, error)

func Decorate(f APIHandler, ds ...Decorator) httprouter.Handle {
   decorated := f
   for _, decorate := range ds {
      decorated = decorate(decorated)
   }
   return func(w http.ResponseWriter, req *http.Request, ps httprouter.Params) {
      decorated(w, req, ps)
   }
}
```

- decorator把一些apihandle变成另外的apihandle。，装饰器模式。



decorator有这几种

- 将http请求记录处理时间，以及错误state code

```go
func Log(logf lg.AppLogFunc) Decorator {
   return func(f APIHandler) APIHandler {
      return func(w http.ResponseWriter, req *http.Request, ps httprouter.Params) (interface{}, error) {
         start := time.Now()
         response, err := f(w, req, ps)
         elapsed := time.Since(start)
         status := 200
         if e, ok := err.(Err); ok {
            status = e.Code
         }
         logf(lg.INFO, "%d %s %s (%s) %s",
            status, req.Method, req.URL.RequestURI(), req.RemoteAddr, elapsed)
         return response, err
      }
   }
}
```

- v1则是负责出错的时候，写回错误信息

```go
func V1(f APIHandler) APIHandler {
   return func(w http.ResponseWriter, req *http.Request, ps httprouter.Params) (interface{}, error) {
      data, err := f(w, req, ps)
      if err != nil {
         RespondV1(w, err.(Err).Code, err)
         return nil, nil
      }
      RespondV1(w, 200, data)
      return nil, nil
   }
}
```

具体的api则负责从url的query中解析参数。

获取参数和返回错误。

错误error，包括状态吗status code

```go
type ReqParams struct {
   url.Values
   Body []byte
}

func NewReqParams(req *http.Request) (*ReqParams, error) {
   reqParams, err := url.ParseQuery(req.URL.RawQuery)
   if err != nil {
      return nil, err
   }

   data, err := ioutil.ReadAll(req.Body)
   if err != nil {
      return nil, err
   }

   return &ReqParams{reqParams, data}, nil
}
```

```go
func (s *httpServer) doDeleteTopic(w http.ResponseWriter, req *http.Request, ps httprouter.Params) (interface{}, error) {
   reqParams, err := http_api.NewReqParams(req)
   if err != nil {
      s.ctx.nsqd.logf(LOG_ERROR, "failed to parse request params - %s", err)
      return nil, http_api.Err{400, "INVALID_REQUEST"}
   }

   topicName, err := reqParams.Get("topic")
   if err != nil {
      return nil, http_api.Err{400, "MISSING_ARG_TOPIC"}
   }

   err = s.ctx.nsqd.DeleteExistingTopic(topicName)
   if err != nil {
      return nil, http_api.Err{404, "TOPIC_NOT_FOUND"}
   }

   return nil, nil
}
```

![截屏2020-09-09 下午8.23.01](/Users/jieyang/Library/Application Support/typora-user-images/截屏2020-09-09 下午8.23.01.png)



- 处理method 找不到，notfound的情况，其实就是返回错误吗加text