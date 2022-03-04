# Lab File System

1. Large file

- Modify macros in `fs.h`, and also modify `file.h`
```cpp
#define NDIRECT 11
#define NINDIRECT (BSIZE / sizeof(uint))
#define MAXFILE (NDIRECT + NINDIRECT + NINDIRECT * NINDIRECT)

// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+2];   // Data block addresses
};

// in-memory copy of an inode
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count
  struct sleeplock lock; // protects everything below here
  int valid;          // inode has been read from disk?

  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+2];
};
```

- Modify `bmap()` in `fs.c`
```cpp
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;

  if(bn < NDIRECT){
    if((addr = ip->addrs[bn]) == 0)
      ip->addrs[bn] = addr = balloc(ip->dev);
    return addr;
  }
  bn -= NDIRECT;

  if(bn < NINDIRECT){
    // Load single-indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT]) == 0)
      ip->addrs[NDIRECT] = addr = balloc(ip->dev);
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
    if((addr = ip->addrs[NDIRECT + 1]) == 0)
      ip->addrs[NDIRECT + 1] = addr = balloc(ip->dev);
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

  for(i = 0; i < NDIRECT; i++){
    if(ip->addrs[i]){
      bfree(ip->dev, ip->addrs[i]);
      ip->addrs[i] = 0;
    }
  }

  if(ip->addrs[NDIRECT]){
    bp = bread(ip->dev, ip->addrs[NDIRECT]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++){
      if(a[j])
        bfree(ip->dev, a[j]);
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT]);
    ip->addrs[NDIRECT] = 0;
  }

  if(ip->addrs[NDIRECT + 1]) {
    bp = bread(ip->dev, ip->addrs[NDIRECT + 1]);
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
    bfree(ip->dev, ip->addrs[NDIRECT + 1]);
    ip->addrs[NDIRECT + 1] = 0;
  }

  ip->size = 0;
  iupdate(ip);
}
```

2. Symbolic links

- Modify `kernel/stat.h kernel/fcntl.h user/usys.pl user/user.h Makefile` as hints tell you
- Add a system call `syslink()`, add its system call number
- Implement system call `sys_syslink()`, store path name `target` in inode data block proceeded by its size
```cpp
uint64 sys_symlink(void) {
  char target[MAXPATH], path[MAXPATH];
  struct inode *ip;

  if(argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0)
    return -1;
  
  begin_op();
  
  if((ip = create(path, T_SYMLINK, 0, 0)) == 0) {
    end_op();
    return -1;
  }

  int len = strlen(target);
  if(len > MAXPATH) 
    panic("sys_symlink: too long pathname\n");
  // write size of sokt link path first, convenient for readi() to read
  if(writei(ip, 0, (uint64)&len, 0, sizeof(int)) != sizeof(int)) {
    end_op();
    return -1;
  }
  if(writei(ip, 0, (uint64)target, sizeof(int), len) != len) {
    end_op();
    return -1;
  }
  
  iupdate(ip);
  iunlockput(ip);

  end_op();
  return 0;
}
``` 

- Modify `sys_open` to look up actual path in soft link inode if `O_NOFOLLOW` is not set. Look up recursively but control the depth to be no more than 10
```cpp
uint64
sys_open(void)
{
  char path[MAXPATH];
  int fd, omode;
  struct file *f;
  struct inode *ip;
  int n;

  if((n = argstr(0, path, MAXPATH)) < 0 || argint(1, &omode) < 0)
    return -1;

  begin_op();

  if(omode & O_CREATE){
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){
      end_op();
      return -1;
    }
  } else {
    if((ip = namei(path)) == 0){
      end_op();
      return -1;
    }
    ilock(ip);
    if(ip->type == T_DIR && omode != O_RDONLY){
      iunlockput(ip);
      end_op();
      return -1;
    }
  }

  if(ip->type == T_DEVICE && (ip->major < 0 || ip->major >= NDEV)){
    iunlockput(ip);
    end_op();
    return -1;
  }
  
  if(!(omode & O_NOFOLLOW)) {
    // Don't treat it as a normal file, look up actual path if T_SYMLINK is set
    char path[MAXPATH];
    int len = 0, depth = 0;
    while(ip->type == T_SYMLINK && depth < 10) {
      // recursively look up softlinks
      if(readi(ip, 0, (uint64)&len, 0, sizeof(int)) != sizeof(int)) {
        iunlockput(ip);
        end_op();
        return -1;
      }
      
      if(len > MAXPATH)
        panic("sys_open: too long pathname\n");
      
      if(readi(ip, 0, (uint64)path, sizeof(int), len) != len) {
        iunlockput(ip);
        end_op();
        return -1;
      }
      iunlockput(ip);
      // new soft link, deepen recursion depth
      if((ip = namei(path)) == 0) {
        end_op();
        return -1;
      }
      ilock(ip);
      depth++;

      if(ip->type == T_DIR && omode != O_RDONLY) {
        iunlockput(ip);
        end_op();
        return -1;
      }
      if(ip->type == T_DEVICE && (ip->major < 0 || ip->major >= NDEV)){
        iunlockput(ip);
        end_op();
        return -1;
      }

      if(depth >= 10) {
        iunlockput(ip);
        end_op();
        return -1;
      }
    }
  }

  if((f = filealloc()) == 0 || (fd = fdalloc(f)) < 0){
    if(f)
      fileclose(f);
    iunlockput(ip);
    end_op();
    return -1;
  }

  if(ip->type == T_DEVICE){
    f->type = FD_DEVICE;
    f->major = ip->major;
  } else {
    f->type = FD_INODE;
    f->off = 0;
  }
  f->ip = ip;
  f->readable = !(omode & O_WRONLY);
  f->writable = (omode & O_WRONLY) || (omode & O_RDWR);

  if((omode & O_TRUNC) && ip->type == T_FILE){
    itrunc(ip);
  }

  iunlock(ip);
  end_op();

  return fd;
}
```