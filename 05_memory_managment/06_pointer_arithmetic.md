# Pointer arithmetic and low-level access

## Pointer

**Pointer** (указатель) — это переменная, содержащая **typed memory address** (типизированный адрес). Тип указателя сообщает компилятору, сколько данных находится по этому адресу и как их интерпретировать.

```cpp
    int value = 42;
    int* ptr = &value;              // ptr stores the address of 'value'

    std::cout << ptr << "\n";       // prints the address. For example: 0x7ffd3b2a1e4c
    std::cout << *ptr << "\n";      // dereferencing operator — prints 42
```

Оператор ` & ` (addres-of operator) возвращает указатель на переменную. Оператор ` * ` (dereferencing operator) обращается к значению, хранящемуся по адресу, на который указывает указатель.

В 64-битной системе указатель всегда занимает 8 байт, независимо от типа, на который он указывает. Типы ` int* `, ` double* ` и ` Connexion* ` имеют одинаковый размер, различается только их интерпретация.

**Basic pointer operators**

- ` & ` (address-of operator) - получить адрес объекта
- ` * ` (dereferencing operator) - разыменовать указатель

---

## Basic pointer arithmetic

При арифметических операциях, указатель **смещается на размер елемента на который указывает**. Другими словами, указатель сместится на ` sizeof(pointed_type) ` байт.

Смещение вперед:

```cpp
    int array[] = {10, 20, 30, 40, 50};
    int* p = array;                     // points to array[0]. Equivalent to: int* p = &array[0];

    std::cout << *p << '\n';            // 10
    std::cout << *(p + 1) << '\n';      // 20 — advances by sizeof(int) = 4 bytes
    std::cout << *(p + 2) << '\n';      // 30 — advances by 2 × 4 = 8 bytes
    std::cout << *(p + 4) << '\n';      // 50 — advances by 4 × 4 = 16 bytes
```

Смещение в памяти, если предположить что ` array ` начинается с указанного адреса ` 0x1000 `:

```text
    Address :   0x1000   0x1004   0x1008   0x100C   0x1010
    Value   :   [ 10 ]   [ 20 ]   [ 30 ]   [ 40 ]   [ 50 ]
    Pointer :     p        p+1      p+2      p+3      p+4
```

Компилятор преобразует ` p + n ` в ` address-of_p + n × sizeof(int) `. Это автоматическое преобразование обеспечивает согласованность арифметики указателей с типами.

Смещение назад работает аналогично: ` p - 1 ` сдвигает указатель на один элемент назад:

```cpp
    int array[] = {10, 20, 30, 40, 50};

    int* begin = &array[0];
    int* end   = &array[4];

    std::ptrdiff_t distance = end - begin;

    std::cout << distance << "\n";   // 4
```

При использовании типов с большм размером, изменяется шаг смещения:

```cpp
    double array[] = {1.1, 2.2, 3.3};
    double* d = array;

    // d + 1 advances by sizeof(double) = 8 bytes

    std::cout << *d << "\n";       // 1.1
    std::cout << *(d + 1) << "\n"; // 2.2
    std::cout << *(d + 2) << "\n"; // 3.3
```

---

## Equivalence of arrays and pointers

В большинстве случаев **массив неявно преобразуется в указатель** (array-to-pointer decay) на первый элемент. Это свойство создает синтаксическую эквивалентность между индексированным доступом и арифметикой указателей:

```cpp
    int array[] = {100, 200, 300};

    // These four notations are equivalent:

    array[2];       // access by index — the most readable
    *(array + 2);   // explicit pointer arithmetic
    *(2 + array);   // addition is commutative
```

Компилятор выполняет трансляцию ` array[i] ` в ` *(array + i) ` во всех случаях. Благодаря этой эквивалентности указатели можно использовать точно так же, как и массивы:

