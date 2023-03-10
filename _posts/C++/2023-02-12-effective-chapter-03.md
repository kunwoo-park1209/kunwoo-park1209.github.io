---
title : "[Effective C++] 3. Resource Management"
categories:
  - Effective C++
tags:
  - [c++]

toc: true
toc_sticky: true

date: 2023-02-12
last_modified_at: 2023-02-14
---

## Item 13: Use objects to manage resources.

It is recommended to use objects to manage resources, such as memory, files, and network connections. The idea is to use a class that wraps the resource and takes care of acquiring and releasing it in a controlled manner, to ensure proper behavior, exception safety, and resource management. For example, instead of manually allocating and freeing memory using `new` and `delete`, it is recommended to use smart pointers, such as `std::unique_ptr` or `std::shared_ptr`, to automatically manage the memory and prevent memory leaks.

```c++
Investment* CreateInvestment() { return new Investment(); }

// Potential memory leak
void f() {
    Investment *pInv = CreateInvestment();
    ... // ERROR occurs!!!
    delete pInv;
}

// Use smart pointers to avoid memory leak
void f() {
    std::unique_ptr<Investment> pInv(CreateInvestment());
    ...
}
```

```c++
std::unique_ptr<Investment> pInv1(CreateInvestment());
//std::unique_ptr<Investment> pInv2(pInv1); <- ERROR: pInv1 deleted after copy
std::unique_ptr<Investment> pInv2(std::move(pInv1));
if (pInv1 == nullptr) std::cout << "pInv1 is nullptr!" << std::endl;
//pInv1 = pInv2; <- ERROR!: pInv2 deleted after copy
pInv1 = std::move(pInv2);
if (pInv2 == nullptr) std::cout << "pInv2 is nullptr!" << std::endl;

/* Output:
pInv1 is nullptr!
pInv2 is nullptr!
*/
```

```c++
std::shared_ptr<Investment> pInv1(CreateInvestment());
//std::shared_ptr<Investment> pInv2(std::move(pInv1)); <- pInv1 becomes nullptr
std::shared_ptr<Investment> pInv2(pInv1);
//pInv1 = std::move(pInv2); <- pInv2 becomes nullptr
pInv1 = pInv2;
std::cout << "Reference Count of pInv1 : " << pInv1.use_count() << std::endl;
std::cout << "Reference Count of pInv2 : " << pInv2.use_count() << std::endl;

/* Output:
Reference Count of pInv1 : 2
Reference Count of pInv2 : 2
*/
```

```c++
/* Do not use smart pointers with dynamically allocated arrays because both
 * unique_ptr and shared_ptr use delete instead of delete [], which leads to
 * memory leak. */
std::unique_ptr<int> spi(new int[1024]);
std::shared_ptr<int> spi(new int[1024]);

// Removing memory leak in C++17
std::shared_ptr<int[]> spi(new int[1024]);
```

## Item 14: Think carefully about copying behavior in resource-managing classes.

When designing a resource-managing class (e.g. a class that manages memory or file handles), one should consider the behavior of the class when objects are copied, and make sure it is appropriate for the class's intended use. This could include defining a copy constructor and an assignment operator that makes a "deep" copy of the object's resource, or declaring these functions private to prevent copying and enforce resource ownership.

## Item 15: Provide access to raw resources in resource-managing classes.

It is encouraged to provide access to the raw resources managed by resource-managing classes, in a controlled and safe manner, so that the user can have full control over the resource when necessary. This can be achieved through providing accessor functions, but making sure that the accessor does not allow the user to do anything harmful, such as freeing the resource or copying it multiple times. By providing access to the raw resource, the user can make more efficient use of it and avoid unnecessary resource allocation and deallocation.

```c++
std::shared_ptr<Investment> pInv(createInvestment());
int daysHeld(const Investment *pi);

// ERROR!
int days = daysHeld(pInv);
/* Explicit Conversion (i.e., return a copy of the raw pointer inside the 
 * smart pointer object) */
int days = daysHeld(pInv.get());
/* Implicit Conversion (i.e., use operator to convert to the raw pointer) */
int days = daysHeld(*pInv);
```

## Item 16: Use the same form in corresponding uses of `new` and `delete`.

If you use a form of new (e.g. new[]) to allocate memory, you should use the corresponding form of delete (e.g. delete[]) to deallocate it. If you use the wrong form, you risk undefined behavior, including crashes and memory leaks. It's important to be consistent and use the correct form of new and delete to ensure the safe and proper allocation and deallocation of memory in your program.

```c++
/* At least 99 of the 100 string objects pointed to by stringArray are unlikely
 * to be properly destroyed */

std::string *stringArray = new std::string[100];
delete stringArray;

/* If you use [] in a new expression, you must use [] in the corresponding delete 
 * expression. If you don't use [] in a new expression, don't use [] in the 
 * matching delete expression. */

std::string *stringArray = new std::string[100];
delete [] stringArray;

typedef std::string AddressLines[4]; // AddressLines is an array of string
std::string *pal = new AddressLines;
delete [] pal;
```

## Item 17: Store `new`ed objects in smart pointers in standalone statements.

It is important to use smart pointers to manage dynamically allocated objects in C++. The author suggests that dynamically allocated objects should always be stored in smart pointers, and that these smart pointers should be declared in standalone statements (i.e., not as part of a larger expression). This helps to ensure that the dynamically allocated objects are properly managed and that their lifetime is well defined, even in the presence of exceptions or other error conditions.

```c++
/* The following function call may leak resources as follows:
 * 1. Execute "new Widget"
 * 2. Call priority() before calling the shared_ptr constructor, but yields exception */

processWidget(std::shared_ptr<Widget>(new Widget), priority());

/* The way to avoid problems is simple: use a separate statement to create
 * the Widget and store it in a smart pointer */

std::shared_ptr<Widget> pw(new Widget);
processWidget(pw, priority());
```

## References

[1] Scott Meyers - Effective C++ 3rd edition