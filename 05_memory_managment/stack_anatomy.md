# Memo: Stack

## Hardware mechanism, not software

The stack isn't an invention of the C++ language or even the operating system. It's a mechanism directly supported by the processor. On the x86_64 architecture, a dedicated register, RSP (Stack Pointer Register), constantly points to the top of the stack. A second register, RBP (Base Pointer Register), points to the base of the current stack frame, allowing for easy navigation between local variables.

Allocating memory on the stack is literally the same as subtracting a value from RSP. Freeing that memory is the same as adding a value to RSP. Both of these operations take one processor cycle each. There's no space to search for, no data structure to update, no system call. This is what makes the stack incredibly fast.

---

## Stack frame: anatomy of a function call

Every time a function is called, the processor creates a stack frame on the stack. This frame contains everything the function needs to execute and everything it needs to return correctly to the caller.

Example:

```cpp
    int add(int a, int b)
    {
        int result = a + b;

        return result;
    }

    int main()
    {
        int x = 10;
        int y = 20;

        int sum = add(x, y);

        return 0;
    }
```

At the moment of the append operation, the stack looks like this:

```text
    High addresses

        Stack frame of main()

            x = 10
            y = 20

            add = not yet assigned

    RBP of main

        Stack frame of add()

        Return address to main()

        Former RBP backed up

            a = 10  parameter
            b = 20  parameter

            resut = 30

    RSP top of stack

    Low addresses
```

When the add() process terminates, its stack frame is destroyed by restoring the previous RSP frame. Execution resumes from the saved return address. The bytes occupied by the previous add() frame are not "erased" they remain in memory, but are considered free and will be overwritten the next time the function is called.

---

## Lifespan is related to scope

The most important property of the stack from a C++ programmer's perspective is that the lifetime of variables is automatically linked to their scope. When execution exits a block {}, all variables declared within that block are destroyed and if they are C++ objects, their destructors are called.

```cpp
    #include <string>
    #include <iostream>

    void demo_scope()
    {
        std::string first_name = "Jane";        // built on the stack
        std::cout << first_name << "\n";

        {
            std::string last_name = "Desert";   // built on the stack (internal scope)
            std::cout << last_name << "\n";
        }                                       // destructor of object 'last_name' called here. Internal memory freed

        std::cout << first_name << "\n";        // 'last_name' no longer exists, but 'first_name' is still valid

    }                                           // destructor of object 'first_name' called here
```

This automatic destruction mechanism at the end of scope forms the basis of RAII (Resource Acquisition Is Initialization), the most fundamental pattern in modern C++. But the essential idea is that the stack guarantees that every correctly constructed object will be destroyed "sometime" not "when the garbage collector comes", but precisely at the closing of the enclosing block, in the reverse order of construction.

---

## Destruction order: LIFO

Objects on the stack are destroyed in the reverse order of their creation. This is a direct consequence of the LIFO (Last In, First Out) structure of the stack, and it is a guarantee of the language:

```cpp
    #include <iostream>

    struct Trace
    {
        std::string name_{};

        Trace(std::string name) : name_(std::move(name))
        {
            std::cout << "  Construction of " << name_ << "\n";
        }

        ~Trace()
        {
            std::cout << "  Destruction of " << name << "\n";
        }
    };

    void order_lifo()
    {
        std::cout << "Beginning\n";

        Trace a("A");
        Trace b("B");
        Trace c("C");

        std::cout << "End of block\n";
    }
```

Result:

```text
    Beginning

    Construction of A
    Construction of B
    Construction of C

    End of block

    Destruction of C
    Destruction of B
    Destruction of A
```

`This reversed order is not a minor detail`. It ensures that `if an object B depends on an object A` built before, `B will be destroyed first`. This is a crucial property for proper resource management.

---

## Stack size

The stack is not infinitely expandable. Under Linux, each thread has a fixed-size stack, determined at startup. The default value is usually 8 MB for the main thread. For threads created with std::threador pthread_create, the default stack size is also 8 MB on most distributions, but it is configurable via the pthread attributes:

```cpp
    #include <pthread.h>

    pthread_attr_t attr;  
    pthread_attr_init(&attr);  
    pthread_attr_setstacksize(&attr, 2 * 1024 * 1024);

    pthread_t thread;  
    pthread_create(&thread, &attr, my_function, nullptr);  
    pthread_attr_destroy(&attr);  
```

Why this limit? Two reasons. First, it allows the system to detect infinite recursion. When the stack overflows, the program receives a signal SIGSEGV rather than silently consuming all the memory. Second, in a multi-threaded program, each thread has its own stack. If you create 100 threads with 8 MB stacks, that already represents 800 MB of reserved address space (even though physical pages are only allocated when they are actually used, thanks to the kernel's lazy allocation).

---

## Stack overflow

A stack overflow occurs when a program attempts to use more space than the stack can provide. The most common cause is unbounded recursion:

```cpp
                            // ⚠️ Infinite recursion guaranteed stack overflow
    void my_recursion()
    {
        int array[1000];    // 4 000 bytes per call

        my_recursion();     // Each call stacks a new frame
    }
```

Each recursive call adds a stack frame. With a local array of 4,000 bytes per frame, the 8 MB stack is saturated after approximately 2,000 calls. The result is a signal SIGSEGV the infamous Segmentation Fault .

But infinite recursion isn't the only cause. Allocating large structures on the stack can also cause an overflow:

```cpp
    void allocation_excessive()
    {
        double matrix[1000][1000];   // 8 MB at a time probable stack overflow
    }
```

A 1000 × 1000 array double occupies 8,000,000 bytes, which is almost the entire default stack. In practice, the frame of the calling function already occupies part of the stack, so this code will very likely crash.

`The practical rule is:` never allocate large structures on the stack. If you need a significant size array, use one std::vector that stores its data on the heap, or explicitly allocate it on the heap.

---

## Correct recursion and reasonable depth

Recursion is not to be dismissed, it is a powerful tool when depth is controlled. A balanced binary tree traversal of 1 million nodes requires only about 20 levels of recursion (log₂(1,000,000) ≈ 20), which is perfectly manageable:

```cpp
    struct Node
    {
        int value {};
        Node* left {};
        Node* right{};
    };

    int sum(const Node* node)
    {
        if (node == nullptr)
        {
            return 0;
        }

        return node -> value + sum(node -> left) + sum(node -> right); // Maximum depth ~20 for a balanced tree of 1M nodes without risk
    }
```

Conversely, recursively traversing a linked list of 1 million elements pushes 1 million frames onto the stack, guaranteed stack overflow. The iterative version is therefore imperative:

```cpp

    // Recursive — stack overflow for a long list

    int summ_recursive(const Node* node)
    {
        if (node == nullptr)
        {
            return 0;
        }

        return node -> value + summ_recursive(node -> next);  // 1M calls crash
    }

    // Iterative — uses a constant stack space

    int summ_iterative(const Node* node)
    {
        int total = 0;

        while (node != nullptr)
        {
            total += node -> value;

            node = node -> next;
        }

        return total;
    }
```

> **Note:** Unlike some functional languages, C++ does not guarantee tail call optimization. GCC and Clang can apply it to -O2 C++ or higher, but it is not guaranteed by the standard. Do not rely on it to avoid a stack overflow.

---

## Summary

Let's recap the essential characteristics of the stack that make it the preferred memory area for local variables:

- Speed. Allocating and freeing memory on the stack takes one or two CPU cycles. No other allocation mechanism can compete.

- Determinism. Destruction occurs at a precise and predictable point in the program (the scope exit), in a guaranteed order (reverse LIFO). There is no uncertainty about the "when" or the "in what order".

- Security. No memory leaks possible, everything allocated on the stack is automatically freed. No double free, the release is not under your control. No fragmentation, the LIFO structure prevents it by design.

- The limitations. The size is fixed and relatively modest (8 MB by default). The lifespan is rigidly tied to the scope, it's impossible to keep a variable alive beyond its function without copying or moving it. And sharing between threads is impossibl,  each thread has its own stack.

- It is precisely when these limitations become an obstacle excessive data volume, flexible lifespan, sharing between components that the heap comes into play. This is the subject of the following section.

---
