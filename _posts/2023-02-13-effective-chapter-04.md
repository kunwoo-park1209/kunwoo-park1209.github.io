---
title : "[Effective C++] 4. Designs and Declarations"
categories:
  - Effective C++
tags:
  - [c++]

toc: true
toc_sticky: true

date: 2023-02-13
last_modified_at: 2023-02-18
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

## Item 19: Treat class design as type design.

The author emphasizes the importance of treating class design as type design, and to use public inheritance as a means of expressing is-a relationships between types. The author encourages using composition as a means of code reuse and to implement has-a relationships between types.

The author also advises against inheriting implementation, as doing so can lead to problems with maintenance, efficiency, and robustness. Additionally, the author advises against deriving from classes that have no virtual destructor and against inheriting from classes that are not designed to be base classes.

Overall, the item stresses that designing classes correctly is crucial to writing effective C++ code.

## Item 20: Prefer pass-by-reference-to-`const` to pass-by-value.

The author recommends preferring pass-by-reference-to-`const` to pass-by-value for function parameters. The main reason for this is to avoid the overhead of copying objects when they are passed as arguments to functions. When an object is passed by value, a copy of the object is created, which can be expensive for large objects or containers.

In contrast, pass-by-reference-to-`const` avoids the overhead of copying the object while allowing the function to read the object's data without modifying it. This technique is particularly useful for large objects, such as containers or user-defined types, which may have expensive copy constructors or assignment operators.

Additionally, the author notes that for built-in types or small user-defined types, it may be more efficient to pass by value instead of by reference, especially if the function does not modify the argument.

Overall, the author recommends carefully considering the trade-offs between pass-by-value and pass-by-reference-to-`const` for each function parameter, taking into account the size and complexity of the object being passed, as well as the intended usage of the parameter within the function.

```c++
class Window
{
  public:
    virtual void display() const {
      std::cout << "Window" << std::endl;
    }
};

class WindowWithScrollBars: public Window
{
  public:
    virtual void display() const {
      std::cout << "WindowWithScrollBars" << std::endl;
    }
};

/* 1. Pass-by-value */
void printNameAndDisplay(Window w)
{
  w.display(); /* Output: Window */
}

/* 2. Pass-by-value-to-const */
void printNameAndDisplay(const Window &w)
{
  w.display(); /* Output: WindowWithScrollBars */
}

int main() {
  WindowWithScrollBars wwsb;
  printNameAndDisplay(wwsb);
  return 0;
}
```

## Item 21: Don't try to return a reference when you must return an object.

The author advises against trying to return a reference when you must return an object. Returning a reference is usually more efficient than returning an object, but there are situations where it is impossible or dangerous to do so. For example, if you want to return a reference to a local object, that object will be destroyed when the function ends, and the reference will be invalid.

The item also explains that returning an object by value can be optimized through the use of return value optimization (RVO) or move semantics, which avoid the creation of unnecessary copies of the object.

In summary, the item suggests that you should return objects by value when appropriate and use references only when necessary or safe to do so.

```c++
class Rational {
 public:
  Rational(int numerator = 0, int denominator = 1) : n(numerator), d(denominator) {}
  
 private:
  int n, d;
 
 friend const Rational& operator*(const Rational& lhs, const Rational& rhs);
 friend bool operator==(const Rational& lhs, const Rational& rhs);
};

/* Problem 1: Returning a reference to a local object is broken */
const Rational& operator*(const Rational& lhs, const Rational& rhs)
{
  Rational result(lhs.n * rhs.n, lhs.d * rhs.d);
  return result;
}

int main() {
  Rational a(1, 2);
  Rational b(3, 5);
  Rational c = a * b;
  return 0;
}

/* Problem 2: Memory leak */
const Rational& operator*(const Rational& lhs, const Rational& rhs)
{
  Rational *result = new Rational(lhs.n * rhs.n, lhs.d * rhs.d);
  return *result;
}

int main() {
  Rational a(1, 2);
  Rational b(3, 5);
  Rational c = a * b;
  return 0;
}

/* Problem 3: Unexpected result */
bool operator==(const Rational& lhs, const Rational& rhs) {
  return lhs.n == rhs.n && lhs.d == rhs.d;
}

const Rational& operator*(const Rational& lhs, const Rational& rhs)
{
  static Rational result;
  result = Rational(lhs.n * rhs.n, lhs.d * rhs.d);
  return result;
}

int main() {
  Rational a(1, 2);
  Rational b(3, 4);
  Rational c(5, 6);
  Rational d(7, 8);
  std::cout << ((a * b) == (c * d)) << std::endl; /* Always return true */
}

/* Better write a function that returns a new object */
const Rational operator*(const Rational& lhs, const Rational& rhs)
{
  return Rational(lhs.n * rhs.n, lhs.d * rhs.d);
}
```

