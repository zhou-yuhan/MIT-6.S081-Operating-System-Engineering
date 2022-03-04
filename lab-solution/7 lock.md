# Lab Lock

1. Memory allocator

- Create `kmems` as an array for each cpu and initialize each lock in `kinit()`
```cpp
struct {
  struct spinlock lock;
  struct run *freelist;
} kmems[NCPU];

void
kinit()
{
  for(int i = 0; i < NCPU; ++i) 
    initlock(&kmems[i].lock, "kmem");
  freerange(end, (void*)PHYSTOP);
}
```

- Modify `kfree()` to add freed page into the current cpu's own `freelist`, thus cpus can steal memory from each
other and never return. Stealing keeps balance.
```cpp
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  push_off();
  int cpu = cpuid();

  acquire(&kmems[cpu].lock);
  r->next = kmems[cpu].freelist;
  kmems[cpu].freelist = r;
  release(&kmems[cpu].lock);

  pop_off();
}
```

- Modify `kalloc()`, steal a page from another cpu if its own memory runs out
```cpp
void *
kalloc(void)
{
  struct run *r;

  push_off();
  int cpu = cpuid();

  acquire(&kmems[cpu].lock);
  r = kmems[cpu].freelist;
  if(r) {
    kmems[cpu].freelist = r->next;
  } else {
    // steal a page from other cpus
    for(int i = 0; i < NCPU; ++i) {
      if(i != cpu) {
        acquire(&kmems[i].lock);
        if(kmems[i].freelist) {
          r = kmems[i].freelist;
          kmems[i].freelist = kmems[i].freelist->next;
          release(&kmems[i].lock);
          break;
        }
        release(&kmems[i].lock);
      }
    }
  }
  release(&kmems[cpu].lock);

  pop_off();

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

In this case, the very first cpu which runs `freerange()` owns all free memory and others steal from it in need.

2. Buffer cache

- Overview

`bcache` contains a `hashtable`, each of which is a double linked list containing keys hashed to this slot (i.e. using open hashing method). `bget` looks up in `hashtable[hash(b->blockno)]` to find a buffer, if found (i.e. cached), return it. Else it tries to steal a least recent used buffer from another hashtable slot.

Each double linked list in a hashtable slot has its own lock, thus blockes hashed into different slots acquire different locks. When a new buffer is added to the list, we add it to the front so that if we go through `b->prev` we get LRU buffer, and other slots always steal it if it's idle.

Since the implementation of the linked list does not contains a guard head, it's a painful to delete and add items. Here illustrates the structure of this list:

```cpp
                      -->               --> 
*hashtable[index].prev   hashtable[index]  *hashtable[index].next
                      <--                <--
        LRU buffers            tail            head & MRU buffers
```

Remember this first element in the list is `*hashtable[index].next`

- Implementation
```cpp
// Buffer cache.
//
// The buffer cache is a linked list of buf structures holding
// cached copies of disk block contents.  Caching disk blocks
// in memory reduces the number of disk reads and also provides
// a synchronization point for disk blocks used by multiple processes.
//
// Interface:
// * To get a buffer for a particular disk block, call bread.
// * After changing buffer data, call bwrite to write it to disk.
// * When done with the buffer, call brelse.
// * Do not use the buffer after calling brelse.
// * Only one process at a time can use a buffer,
//     so do not keep them longer than necessary.


#include "types.h"
#include "param.h"
#include "spinlock.h"
#include "sleeplock.h"
#include "riscv.h"
#include "defs.h"
#include "fs.h"
#include "buf.h"

#define NBUCKET 13

struct {
  struct spinlock lock[NBUCKET];
  struct buf buf[NBUF];
  struct buf hashtable[NBUCKET];
} bcache;

int hash(int x) {
  return x % NBUCKET;
}

void
binit(void)
{
  struct buf *b;

  for(int i = 0; i < NBUCKET; ++i) {
    initlock(&bcache.lock[i], "bcache.lock[i]");
    b = &bcache.hashtable[i];
    b->next = b;
    b->prev = b;
  }
  // Initially, hashtable[0] holds all buffer blocks 
  for(b = bcache.buf; b < bcache.buf + NBUF; b++) {
    b->next = bcache.hashtable[0].next;
    b->prev = &bcache.hashtable[0];
    bcache.hashtable[0].next->prev = b;
    bcache.hashtable[0].next = b;
    initsleeplock(&b->lock, "buffer");
  }
}

// Look through buffer cache for block on device dev.
// If not found, steal a buffer from another slot
// In either case, return locked buffer.
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;

  int idx = hash(blockno);

  acquire(&bcache.lock[idx]);

  // Is the block already cached?
  for(b = bcache.hashtable[idx].next; b != &bcache.hashtable[idx]; b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      release(&bcache.lock[idx]);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // Not cached.
  for(int i = hash(blockno + 1); i != idx; i = (i + 1) % NBUCKET){
    acquire(&bcache.lock[i]);
  // Recycle the least recently used (LRU) unused buffer.
    for(b = bcache.hashtable[i].prev; b != &bcache.hashtable[i]; b = b->prev)
    if(b->refcnt == 0) {
      // find a block to steal, steal it to list bcache.hashtable[idx]
      b->dev = dev;
      b->blockno = blockno;
      b->valid = 0;
      b->refcnt = 1;
      b->next->prev = b->prev;
      b->prev->next = b->next;
      // release the lock of the stolen hashtable as soon as possible
      release(&bcache.lock[i]);
      b->next = bcache.hashtable[idx].next;
      b->prev = &bcache.hashtable[idx];
      bcache.hashtable[idx].next->prev = b;
      bcache.hashtable[idx].next = b;

      release(&bcache.lock[idx]);
      acquiresleep(&b->lock);
      return b;
    }
    release(&bcache.lock[i]);
  }
  panic("bget: no buffers");
}

// Return a locked buf with the contents of the indicated block.
struct buf*
bread(uint dev, uint blockno)
{
  struct buf *b;

  b = bget(dev, blockno);
  if(!b->valid) {
    virtio_disk_rw(b, 0);
    b->valid = 1;
  }
  return b;
}

// Write b's contents to disk.  Must be locked.
void
bwrite(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("bwrite");
  virtio_disk_rw(b, 1);
}

// Release a locked buffer.
// Move to the head of the most-recently-used list.
void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  int idx = hash(b->blockno);

  acquire(&bcache.lock[idx]);
  b->refcnt--;
  if (b->refcnt == 0) {
    // no one is waiting for it.
    b->next->prev = b->prev;
    b->prev->next = b->next;
    b->next = bcache.hashtable[idx].next;
    b->prev = &bcache.hashtable[idx];
    bcache.hashtable[idx].next->prev = b;
    bcache.hashtable[idx].next = b;
  }
  
  release(&bcache.lock[idx]);
}

void
bpin(struct buf *b) {
  int idx = hash(b->blockno);
  acquire(&bcache.lock[idx]);
  b->refcnt++;
  release(&bcache.lock[idx]);
}

void
bunpin(struct buf *b) {
  int idx = hash(b->blockno);
  acquire(&bcache.lock[idx]);
  b->refcnt--;
  release(&bcache.lock[idx]);
}
```
