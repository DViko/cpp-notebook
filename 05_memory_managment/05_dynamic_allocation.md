# Dynamic Allocation: new/delete, new[]/delete[]

## The fundamental contract

Dynamic memory allocation in C++ is governed by a simple and unforgiving contract:

> every new must be matched by exactly one delete, and every new[] by exactly one delete[].

- No `delete` → memory leak.
- Two `delete` calls → undefined behavior.
- Wrong form of `delete` → undefined behavior.

There is no tolerance layer, no runtime safety net built into the language.

This strict contract is precisely why modern C++ strongly discourages raw `new/delete` usage. However, understanding them is essential to understanding what smart pointers actually abstract.

---

## Allocating and freeing a single object

**Basic syntax:**

The `new` operator **allocates memory on the heap**, constructs an object in-place, and returns a pointer:

```cpp
int* p = new int;        // uninitialized int on the heap
int* q = new int(42);    // initialized to 42
int* r = new int{42};    // uniform initialization (C++11)
```

Usage:

```cpp
std::cout << *q << "\n"; // 42
```

Deallocation:

```cpp
delete p;
delete q;
delete r;
```

`delete` **calls the destructor** (trivial for int, essential for complex types) and then **releases memory back** to the allocator.

---

## With user-defined types

Dynamic allocation becomes meaningful when constructors perform real work:

```cpp
#include <string>
#include <iostream>

class Connection
{
    public:
        Connection(const std::string& host, int port) : host_(host), port_(port)
        {
            std::cout << "Connection opened to " << host_ << ":" << port_ << "\n";
        }

        ~Connection()
        {
            std::cout << "Connection closed (" << host_ << ":" << port_ << ")\n";
        }

        void send(const std::string& message)
        {
            std::cout << "Sending: " << message << "\n";
        }

    private:
        std::string host_;
        int port_;
};

```

```cpp
int main()
{
    Connection* conn = new Connection("server.local", 8080);

    conn->send("ping");

    delete conn;

    return 0;
}
```

Output:

```text
Connection opened to server.local:8080
Sending: ping
Connection closed (server.local:8080)
```

**Key idea:**

delete is not just memory release — it triggers destruction first.

This is a major difference from C’s free(), which only releases raw memory.

---

## Allocating and freeing arrays

`new[] / delete[]` for dynamic arrays:

```cpp
double* temperatures = new double[5];

temperatures[0] = 18.5;
temperatures[1] = 20.3;
temperatures[2] = 22.1;
temperatures[3] = 19.7;
temperatures[4] = 21.0;

delete[] temperatures;
```

Initialization variants:

```cpp
int* values = new int[4]{10, 20, 30, 40};
int* zeros  = new int[100]();   // zero-initialized
```

## Arrays of objects

When allocating arrays of objects, C++:

- calls default constructor for each element
- calls destructor for each element in reverse order on deletion

```cpp
#include <iostream>

class Sensor {
public:
    Sensor()  { std::cout << "  Sensor constructed\n"; }
    ~Sensor() { std::cout << "  Sensor destroyed\n"; }
};
int main()
{
    std::cout << "Allocating 3 sensors:\n";
    Sensor* sensors = new Sensor[3];

    std::cout << "\nDeallocating:\n";
    delete[] sensors;
}
```

---

## The strict rule: matching forms

This is critical:

| Allocation | Deallocation | Valid |
| :--------- | :----------- | :---- |
| new        | delete       | ✅     |
| new[]      | delete[]     | ✅     |
| new        | delete[]     | ❌ UB  |
| new[]      | delete       | ❌ UB  |

Why ?

Because `new[]` stores hidden metadata (array size) so `delete[]` can call destructors correctly.

Mismatching the forms leads to:

- missing destructors
- corrupted allocator metadata
- undefined behavior that may appear “fine” for months before exploding under load

**delete on nullptr is safe:**

```cpp
int* p = nullptr;

delete p;     // safe
delete[] p;   // safe
```

This simplifies cleanup logic in legacy code.

---

## Dynamic size allocation

Runtime-sized arrays:

```cpp
void process_data(int n)
{
    double* buffer = new double[n];

    for (int i = 0; i < n; ++i)
    {
        buffer[i] = i * 1.5;
    }

    double sum = 0.0;

    for (int i = 0; i < n; ++i)
    {
        sum += buffer[i];
    }

    std::cout << "Sum: " << sum << "\n";

    delete[] buffer;
}
```

Exception safety problem:

```cpp
void fragile(int n)
{
    double* buffer = new double[n];

    function_that_may_throw();

    delete[] buffer; // never reached
}
```

If an exception occurs before `delete[]`, memory leaks.

---

## The RAII fix

```cpp
void robust(int n)
{
    std::vector<double> buffer(n);

    function_that_may_throw();

} // automatic cleanup here
```

Even on exceptions, stack unwinding ensures destruction.

---

## What `new` actually does

Two-step process:

1. Allocate raw memory (operator new → malloc)
2. Construct object in-place

`delete` reverses it:

1. Call destructor
2. Free memory (operator delete → free)

---

## Mixing allocation APIs is undefined behavior

```cpp
int* p1 = new int(42);
free(p1);       // UB (destructor not called)

int* p2 = (int*)malloc(sizeof(int));
delete p2;      // UB

int* p3 = new int(42);
delete p3;      // correct
```

---

## Allocation failure: std::bad_alloc

```cpp
try {
    long long* p = new long long[1'000'000'000'000'000'000LL];
    delete[] p;
}
catch (const std::bad_alloc& e) {
    std::cerr << "Allocation failed: " << e.what() << "\n";
}
```

nothrow **variant**

```cpp
int* p = new(std::nothrow) int[1'000'000];

if (!p) {
    std::cerr << "Allocation failed\n";
}
```

const with dynamic allocation

```cpp
const int* p = new const int(42);

std::cout << *p;

delete p;
```
---

## Key rules summary

- strict `new ↔ delete / new[] ↔ delete[]` pairing
- exactly one deletion per allocation
- ownership must be explicit
- exceptions break naive cleanup logic
- prefer RAII (`vector`, `unique_ptr`, `shared_ptr`)

---