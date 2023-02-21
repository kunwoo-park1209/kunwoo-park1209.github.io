---
title : "[Effective C++] 5. Implementations"
categories:
  - Effective C++
tags:
  - [c++]

toc: true
toc_sticky: true

date: 2023-02-18
last_modified_at: 2023-02-18
---

## Item 26: Postpone variable definitions as long as possible.

The author suggests to postpone variable definitions as long as possible. This practice helps minimize dependencies between different modules in a program and thereby can reduce coupling. In C++, the order of object construction and destruction is important. Therefore, defining an object before another object that uses it in the constructor will cause undefined behavior. If variable definitions are postponed to the last moment, it is less likely that they will be used before they are fully initialized. This is particularly important for global or static objects. Thus, waiting to define a variable until its value is actually needed can help avoid some subtle bugs in C++ programs.

```c++
/* The string "encrypted" is unused but constructed if an exception is thrown */
std::string encryptPassword(const std::string& password)
{
  using namespace std;
  string encrypted(password);
  
  if (password.length() < MinimumPasswordLength) {
    throw logic_error("Password is too short");
  }

  ...
  
  return encrypted;
}

/* Postpone "encrypted"'s definition until you know you'll need it */
std::string encryptPassword(const std::string& password)
{
  using namespace std;
  
  if (password.length() < MinimumPasswordLength) {
    throw logic_error("Password is too short");
  }

  string encrypted(password);
  
  ...

  return encrypted;
}

```

## References

[1] Scott Meyers - Effective C++ 3rd edition

