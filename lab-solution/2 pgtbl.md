# Lab Pgtbl

1. Print a page table

```cpp
void vmprint_helper(pagetable_t pagetable, int level) {
  if(level > 3)
    return;
  for(int i = 0; i < 512; ++i) {
    if(pagetable[i] & PTE_V) {
      printf("..");
      for(int j = 1; j < level; ++j) 
        printf(" ..");
      printf("%d: pte %p pa %p\n", i, pagetable[i], PTE2PA(pagetable[i]));
      vmprint_helper((pagetable_t)PTE2PA(pagetable[i]), level + 1);
    }
  }
  return;
}

void vmprint(pagetable_t pagetable) {
  printf("page table %p\n", pagetable);
  vmprint_helper(pagetable, 1);
  return;
}
```

2. A kernal page table per process

follow the hints step by step

- add field `kpagetable` to `struct proc` in `kernel/proc.h`

- initialize kernal pagetable for a process, similar to `kvminit()` in `kernel/vm.c`, `kpagetable` does not map kernel stack so far

```cpp
void proc_kvmmap(pagetable_t pagetable, uint64 va, uint64 pa, uint64 sz, int perm) {
  if(mappages(pagetable, va, sz, pa, perm) != 0)
    panic("proc_kvmmap");
}

pagetable_t proc_kvmcreat() {
  pagetable_t kpagetable = uvmcreate();
  if(kpagetable == 0) 
    return 0;

  // uart registers
  proc_kvmmap(kpagetable, UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // virtio mmio disk interface
  proc_kvmmap(kpagetable, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // CLINT
  proc_kvmmap(kpagetable, CLINT, CLINT, 0x10000, PTE_R | PTE_W);

  // PLIC
  proc_kvmmap(kpagetable, PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  // map kernel text executable and read-only.
  proc_kvmmap(kpagetable, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);

  // map kernel data and the physical RAM we'll make use of.
  proc_kvmmap(kpagetable, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);

  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel.
  proc_kvmmap(kpagetable, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);


  return kpagetable; 
}
```

- `procinit()` in `kernel/proc.c` creat a kernel stack on physical memory and map it to the only kernel pagetable for each process. Now we want to map the kernel stack on physical memory to the process's own kernel page table, thus we move the codes in `procinit()` to `allocproc` in `kernel/proc.c`. Therefore each process has access to its kernel stack via `kpagetable`

```cpp
// Look in the process table for an UNUSED proc.
// If found, initialize state required to run in the kernel,
// and return with p->lock held.
// If there are no free procs, or a memory allocation fails, return 0.
static struct proc*
allocproc(void)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == UNUSED) {
      goto found;
    } else {
      release(&p->lock);
    }
  }
  return 0;

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

  // Add kernal page table
  p->kpagetable = proc_kvmcreat();
  if(p->kpagetable == 0) {
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // map kernal stack of this process to its kernal page table
  char* pa = kalloc();
  if(pa == 0)
    panic("allocproc");
  uint64 va = KSTACK((int) (p - proc));
  proc_kvmmap(p->kpagetable, va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
  p->kstack = va;

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  return p;
}
```

- modify `schduler()` in `kernel/proc.c`, use `p->kpagetable` when running a process `p`, if no process is to run, use the global `kernel_pagetable`

```cpp
// Per-CPU process scheduler.
// Each CPU calls scheduler() after setting itself up.
// Scheduler never returns.  It loops, doing:
//  - choose a process to run.
//  - swtch to start running that process.
//  - eventually that process transfers control
//    via swtch back to the scheduler.
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  
  c->proc = 0;
  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    intr_on();
    
    int found = 0;
    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;

        // set satp register to be the kernal page table of this process
        w_satp(MAKE_SATP(p->kpagetable));
        sfence_vma();

        swtch(&c->context, &p->context);


        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;
        found = 1;
      }
      release(&p->lock);
    }
#if !defined (LAB_FS)
    if(found == 0) {
      intr_on();
      // no process to run, use global kernal page table
      kvminithart();
      asm volatile("wfi");
    }
#else
    ;
#endif
  }
}
```

- free kernel pagetable for a process, modify `freeproc()` in `kernel/proc.c` to free kernel stack in physical memory and free the kernel pagetable. DO NOT free physical memory when freeing process's kernel pagetable

