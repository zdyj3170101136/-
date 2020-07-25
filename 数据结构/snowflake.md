```
// Package snowflake provides a very simple Twitter snowflake generator and parser.
package snowflake

import (
   "errors"
   "strconv"
   "sync"
   "time"
)

var (
   // NodeBits holds the number of bits to use for Node
   // Remember, you have a total 22 bits to share between Node/Step
   NodeBits uint8 = 10  // 10位是节点id

   // StepBits holds the number of bits to use for Step
   // Remember, you have a total 22 bits to share between Node/Step
   StepBits uint8 = 12   // 12位是每一秒内的id
   
   nodeMax   int64 = -1 ^ (-1 << NodeBits) // -1 左移十位，说明低10个低bit的0，最后取反，10个低bit的1
   nodeMask        = nodeMax << StepBits   // nodemask表示从13位到23位都是连续的1
   stepMask  int64 = -1 ^ (-1 << StepBits) // 表示低位有12个连续的1
   timeShift       = NodeBits + StepBits   // 左移动10位加上12位为time
   nodeShift       = StepBits              // 左移动12位为nodeshift
)

// A Node struct holds the basic information needed for a snowflake generator
// node
type Node struct {
   mu    sync.Mutex
   time  int64
   node  int64
   step  int64
}

// An ID is a custom type used for a snowflake ID.  This is used so we can
// attach methods onto the ID.
type ID int64

// NewNode returns a new snowflake node that can be used to generate snowflake
// IDs
func NewNode(node int64) (*Node, error) {
   n := Node{}
   n.node = node

   if n.node < 0 || n.node > nodeMax {
      return nil, errors.New("Node number must be between 0 and " + strconv.FormatInt(nodeMax, 10))
   }

   return &n, nil
}

// Generate creates and returns a unique snowflake ID
// To help guarantee uniqueness
// - Make sure your system is keeping accurate system time
// - Make sure you never have multiple nodes running with the same node ID
func (n *Node) Generate() (ID, error) {

   n.mu.Lock()
   defer n.mu.Unlock()

   now := time.Now().UnixNano() / 1e6

   if now < n.time {
      return ID(0), errors.New("")
   }

   if now == n.time { // 如果和之前的time一样，则增加step
      n.step = (n.step + 1) & stepMask

      if n.step == 0 { // 如果达到了1s内生成的最多id数
         for now <= n.time {   //
            now = time.Now().UnixNano() / 1e6
         }
      }
   } else {   // 不然重制step
      n.step = 0
   }

   n.time = now

   r := ID((now)<<timeShift |
      (n.node << nodeShift) |
      (n.step),
   )

   return r, nil
}

// Int64 returns an int64 of the snowflake ID
func (f ID) Int64() int64 {
   return int64(f)
}
```