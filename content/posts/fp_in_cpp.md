+++ 
draft = false
date = 2023-05-18T11:07:05-06:00
title = "Functional Programming in C++"
description = ""
slug = ""
authors = ["Taylor Allred"]
tags = ["c++", "functional programming"]
categories = []
externalLink = ""
series = []
+++


```cpp
#include <algorithm>
#include <concepts>
#include <iostream>
#include <ranges>
#include <vector>

using namespace std::views;

int main()
{
    auto const even = [](const int i) { return 0 == i % 2; };
    auto const square = [](const int i) { return i * i; };
		
    auto vec = iota(1) // Create infinite view of numbers from 1
        | filter(even) // Filter even ones
        | transform([](auto const v) { return v * 2; }) // Transform with lambda
        | transform(square) // Transform referring to lambda value
        | take(10) // Take 10, creates view into 10 elements
        | std::ranges::to<std::vector>(); // Convert to vector, invoking operations

    std::ranges::for_each(vec, [](auto const v) { std::cout << v << ' '; });
}
```

> Above is a platter of functional programming capabilities in C++. You can see several languages features on display: const keyword, lambdas, ranges, and higher order functions. Note: *Must run with C++23, and `std::ranges::to<>()` is only available in the latest version of msvc as of this writing.*
> 

