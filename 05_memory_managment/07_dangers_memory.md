# 📝 Memory leaks, dangling pointers, double frees

## 🔹 The infernal trio

Memory management errors are the primary cause of critical bugs, security vulnerabilities, and unexplained crashes in C++ programs.

They fall into three main categories, each with its own symptoms, causes, and consequences:

- **Memory leak :** allocated memory is never freed.
- **Dangling pointer :** a reference pointer to memory that has already been freed.
- **Double free :** the same memory block is freed twice.

These three categories are not independent — they often arise from the same weaknesses in the code: ambiguous properties, lack of RAII, manual handling of ` new/ delete `. Understanding each one thoroughly is a prerequisite for appreciating why modern C++ went to such lengths to eliminate them.

## 🔹 Memory leaks: the memory that never returns

### 🔸 The mechanism
A memory leak occurs when a block allocated on the heap is never freed, and no program pointer can locate it. The memory remains occupied until the end of the process, at which point the operating system reclaims it.

The most basic case:

```cpp
    void simple_leak()
    { 
        int* p = new int(42);
        // ... no delete
    }   // 'p' is destroyed (local variable on the stack), but the heap block remains allocated
        // no one knows the address anymore, irreversible leak
```

When the function returns, the variable ` p ` is destroyed. But the 4-byte block it pointed to on the heap is still there. No program variable contains its address anymore. It has become an orphan: impossible to free, impossible to reuse.

### 🔸 Pointer crush leaks

A common scenario is the reallocation of a pointer without freeing the old allocation:

```cpp
    void crash_leak()
    { 
        int* p = new int(10);   // allocation of A
        p = new int(20);        // allocation of B — the address of A is lost
        delete p;               // frees B, but A remains orphaned leak
    }
```

Before the second line, ` p ` there was the only path to allocation A. By reassigning ` p `, this path is destroyed. Allocation A is lost forever.