```cpp
    void print(const int* data, int size)
    {
        for (int i = 0; i < size; ++i)
        {
            std::cout << data[i] << " ";
        }
        std::cout << "\n";
    }

    int main()
    {
        int values[] = {1, 2, 3, 4, 5};

        print(values, 5);

        int* heap_values = new int[5]{10, 20, 30, 40, 50};

        print(heap_values, 5);

        delete[] heap_values;
    }
```

Пошаговое перемещение:

Операторы ` ++ ` и ` -- ` применяются к указателям и позволяют перемещаться вперед или назад на один элемент:

```cpp
    int array[] = {5, 10, 15, 20, 25};

    int* p   = array;
    int* end = array + 5;   // points one element past the end

    while (p != end)
    {
        std::cout << *p << " ";
        ++p;
    }

    // Output: 5 10 15 20 25
```

> ⚠️ **Заметка:** ` sizeof(array) ` возвращает **общий размер массива** в байтах, в то время как ` sizeof(pointer-to_array) ` обычно **возвращает 8 байт** на 64 битных системах. Как только массив преобразуется в указатель, информация о его размере теряется. Поэтому функциям принимающим массив, всегда требуется отдельный параметр указывающий на размер массива.

---

## Pointers to structures

Если указатель указывает на объект, структуру или класс, доступ к его членам осуществляется с помощью оператора ` -> `, который объединяет разыменование и доступ к членам:

```cpp
    struct Point
    {
        double x;
        double y;
    };

    Point* p = new Point{3.0, 4.0};

    // These two notations are equivalent:

    std::cout << (*p).x << "\n";    // dereferencing then member access
    std::cout << p->x << "\n";      // shortened syntax

    // Arithmetic on an array of structures

    Point points[] = {{1.0, 2.0}, {3.0, 4.0}, {5.0, 6.0}};

    Point* cursor = points;

    std::cout << cursor->x << "\n"; // 1.0
    std::cout << (cursor + 1)->y << "\n"; // 4.0
    std::cout << (cursor + 2)->x << "\n"; // 5.0

    delete p;
```

Арифметические операции выполняются аналогично: ` cursor + 1 ` продвижение на ` sizeof(Point) ` байт, соответствует следующему элементу массива.

---

## Универсальный указатель

Тип ` void* ` представляет собой указатель **без связанного с ним типа**. Он может содержать адрес любого объекта, но его нельзя напрямую разыменовать или использовать для арифметических операций. Компилятор не знает, на сколько байт его нужно сдвинуть:

```cpp
    int int_value = 42;
    double double_value = 3.14;

    void* generic = &int_value;     // any address can be stored

    // *generic;                   // error — void* cannot be dereferenced
    // generic + 1;                // error — arithmetic cannot be performed on void*


    // To use a void*, it must be cast to a concrete type

    int* p = static_cast<int*>(generic);

    std::cout << *p << "\n";                    // 42

    generic = &double_value;                   // type change

    double* d = static_cast<double*>(generic);

    std::cout << *d << "\n";                    // 3.14
```

> ⚠️ **Заметка:** Если у указателя ` void* ` тип ` int* `, попытка привести его к типу ` double* ` справоцирует неопределенное поведение или сбой, так как процесс разыменования прочитает 8 байт, из которых допустимы только 4.

---

## Modern null pointer

До C++11 нулевой указатель представлялся NULL макросом, определенным как 0 ` null ` или литерал 0. В C++11 был введен ` nullptr `, литерал типа ` std::nullptr_t `, который может быть преобразован только в указатель:

```cpp
    int* p1 = nullptr;  // Modern C++ - unambiguous
    int* p2 = NULL;     // Legacy - NULL is often defined as 0
    int* p3 = 0;        // Compiles but ambiguous — is it an integer or a pointer?
```

Разница не носит чисто косметический характер. Она устраняет реальную неоднозначность при перегрузке функций:

