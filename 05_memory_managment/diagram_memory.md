# Memo: Memory of a process

When a program is run, the Linux kernel does not give it direct access to physical RAM. It assigns it a virtual address space — a private and isolated view of memory, translated into physical addresses by the hardware (MMU, Memory Management Unit ) in collaboration with the kernel.The rest simply doesn't exist from a process's perspective. Any access to an unallocated address results in a Segmentation fault.

This virtualization has two important consequences. First, each process believes it has all the memory to itself. Two programs can use the same virtual address without conflict, because they point to different physical locations. Second, the layout of memory regions is standardized. The kernel and the compiler agree on a conventional arrangement.

---

## Memory diagram

### Here is the memory map of a typical C++ process under Linux, from highest address to lowest:


```text
            High addresses: 0x7FFF

                ----

                    KERNEL SPACE (inaccessible to the user program)

                ----

                    STACK (growing downwards)
                 
                        Stack frame of main()
                 
                            - argc, argv
                            - local variables of main

                        Stack frame of functionA()

                            - return address → main
                            - parameters of functionA
                            - local variables of functionA

                        Stack frame of functionB() ← SP (stack pointer)

                            - return address → functionA
                            - parameters of functionB
                            - local variables of functionB
                ↓           
                ↓
                ↓
                
                    Free space

                ↑
                ↑
                ↑ 
                    HEAP (increasing upwards)
                 
                        - Allocated block (new int[100]) ← 400 bytes
                        - Free block (already deleted) ← hole
                        - Allocated Block (new MyObject()) ← 64 bytes

                ----

                    Memory-Mapped Region (mmap)

                        - Shared Libraries (.so): libc, libstdc++
                        - Mapped Files in memory
                        - Very large allocations (>128 KB using malloc/new)


                    .bss (Block Started by Symbol)

                        - Global and static variables NOT initialized (or initialized to zero)
                        - No space in the executable, set to zero upon kernel loading

                    .data

                        - Initialized Global and Static Variables
                        - Values ​​stored in the ELF executable

                    .rodata (Read-Only Data)

                        - Constants: string literals, const constants, constexpr value tables

                    .text (Code)

                        - Program Machine Instructions
                        - Read-only, shareable between processes


            Low addresses:  0x0040
```

## Details of each region

### .text(code)

This is where the program's machine instructions reside — the result of compilation.This area is in read-only, any attempt to write causes a Segmentation fault. This protection prevents the code from accidentally (or maliciously) modifying itself.

On Linux, if several instances of the same program run simultaneously, they share the same physical pages for .text section. The kernel loads these pages only once and shares them between processes.

### .rodata(read-only data)

String literals ( "Hello, world!"), `global` constants `const`, and some data constexpr are also stored here. As .text, this area is write protected:

```cpp
    const char* greet = "Hello World !";    // "Hello world !" is in .rodata
                                            // `greet` (the pointer) is on the stack or in .data
```

An attempt to modify the pointed-to content. For example via a const_cast — is undefined behavior which usually causes a crash.

### .data(initialized data)

Global and static variables explicitly initialized to a non-zero value are stored here. Their initial values ​​occupy space in the ELF executable on disk:

```cpp
    int global_counter = 42;        // .data
    static double ratio = 3.14;     // .data

    void function()
    {
        static int calls = 1;       // .data (local static, initialized to 1)
    }
```

### .bss(uninitialized data)

Global and static variables that are not initialized, or that are initialized to zero, are placed in the .bss section. The peculiarity of this segment is that it does not occupy space in the executable on disk—only its size is recorded. When the program loads, the kernel allocates the area and fills it with zeros.

```cpp
    int global_array[10000];    // .bss — 40,000 bytes in memory, / but ~0 bytes in the executable
    static long counter;        // .bss — initialized to 0 by the system
```

This is an important optimization: a program that declares large uninitialized global arrays does not produce a multi-megabyte executable.

### mmap (memory-mapped) region

Between the heap and the stack is an area used for memory mappings . This is where the dynamic loader ld-linux places shared libraries .so such as libc.so and libstdc++.so. This area is also used for memory-mapped files mmap()and for very large dynamic allocations — malloc(and therefore new) automatically switches to mmap beyond a threshold (typically 128 KB with glibc).

### heap

Dynamic memory area, managed by the allocator (glibc malloc/ free, which new/ deleteuse internally). The heap grows towards higher addresses. We will explore this in detail in section 5.1.3.

### stack

Automatic memory area for function calls. Each thread has its own stack. It grows towards lower addresses.

---

## View the memory of a real process

```cpp
                                            // demo_memory.cpp
    #include <iostream>
    #include <unistd.h>                     // getpid(), pause()

    int variable_bss[1000];                 // .bss  
    int variable_data = 42;                 // .data  
    const char* message = "Hello";          // pointer in .data, string in .rodata  

    int main()
    {
        int variable_stack = 10;            // stack
        int* variable_heap = new int(99);   // heap

        std::cout << "PID: " << getpid() << '\n';

        std::cout << ".text   (code)    : " << reinterpret_cast<void*>(&main) << '\n';
        std::cout << ".rodata (msg)     : " << static_cast<const void*>(message) << '\n';
        std::cout << ".data             : " << &variable_data << '\n';
        std::cout << ".bss              : " << &variable_bss << '\n';
        std::cout << "heap              : " << variable_heap << '\n';
        std::cout << "stack             : " << &variable_stack << '\n';

        std::cout << "\nPress Enter to exit...\n";
        std::cin.get();

        delete variable_heap;
        return 0;
    }
```

### The program's output (exact addresses may differ due to ASLR):

```text
    PID: 12345  

    .text   (code)    : 0x5612a3401189
    .rodata (msg)     : 0x5612a3402004
    .data             : 0x5612a3404010
    .bss              : 0x5612a3404040
    heap              : 0x5612a4e912a0
    stack             : 0x7ffd3b2a1e4c
```

Observe the address hierarchy: ` .text/` and ` .rodata/` are in the `lower` addresses (prefix / 0x56...), the `heap` is slightly `higher`, and the `stack` is in the `very high` addresses (prefix / 0x7ffd...). This is exactly the diagram seen above.

---

## ASLR: Why the addresses change with each execution

This isn't a coincidence: it's `ASLR` (Address Space Layout Randomization), a `security feature` enabled by default on modern Linux systems.

ASLR randomizes the position of the stack, heap, shared libraries, and even executable code (when compiled as a PIE, Position-Independent Executable, which is the default with GCC and Clang). The goal is to make memory corruption attacks (buffer overflow, return-oriented programming) much more difficult, because the attacker cannot predict the addresses.

With ASLR disabled, addresses become reproducible from one run to the next, which can facilitate experimentation with memory layout.

### ⚠️  `Never disable ASLR in production. It is a critical security protection.`

## Summary

A process's memory diagram isn't a theoretical concept. It's a reality that can be observed and analyzed on a computer. Every declared variable, every allocated object, every called function ultimately ends up within a specific region of this diagram.

---
