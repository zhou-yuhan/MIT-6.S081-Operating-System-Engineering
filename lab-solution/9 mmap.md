# Lab Mmap

- Modify `Makefile user/user.h user/usys.pl kernel/syscall.h kernel/syscall.c` to add new system calls `mmap() munmap()`
- Define a struct `vma` to keep track of mapped memory areas of a process, add array `vmas` in `struct proc`. Initialize `vmas` in `allocproc()`
```cpp
#define MAXVMA 16
struct vma {
  int valid;
  uint64 addr;
  int length;
  int permission;
  int flags;
  struct file *f;
  int offset;
};

// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
  struct vma vmas[MAXVMA];         // memory mapped areas
};

// proc.c allocproc()
  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  for(int i = 0; i < MAXVMA; ++i)
    p->vmas[i].valid = 0;

  return p;
```

- Write `sys_mmap()` to implement `mmap()`: find an unused region in the process's address space (simply on top of the current `p->sz`), but do not actually allocate physical memory or map virtual memory, just like lazy allocation
```cpp
uint64 sys_mmap(void) {
  uint64 addr;
  int len, prot, flags, fd, offset;
  struct proc* p = myproc();
  struct file *f;
  if(argaddr(0, &addr) < 0 || argint(1, &len) < 0 || argint(2, &prot) < 0 || argint(3, &flags) < 0 || argfd(4, &fd, &f) < 0 || argint(5, &offset) < 0)
    return -1;

  // file is not writable but we try to write it in memory and write back to disk
  if(!f->writable && ((prot & PROT_WRITE) && (flags & MAP_SHARED)))
    return -1;

  struct vma* v = 0;
  for(int i = 0; i < MAXVMA; ++i) {
    if(!p->vmas[i].valid) {
       v = &p->vmas[i];
       v->valid = 1;
       break;
    }
  }
  if(v == 0)
    return -1;

  v->addr = p->sz;  // map memory on top of user's address space
  len = PGROUNDUP(len);
  p->sz += len;
  v->length = len;
  v->permission = prot;
  v->flags = flags;
  v->f = filedup(f);
  v->offset = offset;
  return v->addr;
}
```

- Since we introduced some lazy allocation here, we have to modify `uvmunmap()` to avoid panicing when it finds that a page in address space does
not have a valid PTE. Specifically, comment `if(*pte & PTE_V == 0) ...`


- Modify `usertrap()` in `trap.c` to handle page fault caused by access to our mapped memory. Allocate a physical page and clear it (important), fill it with content from corresponding file
```cpp
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(p->killed)
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sstatus &c registers,
    // so don't enable until done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else if(r_scause() == 13 || r_scause() == 15){ 
    // page fault
    uint64 va = r_stval();
    if(va >= p->sz || va < p->trapframe->sp)
      goto error;

    for(int i = 0; i < MAXVMA; ++i) {
      struct vma* v = &p->vmas[i];
      if(v->valid && va >= v->addr && va < v->addr + v->length) {
        // va falls in this vma, allocate a physical page filles with content from file
        char *mem;
        if((mem = (char*)kalloc()) == 0)
          goto error;
        memset(mem, 0, PGSIZE); // This line is important, mmaptest tests whether newly allocated page is filled with 0 not junk (0x5)
        va = PGROUNDDOWN(va);
        if(mappages(p->pagetable, va, PGSIZE, (uint64)mem, (v->permission << 1) | PTE_U) != 0) {
          // PTE_W == 4 and PROT_WRITE == 2, so as other permission bits
          kfree(mem);
          goto error;
        }
        ilock(v->f->ip);
        readi(v->f->ip, 1, va, v->offset + va - v->addr, PGSIZE);
        // why v->offset + va - v->addr? draw a picture and you will figure out
        iunlock(v->f->ip);
        break;
      }
    }
  } else {
  error:
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  if(p->killed)
    exit(-1);

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2)
    yield();

  usertrapret();
}
```

- Write `sys_munmap()` to implement `munmap()`: write back some content if necessary and unmap some region. In this lab `munmap()` always unmap at 
the start of the mapped area so we don't have to worry about a hole in the mapped area (otherwise we have to divide the area into 2 `vma`)
```cpp
uint64 sys_munmap(void) {
  uint64 addr;
  int len;
  struct proc* p = myproc();

  if(argaddr(0, &addr) < 0 || argint(1, &len) < 0)
    return -1;

  struct vma* v = 0;
  for(int i = 0; i < MAXVMA; ++i) {
    if(p->vmas[i].valid && addr >= p->vmas[i].addr && addr < p->vmas[i].addr + p->vmas[i].length) {
      v = &p->vmas[i];
      break;
    }
  }
  if(v == 0)
    return -1;

  int close = 0;
  int offset = v->offset;
  if(addr == v->addr) {
    // in this lab we assume that we unmap at the start
    if(len >= v->length) {
      len = v->length;
      v->valid = 0;
      close = 1;
    } else {
      v->addr += len;
      v->offset += len;
    }
  }  
  len = PGROUNDUP(len);
  int npages = len / PGSIZE;
  v->length -= len;
  // Conceptly p->sz should be shrunk, but in our implementation this is wrong 
  // but if you uncomment the line below it stil passes mmaptest
  // p->sz -= len;

  if(v->flags & MAP_SHARED) {
    v->f->off = offset;
    filewrite(v->f, addr, len);
  }

  uvmunmap(p->pagetable, PGROUNDDOWN(addr), npages, 0);
  if(close) 
    fileclose(v->f);
  return 0;
}
```


- Modify `exit()` to unmap virtual memory areas when exiting
```cpp
void
exit(int status)
{
  struct proc *p = myproc();

  if(p == initproc)
    panic("init exiting");

  for(int i = 0; i < MAXVMA; ++i) {
    struct vma* v = &p->vmas[i];
    if(v->valid) {
      uvmunmap(p->pagetable, v->addr, v->length / PGSIZE, 0);
      v->valid = 0;
    }
  }
```

- Modify `fork()` to copy `vmas` to child process
```cpp
int
fork(void)
{
  int i, pid;
  struct proc *np;
  struct proc *p = myproc();

  // Allocate process.
  if((np = allocproc()) == 0){
    return -1;
  }

  for(int i = 0; i < MAXVMA; ++i) {
    struct vma *v = &p->vmas[i];
    struct vma *nv = &np->vmas[i];
    if(v->valid) {
      memmove(nv, v, sizeof(struct vma));
      filedup(nv->f);
    }
  }
```