# Lab Copy-on-Write Fork

1. implement copy-on-write fork

- Define `PTE_COW` using RSW bit in `kernel/riscv.h`
```cpp
#define PTE_COW (1L << 8) // Lab COW, record whether it is a COW page
```

- Creat a structure `kref` in `kernel/kalloc.c`to record reference count for each physical page. In consideration of concurrency, `kref` should be protected by a spinlock. Add functions `read_pageref() incr_pageref() decr_pageref()` to manipulate this structure. `kref_init()` initializes all counting slots to be 0.
```cpp
// Lab COW, physical page reference count 
struct pageref {
  struct spinlock lock;
  int count[PGROUNDDOWN(PHYSTOP) / PGSIZE];
} kref;

void kref_init() {
  for(int i = 0; i < PGROUNDDOWN(PHYSTOP) / PGSIZE; ++i) {
    kref.count[i] = 0;
  }
}

void incr_pageref(void *pa) {
  acquire(&kref.lock);
  uint64 index = PGIDX((uint64)pa);
  kref.count[index]++;
  release(&kref.lock);
}

void decr_pageref(void *pa) {
  acquire(&kref.lock);
  uint64 index = PGIDX((uint64)pa);
  kref.count[index]--;
  release(&kref.lock);
}

int read_pageref(void *pa) {
  acquire(&kref.lock);
  uint64 index = PGIDX((uint64)pa);
  int result = kref.count[index];
  release(&kref.lock);
  return result;
}
```

- Modify `kinit()` to initialize `kref.lock` and `kref.count`. Modify `kalloc()` and `kfree()` to increase or decrease count when allocating or freeing a physical page. `kfree()` frees a page if only reference count is 0.
```cpp
void
kinit()
{
  initlock(&kmem.lock, "kmem");
  initlock(&kref.lock, "kref");
  freerange(end, (void*)PHYSTOP);
  kref_init();
}

void
kfree(void *pa)
{
  decr_pageref(pa);
  if(read_pageref(pa) > 0) 
    return;
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");
    /* ... */

void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if(r) {
    memset((char*)r, 5, PGSIZE); // fill with junk
    incr_pageref((void*)r); // increase ref count during creation
  }
  return (void*)r;
}
```

- Add `cow_alloc()` to allocate a physical page if the corresponding PTE sets `PTE_COW` and do nothing otherwise.   
```cpp
// allocate a physical page to handle COW page fault.
// if PTE_COW is not set, do nothing since it's not a shared page
// return 0 on success, -1 on error 
int cow_alloc(pagetable_t pagetable, uint64 va) {
  uint64 pa;
  pte_t *pte;
  uint flags;

  if(va >= MAXVA)
    return -1;
  va = PGROUNDDOWN(va);
  if((pte = walk(pagetable, va, 0)) == 0) 
    return -1;
  pa = PTE2PA(*pte);
  if(pa == 0)
    return -1;
  flags = PTE_FLAGS(*pte);

  if(flags & PTE_COW) {
    flags = (flags | PTE_W) & ~PTE_COW;
    void *mem = kalloc();
    if(mem == 0) 
      return -1;
    memmove(mem, (void*)pa, PGSIZE);
    kfree((void*)pa); // this physical page is no longer mapped to va, free it(decrease reference count) 
    *pte = PA2PTE(mem) | flags; // install the new page with PTE_W set
  }
  return 0;
}
```
- Modify `uvmcopy()` in `kernel/vm.c`, map the same page and do not allocate page or copy physical page
```cpp
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  // char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
      
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);    
    // clear PTE_W and set PTE_COW
    if(flags & PTE_W) {
      flags = (flags | PTE_COW) & ~PTE_W;
      *pte = PA2PTE(pa) | flags;
    }
    /*
    if((mem = kalloc()) == 0)
      goto err;
    memmove(mem, (char*)pa, PGSIZE);
    */
    if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0){
      goto err;
    }    
    incr_pageref((void*)pa);
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```

- Modify `usertrap()` in `kernel/trap.c` to handle page fault
```cpp
  } else if((which_dev = devintr()) != 0){
    // ok
  } else if(r_scause() == 13 || r_scause() == 15) {
    // page fault, handle COW
    uint64 va = r_stval();
    if(va >= MAXVA || (va <= PGROUNDDOWN(p->trapframe->sp) && va >= PGROUNDDOWN(p->trapframe->sp) - PGSIZE)) {
      // illegal address, beyond max virtual address or in guard page
      p->killed = 1;
    } else {
      if(cow_alloc(p->pagetable, va) != 0)
        p->killed = 1;
    }
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }
```

- `usertrap()` does not handle page fault that happens in kernel, thus modify `copyout()` when kernel tries to write something to user space and a page fault happens
```cpp
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);

    if(cow_alloc(pagetable, va0) != 0) // detect COW, allocate and map if necessary
      return -1;

    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}
``` 