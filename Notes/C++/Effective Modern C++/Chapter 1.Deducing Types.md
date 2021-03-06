[TOC]

type deduction：类型推导

function templates，auto，deltype expressions，C++14：decltype(auto)

# Item 1: Understand template type deduction

when the template type deduction rules are applied in the context of auto, they sometimes seem less intuitive than when they’re applied to templates.

a function template:

```c++
template<typename T>
void f(ParamType param);
```

call:

```c++
f(expr);
```

编译器会用`expr`推导出`T`和`ParamType`的类型。（后者经常含有修饰符，如const或引用限定符）

（我们可能希望T就是expr的类型，但它并不总是这样工作的）

The type deduced for T is dependent not just on **the type of expr**, but also on **the form of ParamType**.

There are three cases:

- `ParamType` is **a pointer** or **reference type**, but not **a universal reference**. （Item 24中描述。此时只需要知道，它们是存在的，而且它们和lvalue references或rvalue references不同）
- `ParamType` is a **universal reference**.
- `ParamType` is neither **a pointer** nor **a reference**.

## Case 1:`ParamType` is **a pointer** or **reference type**, but not **a universal reference**. 

In that case, type deduction works like this:

1. If `expr`’s type is **a reference**, ignore the reference part.
2. Then pattern-match `expr`’s type against `ParamType` to determine `T`.

```c++
template<typename T>
void f(T& param); // param is a reference
```

```c++
int x = 27; // x is an int
const int cx = x; // cx is a const int
const int& rx = x; // rx is a reference to x as a const int
```

the deduced types for `param` and `T` in various calls are as follows:

```c++
f(x); // T is int, param's type is int&
f(cx); // T is const int,
 // param's type is const int&
f(rx); // T is const int,
 // param's type is const int&
```

（That’s why passing a const object to a template taking a `T&` parameter is safe: the constness of the object becomes part of the type deduced for `T`.）

```c++
template<typename T>
void f(const T& param); // param is now a ref-to-const
```

```c++
int x = 27; // as before
const int cx = x; // as before
const int& rx = x; // as before
```

```c++
f(x); // T is int, param's type is const int&
f(cx); // T is int, param's type is const int&
f(rx); // T is int, param's type is const int&
```

The constness of `cx` and `rx` continues to be respected, but because we’re now assuming that param is a reference-to-`const`, there’s no longer a need for `const` to be deduced as part of `T`.

If param were a **pointer** (or a pointer to const) instead of a reference, things would work essentially the same way:

```c++
template<typename T>
void f(T* param); // param is now a pointer
```

```c++
int x = 27; // as before
const int *px = &x; // px is a ptr to x as a const int
```

```c++
f(&x); // T is int, param's type is int*
f(px); // T is const int,
 // param's type is const int*
```

## Case 2:`ParamType` is a **universal reference**.

（Things are less obvious for templates taking universal reference parameters.）

Such parameters are declared like rvalue references, but they behave differently when lvalue arguments are passed in.

（The complete story is told in Item 24, but here’s the headline version:）

```c++
template<typename T>
void f(T&& param); // param is now a universal reference

int x = 27; // as before
const int cx = x; // as before
const int& rx = x; // as before

f(x); // x is lvalue, so T is int&,
 // param's type is also int&
f(cx); // cx is lvalue, so T is const int&,
 // param's type is also const int&
f(rx); // rx is lvalue, so T is const int&,
 // param's type is also const int&
f(27); // 27 is rvalue, so T is int,
 // param's type is therefore int&&
```

The key point here is that the type deduction rules for universal reference parameters are different from those for parameters that are lvalue references or rvalue references.

In particular, when universal references are in use, type deduction distinguishes between lvalue arguments and rvalue arguments.（That never happens for non-universal references.）

## Case 3:`ParamType` is neither **a pointer** nor **a reference**.

```c++
template<typename T>
void f(T param); // param is now passed by value
```

When `ParamType` is neither a pointer nor a reference, we’re dealing with **pass-by-value**:

1. As before, if `expr`’s type is a reference, ignore the reference part.
2. If, after ignoring `expr`’s referenceness, `expr` is `const`, ignore that, too. If it’s `volatile`, also ignore that. (volatile objects are uncommon.They’re generally used only for implementing device drivers.For details, see Item 40.)

```c++
int x = 27; // as before
const int cx = x; // as before
const int& rx = x; // as before
```

```c++
f(x); // T's and param's types are both int
f(cx); // T's and param's types are again both int
f(rx); // T's and param's types are still both int
```

`param` is an object that’s completely independent of `cx` and `rx`—**a copy of** `cx` or `rx`.The fact that `cx` and `rx` can’t be modified says nothing about whether `param` can be.（That’s why `expr`’s constness (and volatileness, if any) is ignored when deducing a type for `param`: just because `expr` can’t be modified doesn’t mean that a copy of it can’t be.）

