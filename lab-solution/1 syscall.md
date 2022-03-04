# Lab Syscall

1. trace

- add attribute `mask` in `kernel/proc.h` 
```cpp
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
  int mask;                   // Mask for sys_trace
};
``` 

- add `sys_trace()` in `kernel/sysproc.c`
```cpp
uint64 sys_trace(void) {
  int mask;
  if(argint(0, &mask) < 0)
    return -1;
  myproc()->mask = mask;
  return 0;
}
```

- add array `sys_names` in `kernel/syscall.c` to index names of sys calls
```c
static char* sys_names[] = {
[SYS_fork]    "fork",
[SYS_exit]    "exit",
[SYS_wait]    "wait",
[SYS_pipe]    "pipe",
[SYS_read]    "read",
[SYS_kill]    "kill",
[SYS_exec]    "exec",
[SYS_fstat]   "fstat",
[SYS_chdir]   "chdir",
[SYS_dup]     "dup",
[SYS_getpid]  "getpid",
[SYS_sbrk]    "sbrk",
[SYS_sleep]   "sleep",
[SYS_uptime]  "uptime",
[SYS_open]    "open",
[SYS_write]   "write",
[SYS_mknod]   "mknod",
[SYS_unlink]  "unlink",
[SYS_link]    "link",
[SYS_mkdir]   "mkdir",
[SYS_close]   "close",
[SYS_trace]   "trace",
};
```

- modify `syscall()` in `kernel/syscall.c` to print a message when a sys call returns 
```cpp
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
    // specially for sys call trace 
    if((1u << num) & p->mask) {
      printf("%d: syscall %s -> %d\n", p->pid, sys_names[num], p->trapframe->a0);
    }
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

- modify `fork()` in `kernel/proc.c` to copy `mask` attribute from parent to child process
- add sys call number for `sys_trace`

2. sysinfo

- add `freemem()` in `kernel/kalloc.c` to count free memory that cab be used by kernel
```cpp
uint64 freemem(void) {
  struct run* r;
  uint64 freepage = 0;
  acquire(&kmem.lock);
  for(r = kmem.freelist; r; r = r->next) 
    freepage++;
  release(&kmem.lock);
  return freepage * PGSIZE;
}
```

- add `unused_proc()` in `kernel/proc.c` to count unused processes
```cpp
int unused_proc(void) {
  int num = 0;
  struct proc* p;
  for(p = proc; p < &proc[NPROC]; p++) {
    if(p->state != UNUSED)
      num++;
  }
  return num;
}
```

- add `sys_sysinfo()` in `kernel/sysproc.c` to implement syscall `sysinfo()`
```cpp
uint64 sys_sysinfo(void) {
  uint64 addr;
  if(argaddr(0, &addr) < 0) 
    return -1;
  struct sysinfo info;
  struct proc* p = myproc();
  info.freemem = freemem();
  info.nproc = unused_proc();
  if(copyout(p->pagetable, addr, (char*)&info, sizeof(info)) < 0)
    return -1;
  return 0;
}
```