+++
title = "Back to Basics: Shared Pointers"
date = 2022-09-18
draft = false

[taxonomies]
categories = ["Fundamentals"]
tags = ["c++","rust"]

[extra]
lang = "en"
toc = true
show_comment = true
math = false
mermaid = false
cc_license = true
outdate_warn = true
outdate_warn_days = 120
+++

## Introduction
In C++ like in other languages like Rust is very important the concept of data ownership.
In this post, I will show how to create a class that implements **shared pointer** in C++.
Once in a C++ interview I was asked the internal implementation of shared_ptr, apparently that is the line that separates junior C++ engineers and senior C++ engineers. 

So today you will learn how to become a senior C++ engineer :). My recommendation is to avoid your implementation of shared pointer and use the standard implementation that you can find in the standard library in the <span style="font-family:Courier New; font-size:1em;">memory</span> header with name <span style="font-family:Courier New; font-size:1em;">std::shared_ptr</span>.

## Smart pointers
> Smart pointers are constructs that allow to have a unambiguous definition of ownership.
> The shared_ptr type is a smart pointer in the C++ standard library that is designed for scenarios in which more than one owner might have to manage the lifetime of the object in memory.

The mechanism that is used to implement shared pointers is **reference counting**, which consists in having a counter to keep track on how many objects are pointing to the data contained in the shared pointer.

In this case the lifetime of the object is shared between all the objects that copy the shared pointer. 

Here we can see an example, imagine that you have the following struct:

```cpp
struct A
{
    int element1;
    int element2;
};
```

I assume that on my machine int is represented on 4 bytes. The size of this struct is then 8 bytes.
Let's start by using what is available in the standard library: <span style="font-family:Courier New; font-size:1em;">std::shared_ptr</span>.

The following snippet of code will use a <span style="font-family:Courier New; font-size:1em;">std::shared_ptr</span> on struct A.

```cpp
#include <iostream>
#include <memory>

struct A
{
    A() = default;

    int element1 = 10;
    int element2 = 10;
};

int main()
{
    auto p = std::make_shared<A>(); // Position 1
    std::cout << p->element1 << "," << p->element2 << std::endl;
    std::cout << "Count: " << p.use_count() << std::endl; 
    auto p2 = p;                    // Position 2
    std::cout << "Count: " << p.use_count() << std::endl; 
    {
        auto p3 = p; // Position 3
        std::cout << "Count: " << p.use_count() << std::endl;
    }
    std::cout << "Count: " << p.use_count() << std::endl; // Position 4
    return 0;
}
```

Let's briefly analyze this code, in position (1) we create the shared pointer and the reference count goes to 1. After in position (2) we copy p into p2, which means that the reference count increases and the from 1 goes to 2.
At position (3) we perform another assignment operator which increments the reference count which goes from 2 to 3.
Finally, at position (4) the <span style="font-family:Courier New; font-size:1em;">p3</span>
goes out of scope and the counter goes back to 2. When <span style="font-family:Courier New; font-size:1em;">main</span> finishes its execution.

What happens in memory is the following:
- at position (1) the control block and the object are created, so the reference count is one.
{{ figure(src="/img/shared_ptr/shared_ptr1.jpg", alt="First call to make_shared", caption="") }}
- at position (2) the assignment operator is called, which acts as the copy constructor. Now p and p2 will have a pointer to have and to the control block.
{{ figure(src="/img/shared_ptr/shared_ptr2.jpg", alt="", caption="") }}

This behavior translates in C++ with the following code.

```cpp
template <typename T>
class SharedPtr
{
public:
    template <typename... Args>
    explicit SharedPtr(Args&&... args) : m_ptr(new T(std::forward<Args...>(args...))), 
                                         m_count(new size_t(1)) {}

    // Default constructor
    SharedPtr() = default;

    ...

private:
    size_t* m_count = nullptr; // (1) Control block pointer
    T* m_ptr = nullptr;        // (2) Data pointer
};
```

We have two pointers one to the object that we want to share <span style="font-family:Courier New; font-size:1em;">T*</span> and the other to the control block <span style="font-family:Courier New; font-size:1em;">m_count</span>, which simply contains the counter that can be reset and incremented.

