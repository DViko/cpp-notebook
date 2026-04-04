# Memo: Stack vs Heap

## Overview

> #### Every running C++ program has two main memory areas for storing its data: the `stack` and the `heap`. Understanding the difference between these two areas >is probably the most fundamental skill a C++ developer can acquire. It determines your ability to write correct, efficient, and leak-free code

> #### The distinction is simple in principle: the stack is `automatic`, the heap is `manual`. But the implications of this difference permeate the entire language, from object lifetimes to production optimization strategies

---

## Analogy

> ### The stack works like a stack of plates in a restaurant:
> 
> - ####  You place a plate on top, then remove it from the top. It's quick, tidy, and you don't have to make any decisions—the next plate to remove is always the one on top. However, the size of the stack is limited: try stacking a thousand plates and the stack collapses

> ### The heap functions like a warehouse:
> 
> - #### You can store as many objects as you want (within the limits of available space), place them wherever you like, and remove them in any order. It's flexible and spacious, but you must keep a record of what you've stored and where. If you forget to retrieve an object, it occupies space indefinitely. And each storage or retrieval operation takes longer than on the stack because you have to find an available space, update the record, and so on

> ### Two fundamental tensions:
>
> - **`Stack` : fast and automatic, but limited in size and flexibility**
> - **`Heap` : flexible and vast, but slower and entirely under your control**

---

## When is the stack used?

> **`The compiler automatically places all local variables on the stack` — those that you declare inside a function or block:**

```cpp
    void process_request()
    {
        int code = 200;                 // stack
        double response_time = 0.0;     // stack
        std::string status = "OK";      // the string object (small, fixed size) is on the stack
                                        // the actual characters may be on the heap,
                                        // or inside the object itself if SSO applies

        if (code == 200)
        {
            bool success = true;        // stack (scope limited to the `if` block)
        }                               // `success` no longer exists here — automatically removed
    }                                   // `code`, `response_time` and `status` no longer exist here
```


> #### Each function call creates a stack frame containing the function's local variables, parameters, and return address to know where to resume execution once the function has finished. When the function terminates, its stack frame is completely destroyed — it's instantaneous; you simply move the stack pointer.

> #### This mechanism is what makes the stack so efficient: there is no searching for free space, no fragmentation, no deferred cleanup. Allocation and deallocation are reduced to a simple adjustment of a CPU register (the stack pointer).

---

## When is the heap used?

#### The heap comes into play as soon as you need memory whose lifetime exceeds the scope of the function that created it, or whose size is not known at compile time:

```cpp
    int* create_array(int size)
    {
        int* array = new int[size];     // heap: size determined at runtime

        return array;                   // the array survives the function
    }

    void example()
    {
        int* array = create_array(1000);

        // use of array
        
        delete[] array;                 // manual deallocation required
                                        // note: `new[]` must be paired with `delete[]`
    }
```

> #### With the `heap`, you decide when memory is allocated using the `new` operator and when it is freed using the `delete` operator. This freedom is powerful — it allows the creation of dynamic data structures, objects with arbitrary lifetimes, and buffers of variable size — but it comes at a cost: each allocation requires the memory allocator to search for free space, and each failure to `delete` causes a `memory leak`.

### Modern C++ note:

> #### In real-world code, you rarely call `new/delete` directly. Prefer `std::unique_ptr, std::shared_ptr`, or STL containers — they manage heap memory automatically following the RAII principle.


---

## The differences at a glance

```
|               | Stack                         | Heap                                 |
| :------------ | :---------------------------- | :----------------------------------- |
| Management    | Automatic (compiler)          | Manual (developer)                   |
| Allocation    | Increment/decrement stack ptr | Searching for free space (allocator) |
| Speed         | Very fast (~1-10 CPU cycles)  | Slower (may use system calls)        |
| Typical size  | Small (1-8 MB)                | Limited by available RAM/swap        |
| Lifetime      | Scope-based (block/function)  | Until explicit delete                |
| Fragmentation | None                          | Possible over time                   |
| Access order  | LIFO only                     | Random (via pointer)                 |
| Thread-safety | Each thread has its own stack | Shared (needs synchronization)       |
```


> ### A few important points about this table:
> 
> - #### The default stack size under Linux is generally 8 MB. This is not a bug or an arbitrary limitation: this size is sufficient for the vast majority of programs, and limiting it allows the operating system to detect infinite recursion via a stack overflow rather than letting the program consume all the memory.
>
> - #### The heap, on the other hand, can theoretically use all available virtual memory — several terabytes on a modern 64-bit system. In practice, the limit is physical RAM plus swap space.

---

## Common pitfall

> ### Confusing the object with its resources:
>
> - #### One subtle point deserves to be raised now, as it is a frequent source of confusion:

```cpp
    void process()
    {
        std::vector<int> numbers { 1, 2, 3, 4, 5 };

        // Where is `numbers`?
    }
```

> #### The answer is `both`. The numbers object itself — that is, `the vector's control` structure (internal pointer, size, capacity) — is `on the stack`. But `the elements { 1, 2, 3, 4, 5 }` are stored in a buffer allocated `on the heap` by the vector internally.

> #### When `function process finishes`, numbers is destroyed (popped from the stack). Its `destructor is automatically called`, and this destructor releases the internal buffer on the heap. This is `the RAII principle` in action.

> #### This `stack/heap` duality is omnipresent with `STL containers` (std::vector, std::string, std::map ...) and `smart pointers` (std::unique_ptr, std::shared_ptr). The `control object` lives `on the stack`, `the data` it manages lives `on the heap`, and the destructor bridges the two.

---

## Key takeaways

- **Stack:** fast, automatic, but small and LIFO-only. Use it for local variables with short, predictable lifetimes.

- **Heap:** flexible, spacious, but slower and manual. Use it for data that must outlive the function that created it, or when size is unknown at compile time.

- Always pair `new with delete`, `new[] with delete[]`. Better yet, avoid raw new/delete entirely — `use RAII wrappers`.

- Don't confuse an object's location with its resources' location — `many stack objects manage heap memory` behind the scenes.

---
