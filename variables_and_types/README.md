# Memo: Variables and types

## Fundamental types

### Integer types:


| Type      | Typical size (x86-64) |
| :-------- | :-------------------- |
| char      | 1 byte                |
| short     | 2 byte                |
| long      | 8 byte                |
| long long | 8 byte                |

### Important: Type sizes depend on the platform and compiler. On Windows, long is typically 4 bytes on Linux it's 8.

---

### Unsigned versions:

| Type               | Typical size (x86-64) |
| :----------------- | :-------------------- |
| unsigned char      | 1 byte                |
| unsigned short     | 2 byte                |
| unsigned int       | 8 byte                |
| unsigned long      | 8 byte                |
| unsigned long long | 8 byte                |

---

### Real types:

| Type        | Typical size (x86-64) | Accuracy    |
| :---------- | :-------------------- | :---------- |
| float       | 4 byte                | ~7 char     |
| double      | 8 byte                | ~15 char    |
| long double | 16 bytes              | ~19-20 char |

---

### Char types:

| Type          | Typical size (x86-64) |
| :------------ | :-------------------- |
| char          | 1 byte                |
| signed char   | 1 byte                |
| unsigned char | 1 byte                |
| wchar_t       |                       |
| char16_t      | 16 bytes              |
| char32_t      | 32 bytes              |

---

### Logical types:

| Type | Typical size (x86-64) |
| :--- | :-------------------- |
| bool | 1 byte                |

---

### Exact types:

| Type     | Typical size (x86-64) |
| :------- | :-------------------- |
| int8_t   | 8 byte                |
| int16_t  | 16 byte               |
| int32_t  | 32 byte               |
| int64_t  | 64 byte               |
| uint8_t, | 8 byte                |
| uint16_t | 16 bytes              |
| uint32_t | 32 bytes              |
| uint64_t | 32 bytes              |
| size_t   |                       |

---

## Initialization of variables

### There are several ways to initialize in C++, and the choice affects the behavior

```cpp

cpp
    int a = 10;         // copy initialization
    int b( 20 );        // direct initialization
    int c{ 30 };        // uniform initialization
    int d = { 40 };     // copy list initialization
```

### Key rule: list initialization prohibits narrowing conversions:

```cpp
    int x{ 3.14 };      // Compilation ERROR!
    int y = 3.14;       // OK, but y = 3 (data loss)
```

### Examples of narrowing that the compiler will catch:

```cpp
    char c1{ 999 };           // error: 999 doesn't fit in char
    unsigned char uc2{ -1 };  // error: negative value
    int ii{ 2.0 };            // error: double -> int [citation:3]
    float f1{ x };            // error: int -> float may lose precision
```

### Value initialization (zeroing):

```cpp
    int z{};        // z = 0
    int* ptr{};     // ptr = nullptr
    bool flag{};    // flag = false
```

### Default initialization:

```cpp
    int a;      // NOT initialized. Contains garbage
    int b{};    // Initialized to zero
```


## Type inference (C++11 and later)


### `auto` automatic type detection:

```cpp
    auto x = 42;    // x -> int
    auto y = 3.14;  // y -> double
    auto z = &x;    // z -> int*
```

### Important rules for `auto`:

```cpp
    const int ci = 10;
    auto a = ci;        // a -> int (const discarded!)
    auto& b = ci;       // b -> const int& (reference retains const)
    const auto c = ci;  // c -> const int (const explicitly added)
```

### `auto` for links and pointers:

```cpp
    int ref = x;
    auto = ref;         // d -> int (reference dropped)
    auto& e = ref;      // e -> int& (explicit reference)
```

## Literals and their types


### Integer literals:

```cpp
    42      // int
    42U     // unsigned int
    42L     // long
    42LL    // long long
```

### Real literals:

```cpp
    3.14    // double
    3.14f   // float
    3.14F   // float
    3.14l   // long double
    3.14L   // long double
```

### Character literals:

```cpp
    'a'     // char
    L'a'    // wchar_t
    u8'a'   // char (UTF-8)
    u'a'    // char16_t
    U'a'    // char32_t
```

### String literals:

```cpp
    "hello"             // const char[6]
    L"hello"            // const wchar_t[6]
    u8"hello"           // const char[6] (UTF-8)
    u"hello"            // const char16_t[6]
    U"hello"            // const char32_t[6]
    R"(raw \n string)"  // raw string literal - does not escape characters [citation:6]
```
---