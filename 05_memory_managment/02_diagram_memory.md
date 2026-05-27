# Process Memory Layout


## Virtual address space

When you run a compiled program on Linux, the kernel does not give it direct access to physical RAM.

Instead, it provides a **virtual address space**, a private, isolated view of memory. This space is translated into physical addresses by the hardware (**MMU — Memory Management Unit**) in cooperation with the kernel.

On a 64-bit system:

- the virtual address space is extremely large  
- on x86_64, typically up to **2⁴⁸ bytes (~256 TB)**  

However, only the regions actually used by the program are backed by physical memory.

Any access to an unmapped address results in a **segmentation fault**.

**Key consequences:**

- Each process behaves as if it owns the entire memory space

   → same virtual addresses can exist in different processes without conflict  

- Memory layout follows a **standard structure** 
 
   → defined by conventions between the OS and the compiler  

---

## The full memory layout

Typical C++ process memory layout on Linux (high → low addresses):

```text
            High addresses: 0x7FFF

                ----

                    KERNEL SPACE (inaccessible to the user program)

                ----

                    STACK (grows down)
                 
                        Stack frame : main → A → B
                        SR → current frame
                ↓           
                ↓
                ↓
                
                    free space

                ↑
                ↑
                ↑ 
                    HEAP (grows up)
                 
                        allocated block, free gaps

                ----

                    Memory-Mapped Region (mmap)

                        - shared libraries (.so)
                        - mmap files
                        - large allocations

                    .bss

                        - initialized globals

                    .data

                        - initialized globals

                    .rodata

                        - read only data

                    .text

                        - code


            Low addresses:  0x0040
```

## Memory regions explained

### .bss (uninitialized data)

Stores:

- global/static variables initialized to zero or not initialized

```cpp
    int buffer[10000];   // large array
    static long count;   // defaults to 0
```

Key property:

- occupies **no space in the executable**
- memory is zeroed by the kernel at load time

### .data (initialized data)

Stores:

- global/static variables initialized to non-zero values

```cpp
    int counter = 42;

    static double ratio = 3.14;

    void f()
    {
        static int calls = 1;
    }
```

**.rodata (read-only data)**

Stores:

- string literals
- const global data
- some constexpr values

```cpp
    const char* msg = "Hello";
```

- "Hello" → .rodata
- msg (pointer) → stack or .data

Writing to this memory → undefined behavior (usually a crash).

Values are stored in the executable file.

**.text (code segment)**

Contains compiled machine instructions.

- Read-only
- Executable
- Shared between processes

Attempting to modify it results in a segmentation fault.

### Memory-mapped region (mmap)

Used for:

- shared libraries (`libc.so, libstdc++`)
- memory-mapped files
- large allocations (typically >128 KB)

The allocator switches to `mmap()` for large blocks.

### Stack

- automatic memory for function calls
- one stack per thread
- grows downward

### Heap

- dynamic memory region
- managed by allocator (malloc / free, used by new / delete)
- grows upward

---

## Observing a real process

Example program:

```cpp
    #include <iostream>
    #include <unistd.h>

    int var_bss[1000];
    int var_data = 42;
    const char* message = "Hello";

    int main()
    {
        int var_stack = 10;
        int* var_heap = new int(99);

        std::cout << "PID: " << getpid() << "\n";

        std::cout << "Addresses:\n";
        std::cout << "  code   : " << (void*)&main << "\n";
        std::cout << "  rodata : " << (const void*)message << "\n";
        std::cout << "  data   : " << &var_data << "\n";
        std::cout << "  bss    : " << &var_bss << "\n";
        std::cout << "  heap   : " << var_heap << "\n";
        std::cout << "  stack  : " << &var_stack << "\n";

        std::cin.get();

        delete var_heap;
    }
```

Example output:

```text
    code   : 0x5612a3401189
    rodata : 0x5612a3402004
    data   : 0x5612a3404010
    bss    : 0x5612a3404040
    heap   : 0x5612a4e912a0
    stack  : 0x7ffd3b2a1e4c
```

Observation:

- `.text`, `.rodata` → low addresses
- **heap** → middle
- **stack** → high addresses

---

## ASLR (Address Space Layout Randomization)

Memory addresses change between executions due to **ASLR**.

Purpose:

- prevent memory exploitation attacks
- randomize locations of:
- stack
- heap
- libraries
- executable

Disable temporarily:

```bash
    setarch $(uname -m) -R ./program
```

**⚠️  `Never disable ASLR in production. It is a critical security protection.`**

## Summary

- A process uses virtual memory, not direct physical RAM
- Memory is divided into well-defined regions
- Stack and heap grow toward each other
- Only used memory is backed by physical pages
- Layout can be inspected using /proc and debugging tools

Understanding:

- where data lives
- how memory is used
- why crashes happen

---
