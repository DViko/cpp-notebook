# Stack vs Heap

## Overview

Every running C++ program uses two main memory regions to store data: the **stack** and the **heap**.

Understanding the difference between these two areas is one of the most fundamental skills a C++ developer. It directly impacts your ability to write correct, efficient, and memory-safe code.

The distinction is simple in principle:

- the stack is **automatic**
- the heap is **manual**

But the implications go much deeper, affecting everything from object lifetime to performance optimization strategies.

---

## Analogy: stack of plates vs warehouse

Before diving into technical details, here is a simple analogy.

The **stack** works like a stack of plates in a restaurant:

- you place a plate on top
- you remove it from the top

It is fast, ordered, and requires no decisions. The next item to remove is always the last one added.

However, the size is limited. Try stacking a thousand plates, and the whole thing collapses.

The **heap** is more like a warehouse:

- you can store many objects (within available space)
- you can place them anywhere
- you can retrieve them in any order

This makes it flexible and large, but you must keep track of what you stored and where. If you forget to retrieve something, it occupies space indefinitely. Each operation is also slower because it involves searching for free space and updating bookkeeping structures.

This analogy is not perfect, but it highlights the key trade-offs:

- **Stack** → fast and automatic, but limited and rigid  
- **Heap** → flexible and large, but slower and manual

---

## When is the stack used?

The compiler automatically places **local variables** on the stack:

```cpp
    void process_request()
    {
        int status_code = 200;        // stack

        double response_time = 0.0;   // stack

        std::string message = "OK";   // object on stack (internal data may use heap)

        if (status_code == 200)
        {
            bool success = true;      // stack (limited to this block)
        }

        // 'success' no longer exists here
    }

    // all variables are destroyed here
```

Each function call creates a stack frame, which contains:

- local variables
- function parameters
- return address

When the function returns, the entire frame is destroyed instantly. This is just a matter of moving the **stack pointer**.

That is why the stack is so efficient:

- no search for free memory
- no fragmentation
- no delayed cleanup

Allocation and deallocation are reduced to a simple CPU register adjustment.

---

## When is the heap used?

The heap is used when:

- memory must outlive the function that created it
- size is not known at compile time

```cpp
    int* create_array(int size)
    {
        int* array = new int[size];  // heap

        return array;                // survives function return
    }

    void example()
    {
        int* arr = create_array(1000);

        // use arr...

        delete[] arr;                // manual deallocation required
    }
```

With the heap:

- you decide **when to allocate** (`new`)
- and **when to free** (`delete`)

This flexibility enables:

- dynamic data structures
- long-lived objects
- variable-sized buffers

But it comes at a cost:

- allocation is slower
- forgetting `delete` causes memory leaks


---

## The differences at a glance

```text
|               | Stack                     | Heap                          |
| :------------ | :------------------------ | :---------------------------- |
| Management    | Automatic                 | Manual                        |
| Allocation    | Stack pointer movement    | Allocator search              |
| Speed         | Very fast (~1 cycle)      | Slower (may involve syscalls) |
| Size          | Limited (≈1–8 MB typical) | Limited by system memory      |
| Lifetime      | Scope-based               | Until explicit deallocation   |
| Fragmentation | None                      | Possible over time            |
| Access        | LIFO                      | Arbitrary (via pointers)      |
| Threads       | Thread-local              | Shared between threads        |
```

**Notes**

- On Linux, the default stack size is typically ~8 MB (configurable via ulimit -s)
- This limit helps detect infinite recursion via stack overflow
- The heap can grow much larger (virtual memory, often many GB or more)

---

## A common pitfall: object vs resources

> ### Confusing the object with its resources:
>
> - #### One subtle point deserves to be raised now, as it is a frequent source of confusion:

```cpp
    void process()
    {
        std::vector<int> numbers = {1, 2, 3, 4, 5};
    }
```

Where is numbers stored?

- The `std::vector` object itself (pointer, size, capacity) → **stack**
- The actual elements → **heap**

When the function ends:

- the vector object is destroyed (stack)
- its destructor is called
- the **destructor frees** the **heap** memory

This is **RAII** in action.

---

## Stack + Heap together

This duality appears everywhere in modern C++:

- `std::vector`
- `std::string`
- `std::map`
- smart pointers (`std::unique_ptr`, `std::shared_ptr`)

Pattern:

- control object → stack
- managed data → heap
- destructor → cleanup bridge

This design is one of the core strengths of C++: it combines performance with automatic resource management when used correctly.

---
