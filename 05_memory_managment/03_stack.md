# Stack

## A hardware mechanism, not a software one

The stack is not an invention of C++, nor even of the operating system. It is a mechanism supported directly by the CPU.

On x86_64, a dedicated register **RSP (Stack Pointer)** always points to the top of the stack. Another register, **RBP (Base Pointer)**, points to the base of the current stack frame, making it easier to access local variables.

Allocating memory on the stack is literally a matter of subtracting a value from `RSP`.  
Freeing memory means adding a value back to `RSP`.
Both operations take about one CPU cycle.

There is:

- no search for free space  
- no data structures to update  
- no system calls  

This is why the stack is extraordinarily fast.

---

## The stack frame: anatomy of a function call

Each time a function is called, the CPU creates a **stack frame** (activation record).

This frame contains everything needed to:

- execute the function
- return to the caller safely

**Example:**

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

            add = (not yet assigned)


        Stack frame of add()

            return address → main
            saved RBP

            a = 10
            b = 20

            resut = 30

            ← RSP (top of stack)

    Low addresses
```

**Calling convention note**

On x86_64 Linux (System V AMD64 ABI):

- first integer arguments are passed in registers (`RDI`, `RSI`, etc.)
- not on the stack

This diagram is simplified for clarity. In reality, parameters are placed on the stack only if necessary.

---

## Lifetime is tied to scope

The most important property of the stack in C++ is:

- **Object lifetime is automatically tied to lexical scope**

```cpp
    #include <string>
    #include <iostream>

    void demonstrate_scope()
    {
        std::string first_name = "Jane";
        std::cout << first_name << "\n";

        {
            std::string last_name = "River";
            std::cout << last_name << "\n";
        }                                       // destructor of last_name called here

        std::cout << first_name << "\n";

    }                                           // destructor of first_name called here
```

When execution leaves a block:

- all local objects are destroyed
- destructors are called automatically

This behavior is the foundation of **RAII (Resource Acquisition Is Initialization)**.

Destruction is:

- deterministic
- immediate
- in reverse order of construction

Not “later”, not “eventually”, not “when a GC runs” — exactly at scope exit.

---

## Destruction order: guaranteed LIFO

Objects on the stack are destroyed in reverse order of construction.

**Example:**

```cpp
    #include <iostream>

    struct Trace
    {
        std::string name;

        Trace(std::string n) : name(std::move(n))
        {
            std::cout << "Construct " << name << "\n";
        }

        ~Trace()
        {
            std::cout << "Destroy " << name << "\n";
        }
    };

    void lifo_order()
    {
        std::cout << "Start\n";

        Trace a("A");
        Trace b("B");
        Trace c("C");

        std::cout << "End of scope\n";
    }
```

**Output:**

```text
    Start

    Construct A
    Construct B
    Construct C

    End of scope

    Destroy C
    Destroy B
    Destroy A
```

This is not just a detail.

It guarantees that:

- If object **B depends on A**, then object **B is destroyed before A**

Which is essential for correct resource management.

---

## Stack size: limited and configurable

The stack is not infinitely expandable.

On Linux:

- Each thread has its own stack. Default size is typically **~8 MB**

```bash
    ulimit -s  # typically: 8192 (KB)
```

You can modify it:

```bash
    ulimit -s 16384         # 16 MB
    ulimit -s unlimited     # not recommended
```

For threads:

```cpp
    #include <pthread.h>

    pthread_attr_t attr;
    pthread_attr_init(&attr);
    pthread_attr_setstacksize(&attr, 2 * 1024 * 1024);  // 2 MB

    pthread_t thread;
    pthread_create(&thread, &attr, my_function, nullptr);
    pthread_attr_destroy(&attr); 
```

**Why the limit exists**

- Detect infinite recursion (via stack overflow)
- Prevent uncontrolled memory usage
- Each thread has its own stack

Example:

- 100 threads × 8 MB = 800 MB virtual memory reserved

---

## Stack overflow

A stack overflow occurs when a program exceeds stack capacity.

**Infinite recursion**

```cpp
    void infinite_recursion()
    {
        int buffer[1000];       // ~4 KB per call

        infinite_recursion();
    }
```

Each call adds a new frame → stack fills up → crash (`SIGSEGV`).

**Large stack allocations**

```cpp
    void too_large()
    {
        double matrix[1000][1000];  // ~8 MB → likely crash
    }
```

---

**Rule of thumb**

Never allocate large objects on the stack.

Use:

- `std::vector`
- heap allocation

---

## Recursion: safe vs dangerous

Recursion is fine when depth is bounded.

**Safe example (tree)**

```cpp
    int sum(const Node* node)
    {
    if (!node) return 0;

    return node -> value + sum(node -> left) + sum(node -> right);
}
```
Balanced tree with 1M nodes:

- depth ≈ 20 → safe

**Dangerous example (linked list)**

```cpp
    // recursive (dangerous)

    int sum_recursive(const Node* node)
    {
        if (!node) return 0;

        return node -> value + sum_recursive(node -> next);
    }

    // iterative (safe)

    int sum_iterative(const Node* node)
    {
        int total = 0;

        while (node)
        {
            total += node -> value;
            node = node -> next;
        }

        return total;
    }
```

**Important note**

C++ does **not guarantee** tail call optimization. Do not rely on it to prevent stack overflow.

---

## Summary

- **Speed:** allocation is ~1–2 CPU cycles
- **Determinism:** destruction happens at a precise moment
- **Safety:** no leaks, no double free, no fragmentation
- **Limits:** small size, scope-bound lifetime, no sharing between threads

The stack is the default and preferred memory region.

When its limitations become a problem, large data, flexible lifetimes, sharing, the heap becomes necessary.

---
