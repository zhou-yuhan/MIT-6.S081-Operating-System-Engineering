# Lab File System

1. Large file

- Modify macros in `fs.h`. Reasonably, I should define some new constants, but it requires more modifications somewhere and I'm lazy. So current macros work and I keep them.

```cpp
/*
Idealy you should write something like below
#define NDIRSLOT 12
#define NDIRECT 11
...
*/

#define NDIRECT 12
#define NINDIRECT (BSIZE / sizeof(uint))
#define MAXFILE (NDIRECT - 1 + NINDIRECT + NINDIRECT * NINDIRECT)
```

- Modify `bmap()` in `fs.c`
```cpp
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;

  if(bn < NDIRECT - 1){
    if((addr = ip->addrs[bn]) == 0)
      ip->addrs[bn] = addr = balloc(ip->dev);
    return addr;
  }
  bn -= NDIRECT - 1;

  if(bn < NINDIRECT){
    // Load single-indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT - 1]) == 0)
      ip->addrs[NDIRECT - 1] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }
  bn -= NINDIRECT;

  if(bn < NINDIRECT * NINDIRECT) {
    // Load double-indirect block, allocating if necessary.
    uint level1 = bn / NINDIRECT;
    uint level2 = bn % NINDIRECT;
    if((addr = ip->addrs[NDIRECT]) == 0)
      ip->addrs[NDIRECT] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[level1]) == 0) {
      a[level1] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[level2]) == 0) {
      a[level2] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }

  panic("bmap: out of range");
}
```

- Modify `itrunc` in `fs.c`

```cpp
void
itrunc(struct inode *ip)
{
  int i, j;
  struct buf *bp, *bbp;
  uint *a, *b;

  for(i = 0; i < NDIRECT - 1; i++){
    if(ip->addrs[i]){
      bfree(ip->dev, ip->addrs[i]);
      ip->addrs[i] = 0;
    }
  }

  if(ip->addrs[NDIRECT - 1]){
    bp = bread(ip->dev, ip->addrs[NDIRECT - 1]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++){
      if(a[j])
        bfree(ip->dev, a[j]);
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT - 1]);
    ip->addrs[NDIRECT - 1] = 0;
  }

  if(ip->addrs[NDIRECT]) {
    bp = bread(ip->dev, ip->addrs[NDIRECT]);
    a = (uint*)bp->data;
    for(i = 0; i < NINDIRECT; i++) {
      if(a[i]) {
        bbp = bread(ip->dev, a[i]);
        b = (uint*)bbp->data;
        for(j = 0; j < NINDIRECT; j++) {
          if(b[j])
            bfree(ip->dev, b[j]);
        }
        brelse(bbp);
        bfree(ip->dev, a[i]);
      }
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT]);
    ip->addrs[NDIRECT] = 0;
  }

  ip->size = 0;
  iupdate(ip);
}
```