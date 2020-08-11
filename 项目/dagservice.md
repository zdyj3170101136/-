#### dagservice

```
// dagService is an IPFS Merkle DAG service.
// - the root is virtual (like a forest)
// - stores nodes' data in a BlockService
// TODO: should cache Nodes that are in memory, and be
//       able to free some of them when vm pressure is high
type dagService struct {
   Blocks bserv.BlockService
}
```

非叶子结点最多包含 174 个 Link

dagservice主要就是取出block然后把解码成merkle的node。



merkle的node主要有一组link指向其他的node。

以及一个encoded，表示通过protobuf 序列化过后的数据。



link包括这个链接的名字，大小，以及cid。

```
// ProtoNode represents a node in the IPFS Merkle DAG.
// nodes have opaque data and a set of navigable links.
type ProtoNode struct {
   links []*ipld.Link
   data  []byte

   // cache encoded/marshaled value
   encoded []byte

   cached cid.Cid

   // builder specifies cid version and hashing function
   builder cid.Builder
}


// Link represents an IPFS Merkle DAG Link between Nodes.
type Link struct {
	// utf string name. should be unique per object
	Name string // utf8

	// cumulative size of target object
	Size uint64

	// multihash of the target object
	Cid cid.Cid
}
```

#### dagservice

这一层主要就是把这个merklenode通过protobuf序列化成字节流，然后加进datastore里头去。



还有通过一个根cid，得到所有的cid。

其实就是fetchGraph，得到整个graph的图。



主要就是开了32个协程，然后bfs广度搜索。



#### read

通常上面给我们一组cid列表。

让我们取出对应的数据。



这个时候通常还包括预取请求。

当我们对这个cid列表读取i位的时候，如果五步之内的cid没有已经获得。

那么就会提前preload，取得10步以内的所有node。

```
// FetchChild implements the `NavigableNode` interface using node promises
// to preload the following child nodes to `childIndex` leaving them ready
// for subsequent `FetchChild` calls.
func (nn *NavigableIPLDNode) FetchChild(ctx context.Context, childIndex uint) (NavigableNode, error) {
   // This function doesn't check that `childIndex` is valid, that's
   // the `Walker` responsibility.

   // If we drop to <= preloadSize/2 preloading nodes, preload the next 10.
   for i := childIndex; i < childIndex+preloadSize/2 && i < uint(len(nn.childPromises)); i++ {
      // TODO: Check if canceled.
      if nn.childPromises[i] == nil {
         nn.preload(ctx, i)
         break
      }
   }

   child, err := nn.getPromiseValue(ctx, childIndex)

   switch err {
   case nil:
   case context.DeadlineExceeded, context.Canceled:
      if ctx.Err() != nil {
         return nil, ctx.Err()
      }

      // In this case, the context used to *preload* the node (in a previous
      // `FetchChild` call) has been canceled. We need to retry the load with
      // the current context and we might as well preload some extra nodes
      // while we're at it.
      nn.preload(ctx, childIndex)
      child, err = nn.getPromiseValue(ctx, childIndex)
      if err != nil {
         return nil, err
      }
   default:
      return nil, err
   }

   return NewNavigableIPLDNode(child, nn.nodeGetter), nil
}

// Number of nodes to preload every time a child is requested.
// TODO: Give more visibility to this constant, it could be an attribute
// set in the `Walker` context that gets passed in `FetchChild`.
const preloadSize = 10

// Preload at most `preloadSize` child nodes from `beg` through promises
// created using this `ctx`.
func (nn *NavigableIPLDNode) preload(ctx context.Context, beg uint) {
   end := beg + preloadSize
   if end >= uint(len(nn.childCIDs)) {
      end = uint(len(nn.childCIDs))
   }

   copy(nn.childPromises[beg:], GetNodes(ctx, nn.nodeGetter, nn.childCIDs[beg:end]))
}

// Fetch the actual node (this is the blocking part of the mechanism)
// and invalidate the promise. `preload` should always be called first
// for the `childIndex` being fetch.
//
// TODO: Include `preload` into the beginning of this function?
// (And collapse the two calls in `FetchChild`).
func (nn *NavigableIPLDNode) getPromiseValue(ctx context.Context, childIndex uint) (Node, error) {
   value, err := nn.childPromises[childIndex].Get(ctx)
   nn.childPromises[childIndex] = nil
   return value, err
}

// Get the CID of all the links of this `node`.
func getLinkCids(node Node) []cid.Cid {
   links := node.Links()
   out := make([]cid.Cid, 0, len(links))

   for _, l := range links {
      out = append(out, l.Cid)
   }
   return out
}

// GetIPLDNode returns the IPLD `Node` wrapped into this structure.
func (nn *NavigableIPLDNode) GetIPLDNode() Node {
   return nn.node
}

// ChildTotal implements the `NavigableNode` returning the number
// of links (of child nodes) in this node.
func (nn *NavigableIPLDNode) ChildTotal() uint {
   return uint(len(nn.GetIPLDNode().Links()))
}

// ExtractIPLDNode is a helper function that takes a `NavigableNode`
// and returns the IPLD `Node` wrapped inside. Used in the `Visitor`
// function.
// TODO: Check for errors to avoid a panic?
func ExtractIPLDNode(node NavigableNode) Node {
   return node.(*NavigableIPLDNode).GetIPLDNode()
}

// TODO: `Cleanup` is not supported at the moment in the `Walker`.
//
// Called in `Walker.up()` when the node is not part of the path anymore.
//func (nn *NavigableIPLDNode) Cleanup() {
// // TODO: Ideally this would be the place to issue a context `cancel()`
// // but since the DAG reader uses multiple contexts in the same session
// // (through `Read` and `CtxReadFull`) we would need to store an array
// // with the multiple contexts in `NavigableIPLDNode` with its corresponding
// // cancel functions.
//}
```



