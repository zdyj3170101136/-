#### snapshot

```go
// Get latest sequence number.
func (db *DB) getSeq() uint64 {
	return atomic.LoadUint64(&db.seq)
}

type snapshotElement struct {
   seq uint64
   ref int
   e   *list.Element
}

type DB struct {
  snapsMu sync.Mutex
  snapList *list
  
  seq uint64
}
// Acquires a snapshot, based on latest sequence.
func (db *DB) acquireSnapshot() *snapshotElement {
   db.snapsMu.Lock()
   defer db.snapsMu.Unlock()

   seq := db.getSeq()

   if e := db.snapsList.Back(); e != nil {
      se := e.Value.(*snapshotElement)
      if se.seq == seq {
         se.ref++
         return se
      } else if seq < se.seq {
         panic("leveldb: sequence number is not increasing")
      }
   }
   se := &snapshotElement{seq: seq, ref: 1}
   se.e = db.snapsList.PushBack(se)
   return se
}

// Releases given snapshot element.
func (db *DB) releaseSnapshot(se *snapshotElement) {
   db.snapsMu.Lock()
   defer db.snapsMu.Unlock()

   se.ref--
   if se.ref == 0 {
      db.snapsList.Remove(se.e)
      se.e = nil
   } else if se.ref < 0 {
      panic("leveldb: Snapshot: negative element reference")
   }
}

// Gets minimum sequence that not being snapshotted.
func (db *DB) minSeq() uint64 {
   db.snapsMu.Lock()
   defer db.snapsMu.Unlock()

   if e := db.snapsList.Front(); e != nil {
      return e.Value.(*snapshotElement).seq
   }

   return db.getSeq()
}
```