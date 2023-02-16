---
title : "[Effective C++] 2. Constructors, Destructors, and Assignment Operators"
categories:
  - Effective C++
tags:
  - [c++]

toc: true
toc_sticky: true

date: 2023-02-12
last_modified_at: 2023-02-16
---

## Item 5: Know what functions C++ silently writes and calls.

It's important to understand what functions C++ generates and calls behind the scenes, such as default constructor, copy constructor, copy assignment operator, and destructor. If these functions are not defined explicitly, the compiler will generate default versions. Therefore, it is important to understand the behavior of these functions to avoid unintended behavior and improve efficiency.

## Item 6: Explicitly disallow the use of compiler-generated functions you do not want.

Be aware of and control the functions that the C++ compiler generates automatically, such as the copy constructor, assignment operator, and destructor. If there are specific instances where you do not want the compiler to generate these functions, you should explicitly disallow them by declaring them as private and not providing an implementation. This helps prevent unintended behavior and can improve code quality.

## Item 7: Declare destructors virtual in polymorphic base classes.

It is important to make the destructors of base classes virtual when the base class is meant to be used as a polymorphic type. When a polymorphic object is deleted through a pointer to its base class and the base class destructor is not virtual, the derived class's destructor will not be called and this may result in memory leaks or other resource leaks. To avoid this, the base class destructor should be declared virtual to ensure the derived class destructor is called when the polymorphic object is deleted.

## Item 8: Prevent exceptions from leaving destructors.

It is dangerous to let exceptions propagate out of destructors as it can cause resources to not be freed and lead to resource leaks or other unexpected behavior. To avoid this, the author suggests catching and handling exceptions within destructors to ensure proper resource deallocation and program termination.

## Item 9: Never call virtual functions during construction or destruction.

An unexpected behavior can occur when calling virtual functions during the construction or destruction of an object. This is because the object's state may not yet be fully established or may already be partially destroyed, leading to incorrect results or undefined behavior. To avoid these issues, the author recommends not calling virtual functions during construction or destruction, and instead relying on non-virtual functions or using other techniques such as virtual base classes.

## Item 10: Having assignment operators return a reference to `*this`.

Returning a reference to `*this` allows assignment operators to be used in chaining expressions, leading to more readable and concise code. For example, if the assignment operator of a class returns a reference to *this, you can write code like `object1 = object2 = object3`, instead of having to write `object3 = object2; object1 = object3;`. Returning a reference to `*this` also allows for more efficient code because it eliminates the need for an extra copy of the object being assigned.

## Item 11: Handle assignment to self in `operator=`.

It is important to handle self-assignment properly in the assignment operator (`operator=`) of a class. Self-assignment can occur when an object is assigned to itself, such as when `x = x`, and can cause unintended side-effects. The author suggests checking for self-assignment at the beginning of the assignment operator by comparing the address of the object being assigned with the address of the object being assigned to, and returning the object without making any changes if they are the same. This ensures that self-assignment does not cause any errors or unexpected behavior.

```c++
class Widget
{
 ...
 private:
  Bitmap* pb;
};

// self-assignment-unsafe + exception-unsafe
Widget& Widget::operator=(const Widget& rhs) {
  delete pb;
  pb = new Bitmap(*rhs.pb);

  return *this;
}

// self-assignment-safe + exception-safe
Widget& Widget::operator=(const Widget& rhs) {
  Bitmap* pOrig = pb;
  pb = newBitmap(*rhs.pb);
  delete pOrig;

  return *this;
}
```

## Item 12: Copy all parts of an object.

It is important to create proper copy constructors and assignment operators. When defining a custom class or structure, it is necessary to ensure that all parts of the object are copied correctly when the object is copied. If the object contains pointers or dynamically allocated memory, these must also be correctly copied, otherwise the copy of the object will not have the desired behavior. This item reminds programmers to carefully consider the behavior of their objects when they are copied, and to make sure they are properly handled.

```c++
class Customer
{
  public:
    ...
    Customer(const Customer& rhs);
    Customer& operator=(const Customer& rhs);
    ...
    
  private:
    std::string name;
};

// Copy Constructor
Customer::Customer(const Customer& rhs)
: name(rhs.name) {}

// Copy Assignment Operator
Customer& Customer::operator=(const Customer& rhs)
{
  name = rhs.name;
  return *this;
}

class PriorityCustomer : public Customer
{
  public:
    ...
    PriorityCustomer(const PriorityCustomer& rhs);
    PriorityCustomer& operator=(const PriorityCustomer& rhs);
    ...
    
  private:
    int priority;
};

// Copy Constructor
PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs)
: Customer(rhs), priority(rhs.priority) {}

// Copy Assignment Operator
PriorityCustomer& PriorityCustomer::operator=(const PriorityCustomer& rhs)
{
  Customer::operator=(rhs);
  priority = rhs.priority;
  return *this;
}
```

## References

[1] Scott Meyers - Effective C++ 3rd edition