- fetchGraph主要是用于当我们pin了一个节点的时候。
- 取出这个节点的所有子节点到本地来。（从网络获取）。

#### 当get的时候 dfs先序遍历

- 我们找到根目录的块

- 构建一个dagreader结构体

```go
func (api *UnixfsAPI) Get(ctx context.Context, p path.Path) (files.Node, error) {
   ses := api.core().getSession(ctx)

   nd, err := ses.ResolveNode(ctx, p)
   if err != nil {
      return nil, err
   }

   return unixfile.NewUnixfsFile(ctx, ses.dag, nd)
}
```



```go
&dagReader{
   ctx:       ctxWithCancel,
   cancel:    cancel,
   serv:      serv,
   size:      size,
   rootNode:  n,
   dagWalker: ipld.NewWalker(ctxWithCancel, ipld.NewNavigableIPLDNode(n, serv)),
}, nil
```



- 从dagreader里头寻找块的话。
- 通常是客户端Read（[]byte）传过来，然后我们把数据塞进去
- 我们以块为单位遍历文件，但是对方可能是一个[]byte切片，大小不一定和我们的块相同
- 所以当我们把块取到内存之后，我们用一个buffer把它包装，buffer就是以[]byte作为输出
- 读完了会停止遍历



使用dagreader的时候。

- 首先沿着一条边down到最低点
- 然后nextchild遍历完之后就up
- 在遍历的时候会使用到预取
- 预取的时候检查是否五个存在为0，如果存在则提前预取十个。
      -  有十个块
      -  读到第六个块的时候，会检查6，7，8，9，10这五个块，此时没有预取
      -  而当读到第七个块的时候，则会预取11到20这五个块。
      -  也就是说我们有五个块的缓冲时间。
      -  我们假设read读完7，8，9，10

- 也就是说，我们取文件的时候，尽量一次异步读取十个文件，当然啦，底层的文件系统还是只实现单个get语意。

- ```go
  // If we drop to <= preloadSize/2 preloading nodes, preload the next 10.
  ```

```go
// Iterate the DAG through the DFS pre-order walk algorithm, going down
// as much as possible, then `NextChild` to the other siblings, and then up
// (to go down again). The position is saved throughout iterations (and
// can be previously set in `Seek`) allowing `Iterate` to be called
// repeatedly (after a `Pause`) to continue the iteration.
//
// This function returns the errors received from `down` (generated either
// inside the `Visitor` call or any other errors while fetching the child
// nodes), the rest of the move errors are handled within the function and
// are not returned.
```