```cpp
    void process(int value)
    {
        std::cout << "integer\n";
    }

    void process(int* pointer)
    {
        std::cout << "pointer\n";
    }

    process(NULL);      // ambiguous — NULL is 0, which is an int → calls process(int) 
    process(nullptr);   // unambiguous → calls process(int*)
```

Разыменование ` nullptr ` приведет к неопределённому поведению, которое обычно ведет к ` Segmentation fault `:

```cpp
    int* p = nullptr;

    *p = 10;          // Segmentation fault (or worse: silent corruption)
```

Наилучшей практикой является постоянная проверка того, что указатель не равен ` null `, перед его разыменованием, если его значение не гарантируется контекстом:

```cpp
    void utiliser(int* p)
    {
        if (p != nullptr)   // ou simplement : if (p)
        {
            std::cout << *p << "\n";
        }
    }
```

---

## Pointers and constants

Взаимодействие между указателями ` const ` часто является источником путаницы. Существует четыре комбинации, каждая из которых имеет свою собственную семантику:

```cpp
    int value = 10;
    int other = 20;

    // 1. Modifiable pointer to modifiable data

    int* p1 = &value;

    *p1 = 30;                   // Modify the data
    p1 = &other;                // Reassign the pointer


    // 2. Modifiable pointer to constant data

    const int* p2 = &value;     // or: int const* p2

    *p2 = 30;                   // Cannot modify the data via p2
    p2 = &other;                // Can reassign the pointer


    // 3. Constant pointer to modifiable data

    int* const p3 = &value;

    *p3 = 30;                   // Can modify the data
    p3 = &other;                // Cannot reassign the pointer


    // 4. Constant pointer to constant data

    const int* const p4 = &value;

    // *p4 = 30;        // error
    // p4 = &other;     // error
```

Правило чтения: ` const ` **применяется к тому, что находится непосредственно слева от него** (за исключением крайнего левого края, в таком случае применяется к тому, что находится справа от него).

```cpp
    const int* p        // pointer to const int
    int const* p        // same thing
    int* const p        // const pointer to int
    const int* const p  // const pointer to const int
```

Форма ` const int* ` (указатель на константные данные) является наиболее распространенной. Именно эта форма используется когда функция обещает не изменять переданные ей данные:

```cpp
    // The `const` guarantees that the function will not modify the contents of the array

    double average(const double* data, int size)
    {
        double sum = 0.0;

        for (int i = 0; i < size; ++i)
        {
            sum += data[i];

        }

        return sum / size;
    }
```

## Limits of arithmetic

Арифметика указателей допустима только **в пределах массива** *(плюс один элемент за пределами его конца )*. Любой доступ за эти границы является неопределенным поведением, даже если вычисленный адрес соответствует допустимой области памяти.

```cpp
    int array[5] = {10, 20, 30, 40, 50};

    int* p = array;

    *(p + 4);       // last element (array[4])
    *(p + 5);       // undefined behavior — one beyond the past-the-end
    *(p - 1);       // undefined behavior — before the beginning of the array
```

Указатель, следующий за концом строки ( p + 5в этом примере), **действителен для сравнения** , но **не для разыменования**:

```cpp
    int array[5] = {10, 20, 30, 40, 50};

    int* end = array + 5;

    // Comparison with past-the-end — basis of all traversal

    for (int* it = array; it != end; ++it)
    {
        std::cout << *it << " ";
    }

    
    std::cout << *end; // Dereferencing past-the-end — undefined behavior
```

Аналогично, арифметические операции между указателями, которые не указывают на один и тот же массив, являются неопределенным поведением:

```cpp
    int a[10];
    int b[10];

    int* pa = a + 5;
    int* pb = b + 3;

    ptrdiff_t diff = pa - pb;   // undefined behavior — different arrays
```

Эти правила могут показаться ограничительными, но они позволяют компилятору выполнять агрессивные оптимизации. На практике, при использовании контейнеров и итераторов **STL**, эти ограничения обрабатываются автоматически.

---