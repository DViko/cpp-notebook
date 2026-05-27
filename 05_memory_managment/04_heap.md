# Memo: Heap

## Memory on demand

While the stack is a rigid, fast, and automatic mechanism, the heap is a flexible, vast, and fully manual memory space. It allows your program to request memory at any time, in an amount determined at runtime, and to retain it for as long as necessary — independently of function calls and scopes.

This freedom comes at a cost. Each heap allocation involves non-trivial work performed by a software component called the **memory allocator**. Every allocated byte becomes your responsibility: if you forget to free it, it remains occupied until the program terminates. This is the source of the vast majority of memory bugs in C++.

---

## The memory allocator: the invisible manager

When you write `new int(42)` in C++, here's what happens behind the scenes:

- operator `new` is invoked, which may internally call `malloc()` (implementation-dependent, for example, glibc on Linux).
- The allocator (**ptmalloc2**) searches for a sufficiently large free block.
- If found, the block is marked as occupied and its address is returned.
- Otherwise, the allocator requests memory from the Linux kernel via:

    - `brk()` — to extend the heap.
    - `mmap()` — for large allocations.

- The kernel updates the process page tables and returns **virtual memory**.
- The address is returned to your program.

The reverse process occurs during `delete`:

- operator `delete` releases the block (often via `free()`).
- The allocator marks it as free.
- Memory is usually **not returned immediately** to the OS, but kept for future reuse.

---

## Why heap allocation is more expensive

This layered mechanism explains why heap allocation is more expensive than stack allocation:

```text
    C++ code

    ▼

    operator new / operator delete          (C++ layer)

    ▼

    malloc / free                           (user-space allocator, for example ptmalloc2)
                                            manages free/used blocks
    ▼                                       avoids system calls when possible
                                   
    brk / mmap                              (system calls)
                                            kernel allocates virtual pages
    ▼

    Page Tables (MMU)                       (hardware)
                                            translates virtual → physical addresses
```

Each layer adds overhead, which explains why heap allocation is slower than stack allocation.

---

# The real cost of allocation

How expensive is new compared to stack allocation? It depends, but typical orders of magnitude are:

- **Stack allocation**: ~1–2 CPU cycles
- **Heap allocation (best case)**: ~50–100 ns
- **Heap allocation (worst case)**: several microseconds

These costs are negligible for a single allocation, but in tight loops or high-throughput systems, they become significant.

This is why high-performance C++ aims to minimize dynamic allocations by:

- reserving memory in advance (`std::vector::reserve`)
- using custom allocators
- preferring stack allocation when possible

---

## Fragmentation: the silent enemy

Unlike the stack, which remains compact due to its LIFO structure, the heap can become fragmented over time.

### Example

```cpp
    int* a = new int[1000];
    int* b = new int[1000];
    int* c = new int[1000];

    delete[] b;

    int* d = new int[2000];
```
Memory layout after freeing `b`:

```text  
    [a] [free] [c]
```

Even though total free memory is sufficient, there is no **contiguous block** large enough for `d`.

This is called **external fragmentation**.

**Internal fragmentation**:

- Allocators align memory (for example 8 or 16 bytes)
- Each block contains metadata

Allocating a single `int` may actually consume 16–32 bytes.

In long-running programs (servers, daemons), fragmentation can significantly increase memory usage.

---

## Lifetime: freedom and responsibility

The key strength of the heap is that object lifetime is not tied to scope:

```cpp

    #include <string>

    struct User
    {
        std::string name;
        int age;
    };

    User* create_user(const std::string& name, int age)
    {
        return new User{name, age};
    }

    void process()
    {
        User* alice = create_user("Alice", 30);

        // use alice...

        delete alice;
    }
```

This is essential for:

- dynamic data structures (trees, graphs, linked lists)
- polymorphism
- runtime-sized buffers
- cross-function or shared data

But every `new` must match exactly one `delete`:

- Missing `delete` → memory leak
- Double `delete` → undefined behavior
- Use after free → undefined behavior

---

## The heap is shared between threads

Unlike the stack (thread-local), the heap is shared across threads. This enables data sharing but introduces risks such as data races.

Allocators like **ptmalloc2** reduce contention using arenas:

- each thread gets a local arena

- reduces locking

- increases memory usage (each arena keeps its own free blocks)

---

## Lazy allocation and overcommit

Linux uses **overcommit** by default.

When memory is requested via `brk()` or `mmap()`:

- virtual memory is reserved

- physical memory is **not allocated immediately**

Physical pages are assigned only when accessed (*demand paging*).

**Consequences**

- `new` may succeed even if physical memory is insufficient

- failure may occur later during actual use

- the kernel may invoke the **OOM(Out Of Memory) Killer**

Catching `std::bad_alloc` does **not guarantee** future memory availability.

---

## When to use the heap

**Runtime size**

```cpp
    int num{};

    std::cin >> num;
    
    int array[num];                     // VLA(Variable Length Array) non-standard in C++, limited stack size

    std::vector<int> array(num);        // allocating memory on the heap using a vector
```

**Object must outlive its scope**

```cpp
    std::unique_ptr<Config> load_config(const std::string& path)
    {
        auto cfg = std::make_unique<Config>();

        cfg->load(path);

        return cfg;
    }
```

**Large objects**

```cpp
    // risky (stack overflow)

    double matrix[1000][1000];

    // safe
    
    std::vector<std::vector<double>> matrix(1000, std::vector<double>(1000));
```

**Polymorphism**

```cpp
    std::unique_ptr<Animal> create_animal(const std::string& type)
    {
        if (type == "cat") return std::make_unique<Cat>();

        if (type == "dog") return std::make_unique<Dog>();

        return std::make_unique<UnknownAnimal>();
    }
```

## Summary

- **Flexibility**: no constraints on lifetime or size
- **Capacity**: limited by system memory
- **Sharing**: accessible across threads
- **Cost**: slower than stack allocation
- **Fragmentation**: reduces efficiency over time
- **Responsibility**: manual management leads to errors

Modern C++ mitigates these risks using:

- **RAII**
- **smart pointers**

---
