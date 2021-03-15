# Lab Multithreading

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