---
title : "[Effective C++] 1. Accustoming Yourself to C++"
categories:
  - Effective C++
tags:
  - [c++]

toc: true
toc_sticky: true

date: 2023-02-12
last_modified_at: 2023-02-12
---

## Item 1: View C++ as a federation of languages.

C++ should be viewed as a collection of various smaller languages, each contributing its own unique features and capabilities to the overall language. These smaller languages can be grouped into categories such as the C language, object-oriented C++, template C++, and the Standard Template Library (STL). By recognizing this federated structure of the language, developers can make more effective use of C++ by leveraging the strengths of each individual language component.

## Item 2: Prefer `const`s, `enum`s, and `inline`s to `#define`s.

When writing C++ code, it is best to use the keywords `const`, `enum`, and `inline` instead of the preprocessor directive `#define`. The reason for this is that `const`, `enum`, and `inline` are type-safe, scoped, and integrated with the rest of the language, while `#define` is not type-safe, not scoped, and operates at a lower level than the rest of the language. The use of `const`, `enum`, and `inline` results in more readable, maintainable, and error-free code.

## Item 3: Use `const` whenever possible.

One should use the `const` keyword whenever possible. The `const` keyword is used to specify that an object or a variable's value will not change after it has been initialized. This ensures that the value remains constant throughout the program's execution, providing stronger type checking and improved code maintainability. By using `const` whenever possible, the programmer can avoid unintended changes to the value of an object, reducing the likelihood of bugs and making the code easier to understand and maintain. 

## Item 4: Make sure that objects are initialized before they're used

It is important to initialize objects before they are used in your program. This is because uninitialized objects may contain unpredictable values and can cause unexpected behavior in your code. Initializing objects can help prevent bugs and improve the overall reliability of your program. It is recommended to initialize objects in their constructor or by using an initializer list, rather than relying on default initialization or assignment.

## References

[1] Scott Meyers - Effective C++ 3rd edition