But consider the case where `expr` is a const pointer to a const object, and `expr` is passed to a by-value param:

```c++
template<typename T>
void f(T param); // param is still passed by value

const char* const ptr = // ptr is const pointer to const object
 "Fun with pointers";

f(ptr); // pass arg of type const char * const
```

When `ptr` is passed to `f`, the bits making up the pointer are copied into `param`.As such, the pointer itself (`ptr`) will be **passed by value**（the type deduced for param will be `const char*`）.（The constness of what `ptr` points to is preserved during type deduction, but the constness of `ptr` itself is ignored when copying it to create the new pointer, `param`.）

### Array Arguments

Array types are different from pointertypes, even though they sometimes seem to be interchangeable.

```c++
const char name[] = "J. P. Briggs"; // name's type is
 // const char[13]
const char * ptrToName = name; // array decays to pointer
```

But what if an array is passed to **a template taking a by-value parameter**? What happens then?

```c++
template<typename T>
void f(T param); // template with by-value parameter

f(name); // what types are deduced for T and param?
```

这两种声明是等价的，这使人们误认为这两种类型是相同的：

```c++
void myFunc(int param[]);
void myFunc(int* param); // same function as above
```

（Because array parameter declarations are treated as if they were pointer parameters, the type of an array that’s passed to a template function by value is deduced to be a pointer type.）

That means that in the call to the template `f`, its type parameter `T` is deduced to be `const char*`:

```c++
f(name); // name is array, but T deduced as const char*
```

But now comes a curve ball.

Although functions can’t declare parameters that are truly arrays, they can declare parameters that are **references to arrays**!

```c++
template<typename T>
void f(T& param); // template with by-reference parameter

f(name); // pass array to f
```

So if we modify the template `f` to take its argument by reference,and we pass an array to it,the type deduced for `T` is the actual type of the array!

That type includes the size of the array, so in this example, `T` is deduced to be `const char [13]`, and the type of `f`’s parameter (a reference to this array) is `const char (&)[13]`.

（Interestingly, the ability to declare references to arrays enables creation of a template that deduces the number of elements that an array contains:）😀

```c++
// return size of an array as a compile-time constant. (The
// array parameter has no name, because we care only about
// the number of elements it contains.)
template<typename T, std::size_t N> // see info
constexpr std::size_t arraySize(T (&)[N]) noexcept // below on
{ // constexpr
    return N; // and
} // noexcept
```

（As Item 15 explains, declaring this function `constexpr` makes its result available during compilation.）

That makes it possible to declare, say, an array with the same number of elements as a second array whose size is computed from a braced initializer:😀

```c++
int keyVals[] = { 1, 3, 7, 9, 11, 22, 35 }; // keyVals has
// 7 elements
int mappedVals[arraySize(keyVals)]; // so does
// mappedVals
```

Of course, as a modern C++ developer, you’d naturally prefer a `std::array` to a built-in array:

```c++
std::array<int, arraySize(keyVals)> mappedVals; // mappedVals'
// size is 7
```

（As for `arraySize` being declared `noexcept`, that’s to help compilers generate better code. For details, see Item 14.）

### Function Arguments

Function types can decay into **function pointers**, and everything we’ve discussed regarding type deduction for arrays applies to type deduction for functions and their decay into function pointers.

As a result:

```c++
void someFunc(int, double); // someFunc is a function;

// type is void(int, double)
template<typename T>
void f1(T param); // in f1, param passed by value

template<typename T>
void f2(T& param); // in f2, param passed by ref

f1(someFunc); // param deduced as ptr-to-func;
// type is void (*)(int, double)

f2(someFunc); // param deduced as ref-to-func;
// type is void (&)(int, double)
```

This rarely makes any difference in practice, but if you’re going to know about array-to-pointer decay, you might as well know about function-to-pointer decay, too.

So there you have it: the auto-related rules for template type deduction.

（I remarked at the outset that they’re pretty straightforward, and for the most part, they are.The special treatment accorded lvalues when deducing types for universal references muddies the water a bit, however, and the decay-to-pointer rules for arrays and functions stirs up even greater turbidity.）

（Sometimes you simply want to grab your compilers and demand, “Tell me what type you’re deducing!” When that happens, turn to Item 4, because it’s devoted to coaxing compilers into doing just that.）

## Thing to Remember

- During template type deduction, arguments that are references are treated as non-references .（即their referenceness is ignored.）
- When deducing types for universal reference parameters, lvalue arguments get special treatment.
- When deducing types for by-value parameters, const and/or volatile arguments are treated as non-const and non-volatile.
- During template type deduction, arguments that are array or function names decay to pointers, unless they’re used to initialize references.