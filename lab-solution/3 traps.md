# Lab Traps

1. RISC-V assembly

- `a0-a7`, `a2` holds 13
- The compiler calculates `f(8) + 1` at compiling time and t does not call `f` or `g` explicitly
- `0000000000000628 <printf>`
- `0x38`
- `Hello World`, `i` should be set to `726c6400`, `57616` does not have to be changed
- It will print a random value depending on what `a2` contains during runtime

2. Backtrace

- Be careful with C pointer types
```cpp
void backtrace(void) {
  uint64 fp = r_fp();
  uint64 end = PGROUNDUP(fp);
  while(fp < end) {
    printf("%p\n", *((uint64*)(fp - 8)));
    fp = *((uint64*)(fp - 16));
  }
}
```

3. Alarm

- Modify `Makefile user/usys.pl kernel/syscall.h kernel/syscall.c` to add new system calls `sysalarm()` and `sysreturn()`
- Add new field to `proc` in `kernel/proc.h`
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
  int ticks;                   // How many ticks have passed since last alarm
  int alarm_interval;          
  uint64 handler;              // Alarm handler
  struct trapframe alarm_trapframe;   // Back up trapframe to restore trapframe after alarm handler returns 
};
```

- Modify `allocproc()` in `kernel/proc.c` to initialize newly added fields
```cpp
found:
  p->pid = allocpid();

  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    release(&p->lock);
    return 0;
  }

  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;
  
  p->ticks = 0;
  p->alarm_interval = 0;
  p->handler = 0;

  return p;
}
```

- Modify `usertrap()` in `trap.c` to implement alarm machanism when timer ticks
```cpp
  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2) {
    if(p->alarm_interval) { // alarm_interval == 0 indicates that alarm should not happen
      p->ticks++;
      if(p->ticks == p->alarm_interval) {
        // store trapframe before jumping to alarm handler
        memmove(&p->alarm_trapframe, p->trapframe, sizeof(struct trapframe));
        // should call alarm handler function after switch back
        p->trapframe->epc = p->handler;
      }
    }
    yield();
  }
```

- `sys_sigalarm()` install alarm interval and handler, `sys_sigreturn()` restore the states of the process before it invokes handler
```cpp
uint64 sys_sigalarm(void) {
  int interval;
  uint64 handler;
  struct proc *p = myproc();

  if(argint(0, &interval) < 0) 
    return -1;
  if(argaddr(1, &handler) < 0)
    return -1;
  
  p->alarm_interval = interval;
  p->handler = handler;
  p->ticks = 0;
  return 0;
}

uint64 sys_sigreturn(void) {
  struct proc *p = myproc();
  memmove(p->trapframe, &p->alarm_trapframe, sizeof(struct trapframe));
  p->ticks = 0;
  return 0;
}
``` 