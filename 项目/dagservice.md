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