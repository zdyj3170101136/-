```go
// atomicError is a type-safe atomic value for errors.
// We use a struct{ error } to ensure consistent use of a concrete type.
type atomicError struct{ v atomic.Value }

func (a *atomicError) Store(err error) {
   a.v.Store(struct{ error }{err})
}
func (a *atomicError) Load() error {
   err, _ := a.v.Load().(struct{ error })
   return err.error
}

// ErrClosedPipe is the error used for read or write operations on a closed pipe.
var ErrClosedPipe = errors.New("io: read/write on closed pipe")

// A pipe is the shared pipe structure underlying PipeReader and PipeWriter.
type pipe struct {
   wrMu sync.Mutex // Serializes Write operations
   wrCh chan []byte
   rdCh chan int

   once sync.Once // Protects closing done
   done chan struct{}
   rerr atomicError
   werr atomicError
}

func (p *pipe) Read(b []byte) (n int, err error) {
   select {
   case <-p.done:
      return 0, p.readCloseError()
   default:
   }

   select {
   case bw := <-p.wrCh: // 注意bw表示byte write， nr表示numberread
      nr := copy(b, bw)
      p.rdCh <- nr
      return nr, nil
   case <-p.done:
      return 0, p.readCloseError()
   }
}

func (p *pipe) readCloseError() error {
   rerr := p.rerr.Load()
   if werr := p.werr.Load(); rerr == nil && werr != nil {
      return werr
   }
   return ErrClosedPipe
}

func (p *pipe) CloseRead(err error) error {
   if err == nil {
      err = ErrClosedPipe
   }
   p.rerr.Store(err)
   p.once.Do(func() { close(p.done) })
   return nil
}

// nw 表示byte write
func (p *pipe) Write(b []byte) (n int, err error) {
   select {
   case <-p.done:
      return 0, p.writeCloseError()
   default:
      p.wrMu.Lock()
      defer p.wrMu.Unlock()
   }

   for once := true; once || len(b) > 0; once = false {
      select {
      case p.wrCh <- b:
         nw := <-p.rdCh
         b = b[nw:]
         n += nw
      case <-p.done:
         return n, p.writeCloseError()
      }
   }
   return n, nil
}

func (p *pipe) writeCloseError() error {
   werr := p.werr.Load()
   if rerr := p.rerr.Load(); werr == nil && rerr != nil {
      return rerr
   }
   return ErrClosedPipe
}

func (p *pipe) CloseWrite(err error) error {
   if err == nil {
      err = EOF
   }
   p.werr.Store(err)
   p.once.Do(func() { close(p.done) })
   return nil
}

// A PipeReader is the read half of a pipe.
type PipeReader struct {
   p *pipe
}

// Read implements the standard Read interface:
// it reads data from the pipe, blocking until a writer
// arrives or the write end is closed.
// If the write end is closed with an error, that error is
// returned as err; otherwise err is EOF.
func (r *PipeReader) Read(data []byte) (n int, err error) {
   return r.p.Read(data)
}

// Close closes the reader; subsequent writes to the
// write half of the pipe will return the error ErrClosedPipe.
func (r *PipeReader) Close() error {
   return r.CloseWithError(nil) // 注意往往这种调用是建立在同级别。而不是建立在更底层。
}

// CloseWithError closes the reader; subsequent writes
// to the write half of the pipe will return the error err.
func (r *PipeReader) CloseWithError(err error) error {
   return r.p.CloseRead(err)
}

// A PipeWriter is the write half of a pipe.
type PipeWriter struct {
   p *pipe
}

// Write implements the standard Write interface:
// it writes data to the pipe, blocking until one or more readers
// have consumed all the data or the read end is closed.
// If the read end is closed with an error, that err is
// returned as err; otherwise err is ErrClosedPipe.
func (w *PipeWriter) Write(data []byte) (n int, err error) {
   return w.p.Write(data)
}

// Close closes the writer; subsequent reads from the
// read half of the pipe will return no bytes and EOF.
func (w *PipeWriter) Close() error {
   return w.CloseWithError(nil)
}

// CloseWithError closes the writer; subsequent reads from the
// read half of the pipe will return no bytes and the error err,
// or EOF if err is nil.
//
// CloseWithError always returns nil.
func (w *PipeWriter) CloseWithError(err error) error {
   return w.p.CloseWrite(err)
}

// Pipe creates a synchronous in-memory pipe.
// It can be used to connect code expecting an io.Reader
// with code expecting an io.Writer.
//
// Reads and Writes on the pipe are matched one to one
// except when multiple Reads are needed to consume a single Write.
// That is, each Write to the PipeWriter blocks until it has satisfied
// one or more Reads from the PipeReader that fully consume
// the written data.
// The data is copied directly from the Write to the corresponding
// Read (or Reads); there is no internal buffering.
//
// It is safe to call Read and Write in parallel with each other or with Close.
// Parallel calls to Read and parallel calls to Write are also safe:
// the individual calls will be gated sequentially.
func Pipe() (*PipeReader, *PipeWriter) {
   p := &pipe{
      wrCh: make(chan []byte),
      rdCh: make(chan int),
      done: make(chan struct{}),
   }
   return &PipeReader{p}, &PipeWriter{p}
}
```



// pipeReader 和pipeWriter共享pipe这个结构体。

// 但是这两个结构体实现了不同的方法。



// pipe实现三种方法。

// closeread， closewrite

// read， write



- 可以是多写者多读者的操作。
- 通过mutex锁串行化写者，通过无缓冲的wrch和rdch，使得一个写者会block直到另外一边完全消化了数据。

#### read

通过done这个chan来判断是否是否关闭。

如果关闭了，readCloseErr（）， 表示已经关闭了，返回读取的错误。



- 从wrch中接受【】byte切片
- 读取后从rdch中返回数量



#### write

通过done这个chan判断是否关闭。

如果关闭了，writeCloseErr（），写入关闭的错误。

- 不然默认加锁，串行化写操作。



- 考虑到传入的[]byte可能本身就为空
- once 表示第一次绝对执行，之后只要len（b）长度大于0，也会执行。



#### closeread 

closeread和closewrite

- 默认分别为errclosedpipe和eof



- 分别往werr和rderr这两个atomic变量中存储值
- 存储成功，通过once只关闭一次done chan。

#### writeclose和readcloseerr



- writecloseerr会仅当是由于读端关闭引起的错误时才返回错误，不然返回errclosedPipe
- readclosederr也是，返回pipe



- 作用：当读端读取由于写端关闭的错误的时候，会返回写端关闭的错误，至少也是eof
- 对于写端同理
- 而读端读取不会返回自己所定义的错误，而是io.ErrclosedPipe
- 毕竟我们关闭读端时候的错误应该是写端看到的，而不应该是读端。



#### atomicError

通过一个struct{ error }在存放的类型上达成一致。



- 注意，对于形如一行的错误比如Do(func() { {} })
- 其中func后要空一行，而结构体首尾也需要空一行。