### Assignment and Copy
The copy contructor and assignment operator are quite straightforward. We need simply to copy the pointers and then increment the counter. This means that the new shared pointer will point to the same control block and to the same data of the copied instance.

```cpp
template <typename T>
class SharedPtr
{
public:
    ...

    SharedPtr(const SharedPtr<T>& other)
    {
        m_ptr = other.m_ptr;
        m_count = other.m_count;
        (*m_count)++;
    }

    SharedPtr& operator=(const SharedPtr<T>& other)
    {
        // Maybe using copy and swap here
        m_ptr = other.m_ptr;
        m_count = other.m_count;
        (*m_count)++;
    }

    ...

private:
    size_t* m_count = nullptr; // (1) Control block pointer
    T* m_ptr = nullptr;        // (2) Data pointer
};
```

### Destructor
The destructor is intuitive too, when an shared pointer exits the scope what happens is that we need to reduce the counter. The last object to be deallocated will be the one that will destroy the shared resource and the control block.

Here an example of what happens on shared pointer destruction.

{{ figure(src="/img/shared_ptr/shared_ptr_destr.jpg", alt="", caption="") }}

When p2 goes out of scope the counter is decremented and the pointers are deallocated.

```cpp
template <typename T>
class SharedPtr
{
public:
    ...

    ~SharedPtr()
    {
        (*m_count)--;
        if(*m_count == 0)
        {
            delete m_count;
            delete m_ptr;
        }
    }

    ...

private:
    size_t* m_count = nullptr;
    T* m_ptr = nullptr;
};
```

### Full implementation in C++
Here you can find the full implementation in C++.
```cpp
template <typename T>
class SharedPtr
{
public:
    template <typename... Args>
    explicit SharedPtr(Args&&... args) : m_ptr(new T(std::forward<Args...>(args...))), 
                                         m_count(new size_t(1)) {}

    // Default constructor
    SharedPtr() = default;

    SharedPtr(const SharedPtr<T>& other)
    {
        m_ptr = other.m_ptr;
        m_count = other.m_count;
        (*m_count)++;
    }

    SharedPtr& operator=(const SharedPtr<T>& other)
    {
        // Maybe using copy and swap here
        m_ptr = other.m_ptr;
        m_count = other.m_count;
        (*m_count)++;
    }

    size_t use_count() const
    {
        return m_count != nullptr ? *m_count : 0;
    }

    T* get() const
    {
        return *m_ptr;
    }

    T& operator*()
    {
        return *m_ptr;
    }

    T* operator->()
    {
        return m_ptr;
    }

    ~SharedPtr()
    {
        (*m_count)--;
        if(*m_count == 0)
        {
            delete m_count;
            delete m_ptr;
        }
    }

    friend std::ostream& operator<<(std::ostream& os, const SharedPtr<T>& ptr)
    {
        os << "Address pointed is " << ptr.m_ptr << std::endl;
        os << "Value of the counter: " << *ptr.m_count << std::endl;
        return os;
    }

private:
    size_t* m_count = nullptr;
    T* m_ptr = nullptr;
};
```

## Rust
In Rust we have the standard smart pointer Rc that works similar compared to <span style="font-family:Courier New; font-size:1em;">std::shared_ptr</span>.

```rust
use std::rc::Rc;

struct A
{
    element1 : i32,
    element2 : i32,
}


pub fn main()
{
    let a = A{element1:10, element2:10};
    let b = Rc::new(a); // count = 1
    println!("b count {}", Rc::strong_count(&b));

    let c = Rc::clone(&b); // count = 2
    println!("b count {}", Rc::strong_count(&b));

    {
        let d = Rc::clone(&b); // count = 3
        println!("b count {}", Rc::strong_count(&b));
    }

    println!("b count {}", Rc::strong_count(&b)); // count = 2
}
```

## Links
[How to use shared_ptr](https://learn.microsoft.com/en-us/cpp/cpp/how-to-create-and-use-shared-ptr-instances?view=msvc-170)
<br>
[Full Implementation in C++ (Godbolt)](https://www.godbolt.org/z/ETPbT6q5n)
<br>
[Rust Example (Godbolt)](https://www.godbolt.org/z/Gf9hh543d)