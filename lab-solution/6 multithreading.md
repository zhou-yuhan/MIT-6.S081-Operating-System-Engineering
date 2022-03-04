# Lab Multithreading
1. Uthread

- Add field `context` in struct `thread`
```cpp
struct context {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};

struct thread {
  struct context context; 
  // DO NOT use a pointer like struct contest* context, it points to nowhere unless you malloc for it in thread_create
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
};
```

- Complete `thread_switch` in `uthread_switch.S`, actually just copy from `swtch` in `kernel/swtch.S`. Add switch code in `thread_schedule`
```asm
thread_switch:
	/* YOUR CODE HERE */
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
	ret    /* return to ra */
```
```cpp
    /* YOUR CODE HERE
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
    thread_switch((uint64)&t->context, (uint64)&next_thread->context);
```

- Set `ra` to be the address of the thread function and set `sp` to be the address of the thread stack in `thread_creat`
```cpp
  // YOUR CODE HERE
  t->context.ra = (uint64)func;
  t->context.sp = (uint64)t->stack + STACK_SIZE; // The address of the bottom of t's stack is t->stack + STACK_SIZE
```

2. Using threads

The reason why there are missing keys with 2 threads:
```cpp
static void 
insert(int key, int value, struct entry **p, struct entry *n)
{
  struct entry *e = malloc(sizeof(struct entry));
  e->key = key;
  e->value = value;
  e->next = n;
  *p = e;
}
```
In `insert()`, assume that thread1 just executed `e->next = n` and is going to update table header `table[i]`.
If thread2 executes this line at the same time, two newly allocated pointer `e`s would point to the same `n`
in this bucket, but the table header can only stores either. Thus one new key is lost. 

The solution is rather simple, use a mutex lock to protect each bucket(for faster speed) and acquire the lock 
when calling `insert`

```cpp

pthread_mutex_t locks[NBUCKET];

void locksinit() {
  for(int i = 0; i < NBUCKET; ++i) 
    pthread_mutex_init(&locks[i], NULL);
}

static 
void put(int key, int value)
{
  int i = key % NBUCKET;

  // is the key already present?
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    pthread_mutex_lock(&locks[i]);
    insert(key, value, &table[i], table[i]);
    pthread_mutex_unlock(&locks[i]);
  }
}
```

call `locksinit()` in `main()`

3. Barrier

eazy to implement
```cpp
static void 
barrier()
{
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
  pthread_mutex_lock(&bstate.barrier_mutex);
  bstate.nthread++;
  // printf("call barrier, bstate.nthread = %d\n", bstate.nthread);
  if(bstate.nthread < nthread) {
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  } else {
    bstate.nthread = 0;
    bstate.round++;
    pthread_cond_broadcast(&bstate.barrier_cond);
  }
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```