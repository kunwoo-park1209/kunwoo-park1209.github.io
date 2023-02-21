---
title : "[Effective C++] 1. Accustoming Yourself to C++"
categories:
  - Effective C++
tags:
  - [c++]

toc: true
toc_sticky: true

date: 2023-02-12
last_modified_at: 2023-02-21
---

## Item 1: View C++ as a federation of languages.

C++ should be viewed as a collection of various smaller languages, each contributing its own unique features and capabilities to the overall language. These smaller languages can be grouped into categories such as the C language, object-oriented C++, template C++, and the Standard Template Library (STL). By recognizing this federated structure of the language, developers can make more effective use of C++ by leveraging the strengths of each individual language component.

## Item 2: Prefer `const`s, `enum`s, and `inline`s to `#define`s.

When writing C++ code, it is best to use the keywords `const`, `enum`, and `inline` instead of the preprocessor directive `#define`. The reason for this is that `const`, `enum`, and `inline` are type-safe, scoped, and integrated with the rest of the language, while `#define` is not type-safe, not scoped, and operates at a lower level than the rest of the language. The use of `const`, `enum`, and `inline` results in more readable, maintainable, and error-free code.

```c++
#define CALL_WITH_MAX(a, b) f( (a) > (b) ? (a) : (b) )

int a = 5, b = 0;
CALL_WITH_MAX(++a, b);    // a is incremented twice
CALL_WITH_MAX(++a, b+10); // a is incremented once

template <typename T>
inline void CallWithMax(const T& a, const T& b)
{
  f(a > b ? a : b);
}
```

## Item 3: Use `const` whenever possible.

One should use the `const` keyword whenever possible. The `const` keyword is used to specify that an object or a variable's value will not change after it has been initialized. This ensures that the value remains constant throughout the program's execution, providing stronger type checking and improved code maintainability. By using `const` whenever possible, the programmer can avoid unintended changes to the value of an object, reducing the likelihood of bugs and making the code easier to understand and maintain. 

```c++
char greeting[] = "Hello";
char *p = greeting;         // non-const pointer, non-const data
const char *p = greeting;   // non-const pointer, const data
char * const p = greeting;  // const pointer, non-const data
const char * const p = greeting; // const pointer, const data

std::vector<int> vec;

// iter acts like a T* const
const std::vector<int>::iterator iter = vec.begin();
*iter = 10;   // OK
++iter;       // ERROR

// iter acts like a const T*
std::vector<int>::const_iterator cIter = vec.begin();
*cIter = 10;  // ERROR
++cIter;      // OK
```

## Item 4: Make sure that objects are initialized before they're used

It is important to initialize objects before they are used in your program. This is because uninitialized objects may contain unpredictable values and can cause unexpected behavior in your code. Initializing objects can help prevent bugs and improve the overall reliability of your program. It is recommended to initialize objects in their constructor or by using an initializer list, rather than relying on default initialization or assignment.

```c++
class UseInitializationList
{
  public:
    UseInitializationList(const std::string& name, const std::list<PhoneNumber> phones);

  private:
    std::string _name;
    std::list<PhoneNumber> _phones;
    int _nums;
}

/* These are all assignments, not initializations */
UseInitializationList::UseInitializationList(const std::string& name, const std::list<PhoneNumber> phones)
{
  _name = name;
  _phones = phones;
  _nums = 0;
}

/* Call member variable's constructor */
UseInitializationList(const std::string& name, const std::list<PhoneNumber> phones)
: _name(name),
  _phones(phones),
  _nums(0)
{}
```

## References

[1] Scott Meyers - Effective C++ 3rd edition