```go
// CtxReadFull reads data from the DAG structured file. It always
// attempts a full read of the DAG until the `out` buffer is full.
// It uses the `Walker` structure to iterate the file DAG and read
// every node's data into the `out` buffer.
func (dr *dagReader) CtxReadFull(ctx context.Context, out []byte) (n int, err error) {
   // Set the `dagWalker`'s context to the `ctx` argument, it will be used
   // to fetch the child node promises (see
   // `ipld.NavigableIPLDNode.FetchChild` for details).
   dr.dagWalker.SetContext(ctx)

   // If there was a partially read buffer from the last visited
   // node read it before visiting a new one.
   if dr.currentNodeData != nil {
      // TODO: Move this check inside `readNodeDataBuffer`?
      n = dr.readNodeDataBuffer(out)

      if n == len(out) {
         return n, nil
         // Output buffer full, no need to traverse the DAG.
      }
   }

   // Iterate the DAG calling the passed `Visitor` function on every node
   // to read its data into the `out` buffer, stop if there is an error or
   // if the entire DAG is traversed (`EndOfDag`).
   err = dr.dagWalker.Iterate(func(visitedNode ipld.NavigableNode) error {
      node := ipld.ExtractIPLDNode(visitedNode)

      // Skip internal nodes, they shouldn't have any file data
      // (see the `balanced` package for more details).
      if len(node.Links()) > 0 {
         return nil
      }

      err = dr.saveNodeData(node)
      if err != nil {
         return err
      }
      // Save the leaf node file data in a buffer in case it is only
      // partially read now and future `CtxReadFull` calls reclaim the
      // rest (as each node is visited only once during `Iterate`).
      //
      // TODO: We could check if the entire node's data can fit in the
      // remaining `out` buffer free space to skip this intermediary step.

      n += dr.readNodeDataBuffer(out[n:])

      if n == len(out) {
         // Output buffer full, no need to keep traversing the DAG,
         // signal the `Walker` to pause the iteration.
         dr.dagWalker.Pause()
      }

      return nil
   })

   if err == ipld.EndOfDag {
      return n, io.EOF
      // Reached the end of the (DAG) file, no more data to read.
   } else if err != nil {
      return n, err
      // Pass along any other errors from the `Visitor`.
   }

   return n, nil
}

// Save the UnixFS `node`'s data into the internal `currentNodeData` buffer to
// later move it to the output buffer (`Read`) or seek into it (`Seek`).
func (dr *dagReader) saveNodeData(node ipld.Node) error {
   extractedNodeData, err := unixfs.ReadUnixFSNodeData(node)
   if err != nil {
      return err
   }

   dr.currentNodeData = bytes.NewReader(extractedNodeData)
   return nil
}


// Read the `currentNodeData` buffer into `out`. This function can't have
// any errors as it's always reading from a `bytes.Reader` and asking only
// the available data in it.
func (dr *dagReader) readNodeDataBuffer(out []byte) int {

	n, _ := dr.currentNodeData.Read(out)
	// Ignore the error as the EOF may not be returned in the first
	// `Read` call, explicitly ask for an empty buffer below to check
	// if we've reached the end.

	if dr.currentNodeData.Len() == 0 {
		dr.currentNodeData = nil
		// Signal that the buffer was consumed (for later `Read` calls).
		// This shouldn't return an EOF error as it's just the end of a
		// single node's data, not the entire DAG.
	}

	dr.offset += int64(n)
	// TODO: Should `offset` be incremented here or in the calling function?
	// (Doing it here saves LoC but may be confusing as it's more hidden).

	return n
}
```



每个nodePromise都是一个异步的块。

我们开启一个协程遍历文件块

```go
// GetNodes returns an array of 'FutureNode' promises, with each corresponding
// to the key with the same index as the passed in keys
func GetNodes(ctx context.Context, ds NodeGetter, keys []cid.Cid) []*NodePromise {

   // Early out if no work to do
   if len(keys) == 0 {
      return nil
   }

   promises := make([]*NodePromise, len(keys))
   for i := range keys {
      promises[i] = NewNodePromise(ctx)
   }

   dedupedKeys := dedupeKeys(keys)
   go func() {
      ctx, cancel := context.WithCancel(ctx)
      defer cancel()

      nodechan := ds.GetMany(ctx, dedupedKeys)

      for count := 0; count < len(keys); {
         select {
         case opt, ok := <-nodechan:
            if !ok {
               for _, p := range promises {
                  p.Fail(ErrNotFound)
               }
               return
            }

            if opt.Err != nil {
               for _, p := range promises {
                  p.Fail(opt.Err)
               }
               return
            }

            nd := opt.Node
            c := nd.Cid()
            for i, lnk_c := range keys {
               if c.Equals(lnk_c) {
                  count++
                  promises[i].Send(nd)
               }
            }
         case <-ctx.Done():
            return
         }
      }
   }()
   return promises
}
```



- 当我们读取一个块的child的时候
- 我们建立一个nodePromise数组（这个数组的下标就表示child）

- nodePromise整个数组的大小是通过node的links数目获得的
- nodePromise是异步的。
- 而对nodePromise数组的读取会阻塞在一个chan管道上，直到这个nodePromise得到数据然后关闭管道。
- 当我们读取一个块的时候，我们会开启一个后台线程，异步的预取十个块。

- 我们会把异步获得的块包装成一个nodePromise， 赋值给value。
- get会阻塞在done的chan上（巧妙使用chan）

```go
type NodePromise struct {
   value Node
   err   error
   done  chan struct{}

   ctx context.Context
}
```

```go
// Fulfill this promise.
//
// Once a promise has been fulfilled or failed, calling this function will
// panic.
func (np *NodePromise) Send(nd Node) {
   // if promise has a value, don't fail it
   if np.err != nil || np.value != nil {
      panic("already filled")
   }
   np.value = nd
   close(np.done)
}


// Get the value of this promise.
//
// This function is safe to call concurrently from any number of goroutines.
func (np *NodePromise) Get(ctx context.Context) (Node, error) {
	select {
	case <-np.done:
		return np.value, np.err
	case <-np.ctx.Done():
		return nil, np.ctx.Err()
	case <-ctx.Done():
		return nil, ctx.Err()
	}
}
```