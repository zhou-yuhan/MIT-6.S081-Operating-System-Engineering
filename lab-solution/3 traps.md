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

Haven't implemented