```cpp
// free a proc structure and the data hanging from it,
// including user pages.
// p->lock must be held.
static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  // free kernal stack in physical memory
  if(p->kstack) {
    pte_t* pte = walk(p->kpagetable, p->kstack, 0);
    kfree((void*) PTE2PA(*pte)); 
  }
  if(p->kpagetable)
    proc_freekpagetable(p->kpagetable, 1);
  p->pagetable = 0;
  p->kpagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;
}

// Create a user page table for a given process,
// with no user memory, but with trampoline pages.
pagetable_t
proc_pagetable(struct proc *p)
{
  pagetable_t pagetable;

  // An empty page table.
  pagetable = uvmcreate();
  if(pagetable == 0)
    return 0;

  // map the trampoline code (for system call return)
  // at the highest user virtual address.
  // only the supervisor uses it, on the way
  // to/from user space, so not PTE_U.
  if(mappages(pagetable, TRAMPOLINE, PGSIZE,
              (uint64)trampoline, PTE_R | PTE_X) < 0){
    uvmfree(pagetable, 0);
    return 0;
  }

  // map the trapframe just below TRAMPOLINE, for trampoline.S.
  if(mappages(pagetable, TRAPFRAME, PGSIZE,
              (uint64)(p->trapframe), PTE_R | PTE_W) < 0){
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }

  return pagetable;
}
```

- modify `kvmpa()`, change `kernel_pagetable` to `myproc()->kpagetable`

3. simplify `copyin/copyinstr`

- add `u2kvmcopy()` in `kernel/vm.c` to copy user address space to kernel address space via add PTEs in kernel page table
```cpp
void u2kvmcopy(pagetable_t upagetable, pagetable_t kpagetable, uint64 oldsz, uint64 newsz) {
  pte_t *u_pte, *k_pte;
  uint64 va, pa;

  if(oldsz < newsz)
    return;
  oldsz = PGROUNDDOWN(oldsz);
  for(va = oldsz; va < newsz; va += PGSIZE) {
    if ((u_pte = walk(upagetable, va, 0)) == 0) {
      panic("u2kvmcopy: va pte does not exit\n");
    } 
    if((k_pte = walk(kpagetable, va, 1)) == 0) {
      panic("u2kvmcopy: creat kernel pte failed\n");
    }
    pa = PTE2PA(*u_pte);
    uint64 flag = PTE_FLAGS(*u_pte) & (~PTE_U);
    *k_pte = PA2PTE(pa) | flag;
  }
}
```

- modify `fork() exec() sbrk()` to copy user address space into kernel address space when user space changes
```cpp
// fork 
  // increment reference counts on open file descriptors.
  for(i = 0; i < NOFILE; i++)
    if(p->ofile[i])
      np->ofile[i] = filedup(p->ofile[i]);
  np->cwd = idup(p->cwd);

  // copy parent user pagetable to child kernel page table
  u2kvmcopy(p->pagetable, np->kpagetable, 0, p->sz);

  safestrcpy(np->name, p->name, sizeof(p->name));

// exec
  // Allocate two pages at the next page boundary.
  // Use the second as the user stack.
  sz = PGROUNDUP(sz);
  uint64 sz1;
  if((sz1 = uvmalloc(pagetable, sz, sz + 2*PGSIZE)) == 0)
    goto bad;
  sz = sz1;
  uvmclear(pagetable, sz-2*PGSIZE);
  sp = sz;
  stackbase = sp - PGSIZE;

  u2kvmcopy(pagetable, p->kpagetable, 0, sz);

// growproc (sbrk)
// Grow or shrink user memory by n bytes.
// Return 0 on success, -1 on failure.
int
growproc(int n)
{
  uint sz;
  struct proc *p = myproc();

  sz = p->sz;
  if(n > 0){
    // user address range can not overlap kernel address range, the size of user space must be less than PLIC 
    if(sz + n > PLIC) 
      return -1;
    if((sz = uvmalloc(p->pagetable, sz, sz + n)) == 0) {
      return -1;
    }
  } else if(n < 0){
    sz = uvmdealloc(p->pagetable, sz, sz + n);
  }
  // user page table changed, also change kernel page table
  u2kvmcopy(p->pagetable, p->kpagetable, p->sz, sz);
  p->sz = sz;
  return 0;
}
```  

- modify `userinit()` in `kernel/proc.c` to map user address space to kernel in the first user process 
```cpp
// Set up first user process.
void
userinit(void)
{
  struct proc *p;

  p = allocproc();
  initproc = p;
  
  // allocate one user page and copy init's instructions
  // and data into it.
  uvminit(p->pagetable, initcode, sizeof(initcode));
  p->sz = PGSIZE;

  u2kvmcopy(p->pagetable, p->kpagetable, 0, p->sz);

  // prepare for the very first "return" from kernel to user.
  p->trapframe->epc = 0;      // user program counter
  p->trapframe->sp = PGSIZE;  // user stack pointer

  safestrcpy(p->name, "initcode", sizeof(p->name));
  p->cwd = namei("/");

  p->state = RUNNABLE;

  release(&p->lock);
}
```
- modify `copyin() copyinstr()`, just use `copyin_new() copyinstr_new()`
```cpp
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  return copyin_new(pagetable, dst, srcva, len);
}

int
copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
{
  return copyinstr_new(pagetable, dst, srcva, max);
}
``` 