C++ is a powerful language with support for many paradigms of programming. The tools the language gives you allow you to shape your code almost however you like. In recent time, C++ has gotten increased support for functional programming. In this article, I will summarize the main points from reading the book [Functional Programming in C++](https://www.manning.com/books/functional-programming-in-c-plus-plus) in hopes that this can be good quick reference for anyone wishing to learn the topic. 

# Contents

- Pure functions
- `const` keyword
- Lambdas
- Ranges/views

# Pure Functions

A **pure function** is a function that: 

- Takes in arguments and produces a return value
- Can only refer to its arguments and cannot refer to global vars
- Does not mutate its arguments
- Does not perform side-effects (such as I/O)
- Given the same arguments, produces the same return value every time it’s called

You can use pure functions in any language but different languages encourage and facilitate them to differing degrees. Pure functions are useful because they are extremely testable, easier to reason about, and easier to debug. They also work well with higher order functions (functions that take other functions as arguments). 

# *Const* Keyword

The `const` keyword has been around for a long time and was introduced to both C and C++. It statically guarantees that the value will be **immutable**. Immutability and functional programming go hand-in-hand because we favor writing pure functions where no arguments can be mutated. 

```cpp
const int x = 1;
x = 2; // fails
```

The above code will throw an error at compile time because the variable `x` was labelled as `const`.

It gets more complicated when using pointers. 

```cpp
const int x = 1;
int* y = &x; // fails
```

This code will fail because a `int*` cannot refer to a `const int*`. 

```cpp
const int x = 1;
const int* y = &x;
*y = 3; // fails
```

This code fails because `y` refers to an immutable int and therefore cannot change its value. 

```cpp
const int x = 1;
const int z = 2;
const int* y = &x;
y = &z; // works!
```

This code *does* work because while all the `int`s defined are immutable the pointer `y` is *not* immutable so it can be changed to point to something else. 

```cpp
const int x = 1;
const int z = 2;
const int* const y = &x;
y = &z; // fails
```

Now this code fails because `y` is labeled as a “constant pointer to a constant int”. 

As verbose as it may be, it might be worth using `const` keyword in every situation *unless* you know specifically that you intend to mutate a variable. When you think that a variable might not need to be const, it's a good time to think hard about whether or not it needs to be mutated and if the algorithm calls for it. 

This will help you avoid the bugs caused by something changing when it wasn’t supposed to. Also, it will reduce cognitive load to know that a variable will refer to the same value throughout its lifetime. 

[https://en.cppreference.com/w/cpp/language/cv](https://en.cppreference.com/w/cpp/language/cv)

# Lambdas

A “lambda” is simply a name for an “anonymous function” (a function with no name). Languages with first-class functions let us treat them like values (or objects) that can be passed as parameters, returned from other functions, or assigned to variables. Lambda expressions in C++ look unique compared to other languages. 

```cpp
[](const int i) { return 0 == i % 2; };
```

Here is a lambda that captures no values from the surrounding scope (`[]`), takes in an `int`, and returns `0 == i % 2;`. We can pass this to a function or assign it a name like `even`. The syntax for a lambda is `[<capture-list>](<arguments>){<body>}`.

```cpp
auto const even = [](const int i) { return 0 == i % 2; };
```

Here we use the `auto` keyword to infer the type because lambda types in C++ are a little complicated. 

Under the hood, lambdas compile down to a **functor** which is an object whose class has overridden the `()` operator. In other words, it’s an object that that can be called like a function. 

When capturing identifiers from the surrounding lexical scope, it’s important to take **move semantics** into account (I.e. deciding whether the lambda takes ownership of the memory of the captured identifier or simply receives a copy or reference). Lambdas will be especially useful when using functions that take other functions as an argument. 

[https://en.cppreference.com/w/cpp/language/lambda](https://en.cppreference.com/w/cpp/language/lambda)

# Ranges and Views

First, let’s review what an **iterator** is. In C++, it’s common to implement all kinds of data structures (trees, maps, sets, queues, etc.) but you would like to be able to perform operations on each element of the structure in sequence like you would with a simple array. C++ commonly uses the iterator pattern to solve this problem. Iterators are a generic way to operate on a sequence of elements inside any data structure. 

For example, we can get an iterator from a `std::vector` by calling `.begin()` on it. This gives us *essentially* a pointer to the first element of the vector. We can then increment or decrement the iterator to out hearts’ content to get subsequent elements. 
 

```cpp
#include<iostream>
#include<iterator> // for iterators
#include<vector> // for vectors
using namespace std;
int main()
{
    vector<int> ar = { 1, 2, 3, 4, 5 };
      
    // Declaring iterator to a vector
    vector<int>::iterator ptr;
      
    // Displaying vector elements using begin() and end()
    cout << "The vector elements are : ";
    for (ptr = ar.begin(); ptr < ar.end(); ptr++)
        cout << *ptr << " ";
      
    return 0;    
}
```

These are useful, but as of C++20, there is now a higher abstraction called **ranges**. 

A range is an object the carries iterators inside of it and indirectly refers to a real data structure in memory but is easier to use with standard library algorithms and is easier to compose. 

A **view** is a range that has the property that it implements constant-time copy, move, and assignment. ("Range" is a more general term, but for the sake of using the standard library algorithms I will only be using ranges that are also views so I'll use the terms interchangeably). 

Here is an example of the standard library range function `filter` which is is a **higher-order function** that takes in (1) a data structure from which a range can be derived (pretty much any STL container class) and (2) a function predicate and returns a new range. 

```cpp
auto even = [](int i) { return 0 == i % 2; };
vector<int> ar = { 1, 2, 3, 4, 5 };
auto evens = std::views::filter(ar, even);
```

In most languages with higher order functions, `evens` would be a new array or vector with `{2, 4}`. The filter function would apply the predicate to every element and keep only the ones for which the predicate returns `true` and leave out the ones where the predicate returns `false`. However, this is C++ and we care about fine-grained performance control! We don’t want to allocate a new vector unnecessarily. So instead, we get a new view that represents an indirect “lens” with which to “look” into the original vector. When we do finally “look” through that “lens”, we will only see the elements that made it past the filter. (More on “looking” later).

That view `evens` can then be passed off to another algorithm such as `std::transform` which is like `map` in other languages. It takes a view and a function and applies that function to every element. 

 

```cpp
auto even = [](int i) { return 0 == i % 2; };
vector<int> ar = { 1, 2, 3, 4, 5 };
auto evens = std::views::filter(ar, even);
auto square = [](const int i) { return i * i; };
auto squared_evens = std::views::transform(evens, square); 
```

Here, `squared_evens` is a view into the original vector where we only see the even elements after they have been squared. 

So how do we get a new vector? Well, this brings up a very important point: views are **lazy**. This means that even after calling all these standard algorithms, we still have not invoked the `even` function or the `sqaure` function on a single element. It’s only when we *ask* for a value from the range that an element will get sent down the pipeline where it is first filtered and then transformed. 

```cpp
auto even = [](int i) { return 0 == i % 2; };
vector<int> ar = { 1, 2, 3, 4, 5 };
auto evens = std::views::filter(ar, even);
auto square = [](const int i) { return i * i; };
auto squared_evens = std::views::transform(evens, square);
for (int i : squared_evens) {
	std::cout << i << ' ';
}
```

Ranges can be used in for-loops like this. So, here we are printing out all the values that will come out of the view. The result printed to the screen would be "4 16". We could also add these to a new vector. 

```cpp
auto even = [](int i) { return 0 == i % 2; };
vector<int> ar = { 1, 2, 3, 4, 5 };
auto evens = std::views::filter(ar, even);
auto square = [](const int i) { return i * i; };
auto squared_evens = std::views::transform(evens, square);
vector<int> new_vec = {};
for (int i : squared_evens) {
	new_vec.push_back(i);
}
```

Ranges have a fun property where they can be **composed** with the `|` (pipe) operator. This allows us to pass views along to algorithms with a concise syntax. Here is the above example using the pipe. 

```cpp
auto even = [](int i) { return 0 == i % 2; };
auto square = [](const int i) { return i * i; };
vector<int> ar = { 1, 2, 3, 4, 5 };
vector<int> new_vec = {};
for (int i : ar | std::views::filter(even) | std::views::transform(square)) {
	new_vec.push_back(i);
}
```

In one expression (`ar | std::views::filter(even) | std::views::transform(sqaure)` ) we created our view that filters even elements and squares them. You can quite easily mentally visualize the `ar` being passed off to `filter` and then the result of that being passed off to `transform`. Those familiar with the `|` pipe operator in Unix/Linux or the `|>` operator in languages like F# and Elm will find this to be quite natural. 

Overall, ranges, views, and standard library functions are a great way to work with a variety of collection types at a higher level of abstraction while still maintaining C++’s standard for performance. 

[https://en.cppreference.com/w/cpp/ranges](https://en.cppreference.com/w/cpp/ranges). 

# Conclusion

This has been a whirlwind tour of some of the features of modern C++ that facilitate functional programming. C++ at its core has traditionally relied heavily on mutation and side-effects to get work done. However, with recent releases of the language we have seen a gradual shift towards more concise and functional code. At the end of the day, programming functionally in any language will help you separate calculations (pure functions) from actions (side-effects/mutation) and that will make your code easier to reason about, refactor, and test.