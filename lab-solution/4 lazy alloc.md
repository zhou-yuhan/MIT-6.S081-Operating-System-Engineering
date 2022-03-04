# Lab Lazy Allocation

1. Eliminate allocation from `sbrk()`

```cpp
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  // if(growproc(n) < 0)
    // return -1;
  myproc()->sz += n;
  return addr;
}
```

2. Lazy allocation
3. Lazytests and Usertests

- Handle page fault in `usertrap()` in `kernel/trap.c`, allocate a physical page, kill the process if virtual address is illegal or run out of memory
```cpp
  } else if ((which_dev = devintr()) != 0) {
    // ok
  } else if (r_scause() == 15 || r_scause() == 13) {
    // page fault
    pagetable_t pagetable = p->pagetable;
    uint64 va = r_stval();

    if (va >= p->sz || va < p->trapframe->sp) {
      // check if the virtual address is legal
      p->killed = 1;
    } else {
      char *mem = kalloc();
      va = PGROUNDDOWN(va);
      if (mem == 0) {
        printf("run out of memory\n");
        p->killed = 1;  // allocate physical memory failed, kill the process
      } else {
        memset(mem, 0, PGSIZE);
        if (mappages(pagetable, va, PGSIZE, (uint64)mem,
                     PTE_W | PTE_X | PTE_R | PTE_U) != 0) {
          kfree(mem);
          p->killed = 1;  // map virtual memory failed, kill the process
        }
      }
    }

  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }
```

- Modify `uvmunmap()` in `kernel/vm.c` to avoid panic when unmapping an unallocated page
```cpp
// Remove npages of mappings starting from va. va must be
// page-aligned. The mappings must exist.
// Optionally free the physical memory.
void uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free) {
  uint64 a;
  pte_t *pte;

  if ((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for (a = va; a < va + npages * PGSIZE; a += PGSIZE) {
    if ((pte = walk(pagetable, a, 0)) == 0)
      // panic("uvmunmap: walk");
      continue;
    if ((*pte & PTE_V) == 0)
      // panic("uvmunmap: not mapped");
      continue;
    if (PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if (do_free && (*pte & PTE_V)) {
      uint64 pa = PTE2PA(*pte);
      kfree((void *)pa);
    }
    *pte = 0;
  }
}
```

- `fork()` calls `uvmcopy()` to copy both page table and physical memory to child process from parent, modify `uvmcopy()` in `kernel/vm.c` to avoid panic when a PTE is invalid
```cpp
// Given a parent process's page table, copy
// its memory into a child's page table.
// Copies both the page table and the
// physical memory.
// returns 0 on success, -1 on failure.
// frees any allocated pages on failure.
int uvmcopy(pagetable_t old, pagetable_t new, uint64 sz) {
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  char *mem;

  for (i = 0; i < sz; i += PGSIZE) {
    if ((pte = walk(old, i, 0)) == 0)
      // panic("uvmcopy: pte should exist");
      continue;
    if ((*pte & PTE_V) == 0) 
      // panic("uvmcopy: page not present");
      continue;
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    if ((mem = kalloc()) == 0)
      goto err;
    memmove(mem, (char *)pa, PGSIZE);
    if (mappages(new, i, PGSIZE, (uint64)mem, flags) != 0) {
      kfree(mem);
      goto err;
    }
  }
  return 0;

err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```

- Handle negative `sbrk()` argument, if `n < 0` then shrink the virtual address space, unmap and free physical memory (i.e. lazy allocation but eager deallocation)

```cpp
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  // if(growproc(n) < 0)
    // return -1;
  myproc()->sz += n;
  if(n < 0)
    // shrink memory, unmap and deallocate physical memory
    // or just call growproc(n), which will call the function below
    uvmdealloc(myproc()->pagetable, addr, myproc()->sz);
  return addr;
}
```

- Handle the case that page fault happpens in kernel space, e.g. when handling system calls like `read() write()`. In this case kernel will call `walkaddr()` in `kernel/vm.c` to get the physical address of the user virtual address. Modify `walkaddr()` to allocate a physical page and return it instead of return 0 when facing with an invalid PTE

```cpp
// Look up a virtual address, return the physical address,
// or 0 if not mapped.
// Can only be used to look up user pages.
uint64 walkaddr(pagetable_t pagetable, uint64 va) {
  pte_t *pte;
  uint64 pa;

  if (va >= MAXVA)
    return 0;

  pte = walk(pagetable, va, 0);
  if (pte == 0 || (*pte & PTE_V) == 0) {
    // lazy allocation, allocate a physical page
    struct proc *p = myproc();
    if (va >= p->sz || va < p->trapframe->sp)
      return 0;

    char *mem = kalloc();
    va = PGROUNDDOWN(va);
    if (mem == 0) {
      return 0;
    } else {
      memset(mem, 0, PGSIZE);
      if (mappages(pagetable, va, PGSIZE, (uint64)mem,
                   PTE_W | PTE_X | PTE_R | PTE_U) != 0) {
        kfree(mem);
        return 0;
      }
    }
    // get mapped pte to return the physical address
    pte = walk(pagetable, va, 0);
  }
  if ((*pte & PTE_U) == 0)
    return 0;
  pa = PTE2PA(*pte);
  return pa;
}
``