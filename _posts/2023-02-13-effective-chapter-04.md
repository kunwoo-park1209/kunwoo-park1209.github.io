---
title : "[Effective C++] 4. Designs and Declarations"
categories:
  - Effective C++
tags:
  - [c++]

toc: true
toc_sticky: true

date: 2023-02-13
last_modified_at: 2023-02-14
---

## Item 18: Make interfaces easy to use correctly and hard to use incorrectly.

It is important to design user-friendly and error-proof interfaces for classes and functions. This means that the interfaces should be intuitive and straightforward to use, while also being designed in a way that minimizes the likelihood of mistakes or unintended consequences. This is achieved by providing clear documentation, using clear and consistent naming conventions, and providing adequate error checking and handling mechanisms. The goal is to make it as easy as possible for users to use the interfaces correctly, while making it difficult or impossible to use them incorrectly.

```c++
/* The following interface has at least two errors that clients might easily make. */

class Date {
 public:
  Date(int Month, int day, int year);
 ...
};

Date d(30, 3, 1995); // Oops! Should be "3, 30", not "30, 3"
Date d(3, 40, 1995); // Oops! Should be "3, 30", not "3, 40"

/* Many client errors can be prevented by the introduction of new types and 
 * restricting the values of those types. */

class Month {
 public:
  static Month Jan() { return Month(1); }
  static Month Feb() { return Month(2); }
  ...
  static Month Dec() { return Month(12); }
 
 private:
  explicit Month(int m);
  ...
};

struct Day {
 explicit Day(int m) : val(m) {}

 int val;
};

struct Year {
 explicit Year(int y) : val(y) {}

 int val;
};

class Date {
 public:
  Date(const Month& m, const Day& d, const Year& y);
  ...
};

Date d(Month::Mar(), Day(30), Year(1995));
```

```c++
/* Any interface that requires that clients remember to deallocate dynamically
 * allocated object is prone to incorrect use, because clients can forget to
 * do it. */

Investment *createInvestment();

/* A better interface decision would be to preempt the problem by returning a
 * smart pointer in the first place. */

std::shared_ptr<Investment> createInvestment() {
    std::shared_ptr<Investment> retVal(static_cast<Investment *>(0), getRidOfInvestment);
    ...
    return retVal; 
}
```

## References

[1] Scott Meyers - Effective C++ 3rd edition