## Item 22: Declare data members `private`.

Data members of a class should be declared as private to prevent the direct access to the internal data of a class by outsiders. This ensures the encapsulation of the class and promotes good object-oriented programming practices. By making the data members private, we can control their access and modification through member functions of the class, which enables us to hide the implementation details and provide an abstract interface for the user.

Declaring data members as private also allows us to change the internal representation of the class without affecting the users of the class. Furthermore, it enables us to add new functionality without breaking the existing code. Therefore, making the data members private provides a level of flexibility and modularity to our code.

## Item 23: Prefer non-member non-friend functions to member functions.

The author recommends that non-member non-friend functions be preferred to member functions for better encapsulation, flexibility and efficiency. Non-member functions improve encapsulation by preventing access to the internals of a class, which means that member variables are protected and can only be accessed through public member functions. They also make the interface more flexible by allowing functions to be added more easily without modifying the class itself. In addition, non-member functions can be more efficient, as they don't require implicit this pointer parameter, and can often be inlined by the compiler. The non-member functions can still be granted access to private members of the class by being declared as friends, and are used for operator overloading and type conversions.

```c++
class WebBrowser
{
  public:
    void clearCache();
    void clearHistory();
    void removeCookies();
    
    // 1. member function (not preferred!)
    void clearEverything() {
      clearCache();
      clearHistory();
      removeCookies();
    }
};

// 2. non-member function (preferred!)
void clearBrowser(WebBrowser& wb)
{
  wb.clearCache();
  wb.clearHistory();
  wb.removeCookies();
}
```

## Item 24: Declare non-member functions when type conversions should apply to all parameters.

You should declare non-member functions that act on all parameters using implicit conversions instead of declaring them as member functions. This is because non-member functions allow conversions to be applied to all parameters, not just the implicit object parameter. This helps to improve encapsulation by keeping the number of functions with access to a class's private parts to a minimum, while also enabling extensibility, and enhancing flexibility.

```c++
class Rational
{
  public:
    Rational(int numerator = 0, int denominator = 1);
    int numerator() const;
    int denominator() const;
    const Rational operator*(const Rational& rhs) const;
  private:
    int n;
    int d;
};

int Rational::numerator() const
{
  return n;
}

int Rational::denominator() const
{
  return d;
}

Rational::Rational(int numerator, int denominator)
{
  n = numerator;
  d = denominator;
}

const Rational Rational::operator*(const Rational& rhs) const
{
  return Rational(this->n*rhs.numerator(), this->d*rhs.denominator());
}

void printRational(const Rational& r)
{
  std::cout << r.numerator() << " / " << r.denominator() << std::endl;
}


int main()
{
  Rational oneHalf(1, 2);
  
  Rational result = oneHalf * 2;
  /* The following fails! 
  Rational result = 2 * oneHalf;
  */ 
  printRational(result); /* 2 / 2 */

  return 0;
}

/* Make operator* a non-member function, allowing compilers to perform implicit type
 * conversions on all arguments */
const Rational operator *(const Rational& lhs, const Rational& rhs) const
{
  return Rational(lhs.numerator() * rhs.numerator(),
                  lhs.denominator() * rhs.denominator());
}
```

## Item 25: Consider support for a non-throwing `swap`.

The `swap` function is commonly used in sorting and searching algorithms, and its performance can be improved by using a non-throwing `swap`. This is achieved by creating a specialized `swap` function for a particular class that swaps the internal data members using the no-throw guarantee. Additionally, `swap` can be useful for implementing the copy-and-swap idiom, which makes it easier to write correct and exception-safe copy assignment operators. By providing a non-throwing `swap` function, classes can improve their performance and provide better support for the standard library algorithms.

```c++
/* Typical implementation of std::swap
 * However, it involves copying three objects which may be costly for swapping large objects */
namespace std
{
  template<typename T>
  void swap(T& a, T& b)
  {
    T temp(a);
    a = b;
    b = temp;
  }
}

/* Specialize std::swap for Widget class */
class Widget
{
  public:
    void swap(Widget& other)
    {
      using std::swap; /* if declared, the compiler tries to use std::swap first */
      swap(member, other.member);
    }
  
  private:
    int *member;
};

namespace std
{
  template<>
  void swap<Widget>(Widget& a, Widget& b)
  {
    a.swap(b);
  }
}
```

## References

[1] Scott Meyers - Effective C++ 3rd edition