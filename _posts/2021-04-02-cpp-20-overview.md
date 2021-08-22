---
layout: post
title:  "All C++20 core language features with examples"
date:   2021-04-02 14:12:00 +0300
post_url: 2021-04-02-cpp-20-overview
---

## Introduction

The story behind this article is very simple, I wanted to learn about new C++20
language features and to have a brief summary for all of them on a single page.
So, I decided to read all proposals and create this "cheat sheet" that
explains and demonstrates each feature.
This is not a "best
practices" kind of article, it serves only demonstrational purpose.
Most examples were inspired or directly taken from corresponding proposals,
all credit goes to their authors and to members of ISO C++ committee for
their work. Enjoy!

## Table of contents

* [Concepts](#concepts)
* [Modules](#modules)
* [Coroutines](#coroutines)
* [Three-way comparison](#three-way-comparison)
* [Lambda expressions](#lambda-features)
  * [Allow lambda-capture `[=, this]`](#lambda-this)
  * [Template parameter list for generic lambdas](#lambda-templ-params)
  * [Lambdas in unevaluated contexts](#lambda-uneval-ctx)
  * [Default constructible and assignable stateless lambdas](#lambda-def-ctor)
  * [Pack expansion in lambda init-capture](#lambda-pack-exp)
* [Constant expressions](#constexpr-features)
  * [Immediate functions(`consteval`)](#consteval)
  * [`constexpr` virtual function](#constexpr-virtual)
  * [`constexpr` try-catch blocks](#constexpr-try-catch)
  * [`constexpr` `dynamic_cast` and polymorphic `typeid`](#constexpr-dyn-cast)
  * [Changing the active member of a `union` inside `constexpr`](#constexpr-union)
  * [`constexpr` allocations](#constexpr-alloc)
  * [Trivial default initialization in `constexpr` functions](#constexpr-trivial-def-init)
  * [Unevaluated `asm`-declaration in `constexpr` functions](#constexpr-asm)
  * [`std::is_constant_evaluated()`](#is-const-eval)
* [Aggregates](#aggregates)
  * [Prohibit aggregates with user-declared constructors](#aggr-no-ctor)
  * [Class template argument deduction for aggregates](#ctad-aggr)
  * [Parenthesized initialization of aggregates](#aggr-paren-init)
* [Non-type template parameters](#nttp)
  * [Class types in non-type template parameters](#class-types-nttp)
  * [Generalized non-type template parameters](#nttp-gen)
* [Structured bindings](#struct-bindings)
  * [Lambda capture and storage class specifiers for structured bindings](#structbind-specs)
  * [Relaxing the structured bindings customization point finding rules](#fix-structbind-cp)
  * [Allow structured bindings to accessible members](#fix-structbind-access)
* [Range-based `for` loop](#range-based-for)
  * [init-statements for range-based `for` loop](#init-range-for)
  * [Relaxing the range-based `for` loop customization point finding rules](#fix-range-for-cp)
* [Attributes](#attributes)
  * [`[[likely]]` and `[[unlikely]]`](#attr-likely)
  * [`[[no_unique_address]]`](#attr-no-uniq-addr)
  * [`[[nodiscard]]` with message](#discard-msg)
  * [`[[nodiscard]]` for constructors](#fix-nodiscard-ctor)
* [Character encoding](#encoding)
  * [`char8_t`](#char8t)
  * [Stronger Unicode requirements](#stronger-unicode)
* [Sugar](#sugar)
  * [Designated initializers](#designated-init)
  * [Default member initializers for bit-fields](#bitfield-def-init)
  * [More optional `typename`](#less-typename)
  * [Nested `inline` namespaces](#nested-inline-ns)
  * [`using enum`](#using-enum)
  * [Array size deduction in new-expressions](#fix-arr-size)
  * [Class template argument deduction for alias templates](#ctad-alias)
* [`constinit`](#constinit)
* [Signed integers are two's complement](#int-twos-compl)
* [`__VA_OPT__` for variadic macros](#va-opt)
* [Explicitly defaulted functions with different exception specifications](#diff-except-spec)
* [Destroying `operator delete`](#destr-delete)
* [Conditionally `explicit` constructors](#explicit-conditional)
* [Feature-test macros](#feature-test-macros)
* [Known-to-unknown bound array conversions](#array-conv)
* [Implicit move for more local objects and rvalue references](#more-impl-moves)
* [Conversion from `T*` to `bool` is narrowing](#narrowing-ptr-bool-conv)
* [Deprecate some uses of `volatile`](#depr-volatile)
* [Deprecate comma operator in subscripts](#depr-comma-subs)
* [Fixes](#fixes)
  * [Initializer list constructors in class template argument deduction](#fix-init-list-ctad)
  * [`const&`-qualified pointers to members](#fix-const-qual)
  * [Simplifying implicit lambda capture](#fix-impl-capture)
  * [`const` mismatch with defaulted copy constructor](#fix-const-mismatch)
  * [Access checking on specializations](#fix-spec-access-check)
  * [ADL and function templates that are not visible](#fix-adl)
  * [Specify when `constexpr` function definitions are needed for constant evaluation](#fix-constexpr-inst)
  * [Implicit creation of objects for low-level object manipulation](#fix-impl-creation)

---

## Concepts {#concepts}

The basic idea behind concepts is to specify what's needed from a template
argument so the compiler can check it before instantiation. As a result, the
error message, if any, is much cleaner, something like `constraint X was not satisfied`.
Before C++20 it was possible to use tricky `enable_if` constructions or 
just fail during template instantiation with cryptic error messages. With concepts 
failure happens early and the error message is much cleaner.

### Requires expression

Let's start with `requires-expression`. It's an expression that contains 
actual 
requirements for template arguments, it evaluates to `true` if they are satisfied 
and `false` otherwise.

```cpp
template<typename T> /*...*/
requires (T x) // optional set of fictional parameter(s)
{
    // simple requirement: expression must be valid
    x++;    // expression must be valid
    
    // type requirement: `typename T`, T type must be a valid type
    typename T::value_type;
    typename S<T>;

    // compound requirement: {expression}[noexcept][-> Concept];
    // {expression} -> Concept<A1, A2, ...> is equivalent to
    // requires Concept<decltype((expression)), A1, A2, ...>
    {*x};  // dereference must be valid
    {*x} noexcept;  // dereference must be noexcept
    // dereference must  return T::value_type
    {*x} noexcept -> std::same_as<typename T::value_type>;
    
    // nested requirement: requires ConceptName<...>;
    requires Addable<T>; // constraint Addable<T> must be satisfied
};
```

### Concept

Concept is simply a named set of such constraints or their logical combination.
Both concept and requires-expression render to a compile-time bool value and 
can be used as a normal value, for example in `if constexpr`.

```cpp
template<typename T>
concept Addable = requires(T a, T b)
{
    a + b;
};

template<typename T>
concept Dividable = requires(T a, T b)
{
    a/b;
};

template<typename T>
concept DivAddable = Addable<T> && Dividable<T>;

template<typename T>
void f(T x)
{
    if constexpr(Addable<T>){ /*...*/ }
    else if constexpr(requires(T a, T b) { a + b; }){ /*...*/ }
}
```

### Requires clause

To actually constrain something we need `requires-clause`. It may appear right
after `template<>` block or as the last element of a function declaration, or
even at both places at once, lambdas included:

```cpp
template<typename T>
requires Addable<T>
auto f1(T a, T b) requires Subtractable<T>; // Addable<T> && Subtractable<T>

auto l = []<typename T> requires Addable<T>
    (T a, T b) requires Subtractable<T>{};

template<typename T>
requires Addable<T>
class C;

// infamous `requires requires`. First `requires` is requires-clause,
// second one is requires-expression. Useful if you don't want to introduce new
// concept.
template<typename T>
requires requires(T a, T b) {a + b;}
auto f4(T x);
```

Much cleaner way is to use concept name instead of `class/typename` keyword in
template parameter list:

```cpp
template<Addable T>
void f();
```

Template template parameters can also be constrained. In this case argument must 
be less or equally constrained than parameter. Unconstrained template template
parameters still can accept constrained templates as arguments:

```cpp
template<typename T>
concept Integral = std::integral<T>;

template<typename T>
concept Integral4 = std::integral<T> && sizeof(T) == 4;

// requires-clause also works here
template<template<typename T1> requires Integral<T1> typename T>
void f2(){}

// f() and f2() forms are equal
template<template<Integral T1> typename T>
void f(){
    f2<T>();
}

// unconstrained template template parameter can accept constrained arguments
template<template<typename T1> typename T>
void f3(){}

template<typename T>
struct S1{};

template<Integral T>
struct S2{};

template<Integral4 T>
struct S3{};

void test(){
    f<S1>();    // OK
    f<S2>();    // OK
    // error, S3 is constrained by Integral4 which is more constrained than
    // f()'s Integral
    f<S3>();

    // all are OK
    f3<S1>();
    f3<S2>();
    f3<S3>();
}
```

Functions with unsatisfied constraints become "invisible":

```cpp
template<typename T>
struct X{
    void f() requires std::integral<T>
    {}
};

void f(){
    X<double> x;
    x.f();  // error
    auto pf = &X<double>::f;    // error
}
```

### Constrained `auto`

`auto` parameters now allowed for normal functions to make them generic just like
generic lambdas. Concepts can be used to constrain placeholder 
types(`auto`/`decltype(auto)`) in various contexts.
For parameter packs, `MyConcept... Ts` requires
`MyConcept` to be true for each element of the pack, not for the whole pack at 
once, e.g. `requires<T1> && requires<T2> && ... && requires<TLast>`.

```cpp
template<typename T>
concept is_sortable = true;

auto l = [](auto x){};
void f1(auto x){}               // unconstrained template
void f2(is_sortable auto x){}   // constrained template

template<is_sortable auto NonTypeParameter, is_sortable TypeParameter>
is_sortable auto f3(is_sortable auto x, auto y)
{
    // notice that nothing is allowed between constraint name and `auto`
    is_sortable auto z = 0;
    return 0;
}

template<is_sortable auto... NonTypePack, is_sortable... TypePack>
void f4(TypePack... args){}

int f();

// takes two parameters
template<typename T1, typename T2>
concept C = true;
// binds second parameter
C<double> auto v = f(); // means C<int, double>

struct X{
    operator is_sortable auto() {
        return 0;
    }
};

auto f5() -> is_sortable decltype(auto){
    f4<1,2,3>(1,2,3);
    return new is_sortable auto(1);
}
```

### Partial ordering by constraints

*This section was inspired by the article
[Ordering by constraints](https://akrzemi1.wordpress.com/2020/05/07/ordering-by-constraints/)
by Andrzej Krzemieński. Check it out for a more thorough explanation.*

Aside from specifying requirements for a single declaration, constraints can be
used to select the best alternative for a normal function, template function or
a class template. To do so, constraints have a notion of partial ordering, that is,
one constraint can be *at least* or *more* constrained than the other or they can
be *unordered*(unrelated). Compiler decomposes(the Standard uses term
*normalization* but for me *decomposition* sounds better) constraint into a conjunction/
disjunction of *atomic* constraints. Intuitively, `C1 && C2` is more constrained than
`C1`, `C1` is more constrained than `C1 || C2` and any constraint is more
constrained than the unconstrained declaration. When more than one candidate
with satisfied constraints are present, the most constrained one is chosen. If
constraints are unordered, the usage is ambiguous.

```cpp
template<typename T>
concept integral_or_floating = std::integral<T> || std::floating_point<T>;

template<typename T>
concept integral_and_char = std::integral<T> && std::same_as<T, char>;

void f(std::integral auto){}        // #1
void f(integral_or_floating auto){} // #2
void f(std::same_as<char> auto){}   // #3

// calls #1 because std::integral is more constrained
// than integral_or_floating(#2)
f(int{});
// calls #2 because it's the only one whose constraint is satisfied
f(double{});
// error, #1, #2 and #3's constraints are satisfied but unordered
// because std::same_as<char> appears only in #3
f(char{});

void f(integral_and_char auto){}    // #4

// calls #4 because integral_and_char is more
// constrained than std::same_as<char>(#3) and std::integral(#1)
f(char{});
```

It's important to understand how the compiler decomposes constraints and when it
can see that they have common atomic constraint and deduce order between
them.
During decomposition, the concept name is replaced with its definition but
`requires-expression` is *not* further decomposed. Two atomic constraints are
identical only if they are represented by the same expression at the same
location.
For example, `concept C = C1 && C2` is decomposed to conjunction of `C1` and
`C2` but `concept C = requires{...}` becomes `concept C = Expression-Location-Pair`
and its body is not further decomposed.
If two concepts
have common or even the same requirements in their `requires-expression`,
they will always be unordered because either their `requires-expression`s are
not equal or they are equal but at different source locations. The same happens
with duplicated usage of a naked type traits - they always represent different
atomic constraints because of different locations, thus, cannot be used for
ordering.

```cpp
template<typename T>
requires std::is_integral_v<T>  // uses type traits instead of concepts
void f1(){}  // #1

template<typename T>
requires std::is_integral_v<T> || std::is_floating_point_v<T>
void f1(){}  // #2

// error, #1 and #2 have common `std::is_integral_v<T>` expression
// but at different locations(line 2 vs. line 6), thus, #1 and #2 constraints
// are unordered and the call is ambiguous
f1(int{});

template<typename T>
concept C1 = requires{      // requires-expression is not decomposed
    requires std::integral<T>;
};

template<typename T>
concept C2 = requires{      // requires-expression is not decomposed
    requires (std::integral<T> || std::floating_point<T>);
};

void f2(C1 auto){}  // #3
void f2(C2 auto){}  // #4

// error, since requires-expressions are not decomposed, #3 and #4 have
// completely unrelated and hence unordered constraints and the call is
// ambiguous
f2(int{});
```

### Conditionally trivial special member functions

For wrapper types like `std::optional` or `std::variant` it's useful to
propagate *triviality* from the types they wrap. For example, `std::optional<int>`
should be trivial but `std::optional<std::string>` shouldn't. In C++17 this can be
achieved using [pretty cumbersome machinery](https://wg21.link/P0848R3#introduction).
Concepts provide a natural solution for this: we can create multiple versions of
the same special member function with different constraints, the compiler will
choose the best one and ignore the others. In this particular case, we need
a trivial set of functions when the wrapped type is a trivial and a non-trivial 
set of
functions when it's not. For this to work, some updates have been made to the
definition of trivial type. In C++17, a trivially copyable class is required to 
have *all* of its copy and move operations either deleted or trivial.
To take concepts into account, the notion of an *eligible special member function*
was introduced. It is a function that's not deleted, whose constraints(if any) are
satisfied and no other special member function of the same kind, with the same
first parameter type(if any), is more constrained. Simply put, it's a
function(s) with the most constrained satisfied constraints(if any). All existing
destructors(yes, now you can have more than one) are now called *prospective*
destructors. Only one "active" destructor is allowed, it's selected
using normal overload resolution.  
A *trivially copyable* class is now a class that has a *trivial* 
non-deleted destructor, *at least one* eligible
copy/move operation and whose all such eligible operations are trivial. A 
*trivial* class is a trivially copyable class that has one or more eligible 
default constructors, all of which are trivial.  
Here's the skeleton of this technique:

```cpp
template<typename T>
class optional{
public:
    optional() = default;

    // trivial copy-constructor
    optional(const optional&) = default;

    // non-trivial copy-constructor
    optional(const optional& rhs)
        requires(!std::is_trivially_copy_constructible_v<T>){
        // ...
    }

    // trivial destructor
    ~optional() = default;

    // non-trivial destructor
    ~optional() requires(!std::is_trivial_v<T>){
        // ...
    }
    // ...
private:
    T value;
};

static_assert(std::is_trivial_v<optional<int>>);
static_assert(!std::is_trivial_v<optional<std::string>>);
```

---

## Modules {#modules}

Modules is a new way to organize C++ code into logical components. Historically,
C++ used C model
which is based on the preprocessor and repetitive textual inclusion. It has
a lot of problems such as macros leakage in and out from headers, 
inclusion-order-dependent
headers, repetitive compilation of the same code, cyclic dependencies, poor 
encapsulation of implementation details and so on. Modules are about to solve them
but not so fast. We won't be able to use their full power until compilers *and*
build tools, such as CMake, will support it too. Full description of Modules is
well beyond the scope of this article, I will only show the basic ideas and use 
cases. For more details you can read
[a series of articles by `vector-of-bool`](https://vector-of-bool.github.io/2019/03/10/modules-1.html)
or just google for other blog posts or talks.

The main idea behind modules is to restrict what's accessible(`export`ed) when 
a module is used(`import`ed) by its clients. This allows true hiding of 
implementation details.

```cpp
// module.cpp
// dots in module name are for readability purpose, they have no special meaning
export module my.tool;  // module declaration

export void f(){}       // export f()
void g(){}              // but not g()

// client.cpp
import my.tool;

f();    // OK
g();    // error, not exported
```

Modules are macro-unfriendly, you
can't pass manually `#define`d macros to module(compiler's built-in and
command-line macros are still visible) and only in one special case you can import
macros from
module. Modules can't have cyclic dependencies. Module is a self-contained
entity, compiler can precompile
each module exactly once so overall compilation time is greatly improved. Import
order doesn't matter for modules.

### Module units

A module can be either *interface* or *implementation* module unit. Only interface
units can contribute to the module's interface, that's why they have `export` in
their
declaration. A module can be a single file or scattered across *partitions*. Each
partition is named in the form `module_name:partition_name`. Partitions are 
`import`able only within the same module and client can `import` only a module as 
a whole. This provides much better encapsulation than header files.

```cpp
// tool.cpp
export module tool; // primary module interface unit
export import :helpers; // re-export(see below) helpers partition

export void f();
export void g();

// tool.internals.cpp
module tool:internals;  // implementation partition
void utility();

// tool.impl.cpp
module tool;    // implementation unit, implicitly imports primary module unit
import :internals;

void utility(){}

void f(){
    utility();
}

// tool.impl2.cpp
module tool;    // another implementation unit
void g(){}

// tool.helpers.cpp
export module tool:helpers; // module interface partition
import :internals;

export void h(){
    utility();
}

// client.cpp
import tool;

f();
g();
h();
```

Note that partitions are imported without specifying module name. This prohibits
importing other module's partitions. Multiple implementation units(
`module tool;`) are allowed, all other units and partitions of any kind must
be unique. All interface partitions must be re-exported by the module via
`export import`.

### Export

Here are various forms of `export`, the general rule is that you can't `export`
names with internal linkage:

```cpp
// tool.cpp
module tool;
export import :helpers; // import and re-export helpers interface partition

export int x{}; // export single declaration

export{         // export multiple declarations
    int y{};
    void f(){};
}

export namespace A{ // export the whole namespace
    void f();
    void g();
}

namespace B{
    export void f();// export a single declaration within a namespace
    void g();
}

namespace{
    export int x;   // error, x has internal linkage
    export void f();// error, f() has internal linkage
}

export class C; // export as incomplete type
class C{};
export C get_c();

// client.cpp
import tool;

C c1;    // error, C is incomplete
auto c2 = get_c();  // OK
```

### Import

Import declarations should precede any other "non-module" declarations, it
allows quick dependency analysis. Otherwise, it's pretty intuitive:

```cpp
// tool.cpp
export module tool;
import :helpers;  // import helpers partition

export void f(){}

// tool.helpers.cpp
export module tool:helpers;

export void g(){}

// client.cpp
import tool;

f();
g();
```

### Header units

There's one special `import` form that allows import of *importable* headers:
`import <header.h>` or `import "header.h"`. Compiler creates a synthesized
*header unit* and makes all declarations implicitly exported.
What headers are actually importable
is implementation-defined but all C++ library headers are so. Perhaps, there will 
be a way to tell the compiler which user-provided headers are importable, such 
headers should not contain non-inline function definitions or variables with
external linkage. It's the only
`import` form that allows import of macros from headers(but you still can't 
re-export them via `export import "header.h"`). Don't use it to import random
legacy header if you're not sure about its content.

### Global module fragment

If you need to use old-school headers within a module, there's a special place
to put `#include`s safely: *global module fragment*:

```cpp
// header.h
#pragma once
class A{};
void g(){}

// tool.cpp
module;             // global module fragment
#include "header.h"
export module tool; // ends here

export void f(){    // uses declarations from header.h
    g();
    A a;
}
```

It must appear before the named module declaration and it can contain only 
preprocessor
directives. All declarations from all global module fragments and non-modular
translation units are attached to a single global module. Thus, all rules for 
normal headers apply here.

### Private module fragment

The final strange beast is a *private module fragment*. Its intent is to hide
implementation details in a single-file module(it's not allowed elsewhere). In 
theory, clients might
not recompile when things in a private module fragment changes:

```cpp
export module tool; // interface

export void f();    // declared here

module :private;    // implementation details

void f(){}          // defined here
```

### No more implicit `inline`

There's also an interesting change regarding `inline`. Member functions defined
within the class definition are *not* implicitly `inline` if that class is attached
to a named module. `inline` functions in a named module can use only names
that are visible to a client.

```cpp
// header.h
struct C{
    void f(){}  // still inline because attached to a global module
};

// tool.cpp
module;
#include "header.h"

export module tool;

class A{};  // not exported

export struct B{// B is attached to module "tool"
    void f(){   // not implicitly inline anymore
        A a;    // can safely use non-exported name
    }

    inline void g(){
        A a;    // oops, uses non-exported name
    }

    inline void h(){
        f();    // fine, f() is not inline
    }
};

// client.cpp
import tool;

B b;
b.f();  // OK
b.g();  // error, A is undefined
b.h();  // OK
```

---

## Coroutines {#coroutines}

Finally, we have stackless(their state is stored in heap, not on stack)
[coroutines](https://en.wikipedia.org/wiki/Coroutine) in C++. C++20 provides
nearly the lowest possible API and leaves rest up to the user.
We've got `co_await`,
`co_yield`, `co_return` keywords and rules for interaction between the caller
and callee. Those rules are so low-level that I see no point in explaining them
here.
You can find more details on [Lewis Baker's blog](https://lewissbaker.github.io/).
Hopefully, C++23 will fill this gap with some library utilities. Until then,
we can use third-party libraries, here's an example 
that uses [cppcoro](https://github.com/lewissbaker/cppcoro):

```cpp
cppcoro::task<int> someAsyncTask()
{
    int result;
    // get the result somehow
    co_return result;
}

// task<> is analog of void for normal function
cppcoro::task<> usageExample()
{
    // creates a new task but doesn't start executing the coroutine yet
    cppcoro::task<int> myTask = someAsyncTask();
    // ...
    // Coroutine is only started when we later co_await the task.
    auto result = co_await myTask;
}

// will lazily generate numbers from 0 to 9
cppcoro::generator<std::size_t> getTenNumbers()
{
    std::size_t n{0};
    while (n != 10)
    {
        co_yield n++;
    }
}

void printNumbers()
{
    for(const auto n : getTenNumbers())
    {
        std::cout << n;    
    }
}
```

---

## Three-way comparison {#three-way-comparison}

Before C++20, to provide comparison operations for a class,
implementations of 6 operators are needed: `==, !=, <, <=, >, >=`.
Usually, four of them contain boiler-plate code that works in terms of `==` and
`<` which contain the real comparison logic.
Common practice 
is to implement them as free functions taking `const T&` to allow comparison of
convertible types.
If you want to support non-convertible types, you need to add two sets of 6 
functions, `op(const T1&, const T2&)` and `op(const T2&, const T1&)` and now you
have 18 comparison operators(check out [`std::optional`](https://en.cppreference.com/w/cpp/utility/optional/operator_cmp)).
C++20 gives us a better way to handle and think about comparisons. Now you need
to focus on `operator<=>()` and sometimes on `operator==()`.
New `operator<=>`(spaceship
operator) implements three-way comparison, it tells whether `a` is less,
equal or greater than `b` in a single call, just like `strcmp()`. It returns a
comparison category(see below) that could be compared to zero. Having this,
compiler can replace calls to `<, <=, >, >=` with call to `operator<=>()` and
check its result(`a < b` becomes `a <=> b < 0`), and calls to `==, !=` to `operator==()`(`a != b` becomes `!(a == b)`).
Due to new lookup rules they can handle asymmetric comparisons, e.g. when you 
provide a single 
`T1::operator==(const T2&)`, you get both `T1 == T2` and `T2 == T1`, the same 
applies to `operator<=>()`.
Now you need to write at most 2 functions to get all 6 comparisons between 
convertible types, and 2 functions to get all 12 comparisons between 
non-convertible types.

#### Comparison categories

The Standard provides three comparison categories(which doesn't prevent you from 
having your own one).
`strong_ordering` implies that exactly one of `a < b`, `a > b`, `a == b` must be 
true and if `a == b` then `f(a) == f(b)`.
`weak_ordering` implies that exactly one of `a < b`, `a > b`, `a == b` must  be 
true and if `a == b` then `f(a)` can be *not* equal to `f(b)`. Such elements are
equivalent but not equal.
`partial_ordering` means that none of `a < b`, `a > b`, `a == b` might  
be true and if `a == b` then `f(a)` can be not equal to `f(b)`. That is, some 
elements may be incomparable.
Important note here is that `f()` denotes a function that accesses only *salient*
attributes. For example, `std::vector<int>` is strongly ordered despite that
two vectors with the same values can have different capacity. Here, capacity is 
not a salient attribute. Example of a weakly ordered type is `CaseInsensitiveString`,
it can store original string as-is but compare in a case-insensitive way.
Example of a partially ordered type is `float/double` because
`NaN` is not comparable to any other value. These categories form hierarchy, i.e., `strong_ordering`
can be converted to `weak_ordering` and `partial_ordering`, and `weak_ordering`
can be converted to `partial_ordering`.

#### Defaulted comparisons

Comparisons could be defaulted just like special member functions. In such case 
they operate in a member-wise fashion by comparing all underlying non-static data 
members with their corresponding operators. Defaulted `operator<=>()` also 
declares defaulted `operator==()`(if there was none),
so you can write  `auto operator<=>(const T&) const = default;` and get all six 
comparison operations with member-wise semantics.

```cpp
template<typename T1, typename T2>
void TestComparisons(T1 a, T2 b)
{
    (a < b), (a <= b), (a > b), (a >= b), (a == b), (a != b);
}

struct S2
{
    int a;
    int b;
};

struct S1
{
    int x;
    int y;
    // support homogeneous comparisons
    auto operator<=>(const S1&) const = default;
    // this is required because there's operator==(const S2&) which prevents
    // implicit declaration of defaulted operator==()
    bool operator==(const S1&) const = default;

    // support heterogeneous comparisons
    std::strong_ordering operator<=>(const S2& other) const
    {
        if (auto cmp = x <=> other.a; cmp != 0)
            return cmp;
        return y <=> other.b;
    }

    bool operator==(const S2& other) const
    {
        return (*this <=> other) == 0;
    }
};

TestComparisons(S1{}, S1{});
TestComparisons(S1{}, S2{});
TestComparisons(S2{}, S1{});
```

Implicitly declared `operator==()` has the same signature as `operator<=>()`
except that return type is `bool`.

```cpp
template<typename T>
struct X
{
    friend constexpr std::partial_ordering operator<=>(X, X) requires(sizeof(T) != 1) = default;
    // implicitly declares:
    // friend constexpr bool operator==(X, X) requires(sizeof(T) != 1) = default;

    [[nodiscard]] virtual std::strong_ordering operator<=>(const X&) const = default;
    // implicitly declares:
    //[[nodiscard]] virtual bool operator==(const X&) const = default; 
};
```

Deduced comparison category is the weakest one of type's members.

```cpp
struct S3{
    int x;      // int-s are strongly ordered
    double d;   // but double-s are partially ordered
    // thus, the resulting category is std::partial_ordering
    auto operator<=>(const S3&) const = default;
};
static_assert(std::is_same_v<decltype(S3{} <=> S3{}), std::partial_ordering>);
```

They must be members or friends and only friends can take by-value.

```cpp
struct S4
{
    int x;
    int y;
    // member version must have op(const T&) const; form
    auto operator<=>(const S3&) const = default;

    // friend version can take arguments by const-reference or by-value
    // friend auto operator<=>(const S3&, const S3&) = default;
    // friend auto operator<=>(S3, S3) = default;
};
```

Can be out-of-class defaulted, just like special member functions.

```cpp
struct S5
{
    int x;
    std::strong_ordering operator<=>(const S5&) const;
    bool operator==(const S5&) const;
};

std::strong_ordering S5::operator<=>(const S5&) const = default;
bool S5::operator==(const S5&) const = default;
```

Defaulted `operator<=>()` uses `operator<=>()` of class members or
their ordering can be synthesized using existing `Member::operator==()` and `Member::operator<()`.
Note that it works only for members and not for the class itself,
existing `T::operator<()` is never used in defaulted `T::operator<=>()`.

```cpp
// not in our immediate control
struct Legacy
{
    bool operator==(Legacy const&) const;
    bool operator<(Legacy const&) const;
};

struct S6
{
    int x;
    Legacy l;
    // deleted because Legacy doesn't have operator<=>(), comparison category
    // can't be deduced
    auto operator<=>(const S6&) const = default;
};

struct S7
{
    int x;
    Legacy l;

    std::strong_ordering operator<=>(const S7& rhs) const = default;
    /*
    Since comparison category is provided explicitly, ordering can be
    synthesized using operator<() and operator==(). They must return exactly
    `bool` for this to work. It will work for weak and partial ordering as well.
    
    Here's an example of synthesized operator<=>():
    std::strong_ordering operator<=>(const S7& rhs) const
    {
        // use operator<=>() for int
        if(auto cmp = x <=> rhs.x; cmp != 0) return cmp;

        // synthesize ordering for Legacy using operator<() and operator==()
        if(l == rhs.l) return std::strong_ordering::equal;
        if(l < rhs.l) return std::strong_ordering::less;
        return std::strong_ordering::greater;
    }
    */
};

struct NoEqual
{
    bool operator<(const NoEqual&) const = default;
};

struct S8
{
    NoEqual n;
    // deleted, NoEqual doesn't have operator<=>()
    // auto operator<=>(const S8&) const = default;

    // deleted as well because NoEqual doesn't have operator==()
    std::strong_ordering operator<=>(const S8&) const = default;
};

struct W
{
    std::weak_ordering operator<=>(const W&) const = default;
};

struct S9
{
    W w;
    // ask for strong_ordering but W can provide only weak_ordering, this will
    // yield an error during instantiation
    std::strong_ordering operator<=>(const S9&) const = default;
    void f()
    {
        (S9{} <=> S9{});    // error
    }
};
```

`union` and reference members are not supported.

```cpp
struct S4
{
    int& r;
    // deleted because of reference member
    auto operator<=>(const S4&) const = default;
};
```

---

## Lambda expressions {#lambda-features}

### Allow lambda-capture `[=, this]` {#lambda-this}

When captured implicitly, `this` is always captured by-reference, even with `[=]`.
To remove this confusion, C++20 deprecates such behavior and allows more explicit
`[=, this]`:

```cpp
struct S{
    void f(){
        [=]{};          // captures this by reference, deprecated since C++20
        [=, *this]{};   // OK since C++17, captures this by value
        [=, this]{};    // OK since C++20, captures this by reference
    }
};
```

---

### Template parameter list for generic lambdas {#lambda-templ-params}

Sometimes generic lambdas are too generic. C++20 allows to use familiar
template function syntax to introduce type names directly.

```cpp
// lambda that expect std::vector<T>
// until C++20:
[](auto vector){
    using T =typename decltype(vector)::value_type;
    // use T
};
// since C++20:
[]<typename T>(std::vector<T> vector){
    // use T
};

// access argument type
// until C++20
[](const auto& x){
    using T = std::decay_t<decltype(x)>;
    // using T = decltype(x); // without decay_t<> it would be const T&, so
    T copy = x;               // copy would be a reference type
    T::static_function();     // and these wouldn't work at all
    using Iterator = typename T::iterator;
};
// since C++20
[]<typename T>(const T& x){
    T copy = x;
    T::static_function();
    using Iterator = typename T::iterator;
};

// perfect forwarding
// until C++20:
[](auto&&... args){
    return f(std::forward<decltype(args)>(args)...);
};
// since C++20:
[]<typename... Ts>(Ts&&... args){
    return f(std::forward<Ts>(args)...);
};

// and of course you can mix them with auto-parameters
[]<typename T>(const T& a, auto b){};
```

---

### Lambdas in unevaluated contexts {#lambda-uneval-ctx}

Lambda expressions can be used in unevaluated contexts, such as `sizeof()`,
`typeid()`, `decltype()`, etc. Here are some key points for this feature, for a
more real-world example see [Default constructible and assignable stateless 
lambdas](#lambda-def-ctor).

The main principle is that lambdas have a unique unknown type, two lambdas and 
their types are never equal.

```cpp
using L = decltype([]{});   // lambdas have no linkage
L PublicApi();              // L can't be used for external linkage

// in template , two different declarations
template<class T> void f(decltype([]{}) (*s)[sizeof(T)]);
template<class T> void f(decltype([]{}) (*s)[sizeof(T)]);

// again, lambda types are never equivalent
static decltype([]{}) f();
static decltype([]{}) f(); // error, return type mismatch

static decltype([]{}) g();
static decltype(g()) g(); // okay, redeclaration

// each specialization has its own lambda with unique type
template<typename T>
using R = decltype([]{});

static_assert(!std::is_same_v<R<int>, R<char>>);

// Lambda-based SFINAE and constraints are not supported, it just fails
template <class T>
auto f(T) -> decltype([]() { T::invalid; } ());
void f(...);

template<typename T>
void g(T) requires requires{
    [](){typename T::invalid x;}; }
{}
void g(...){}

f(0);  // error
g(0);  // error
```

In the following example, `f()` increments the same counter in both
translation units because `inline` function behaves as if there's 
only one definition of it. However, `g_s` violates ODR because despite that 
there's only one definition of it, there are still multiple declarations which
are different because there are two different lambdas in `a.cpp` and `b.cpp`,
thus, `S` has different non-type template argument:

```cpp
// a.h
template<typename T>
int counter(){
    static int value{};
    return value++;
}

inline int f(){
    return counter<decltype([]{})>();
}

template<auto> struct S{ void call(){} };
// cast lambda to pointer
inline S<+[]{}> g_s;

// a.cpp
#include "a.h"
auto v = f();
g_s.call();

// b.cpp
#include "a.h"
auto v = f();
g_s.call();
```

---

### Default constructible and assignable stateless lambdas {#lambda-def-ctor}

In C++20 stateless lambdas are default constructible and assignable which 
allows to use a type of a lambda to construct/assign it later. With
[Lambdas in unevaluated contexts](#lambda-uneval-ctx) we can get a type of a
lambda with `decltype()`
and create a variable of that type later:

```cpp
auto greater = [](auto x,auto y)
{
    return x > y;
};
// requires default constructible type
std::map<std::string, int, decltype(greater)> map;
auto map2 = map;    // requires default assignable type
```

Here, `std::map` takes a comparator type to instantiate it later. While we could
get a lambda type in C++17, it was not possible to instantiate it because 
lambdas were not default constructible.

---

### Pack expansion in lambda init-capture {#lambda-pack-exp}

C++20 simplifies capturing parameter packs in lambdas. Until C++20 they can be
captured by-value, by-reference or do some tricks with `std::tuple` if
we want to move the pack. Now it's much easier, we can create *init-capture pack*
and initialize it with the pack we want to capture. It's not limited to 
`std::move` or `std::forward`, any function can be
applied to pack elements.

```cpp
void g(int, int){}

// C++17
template<class F, class... Args>
auto delay_apply(F&& f, Args&&... args) {
    return [f=std::forward<F>(f), tup=std::make_tuple(std::forward<Args>(args)...)]()
            -> decltype(auto) {
        return std::apply(f, tup);
    };
}

// C++20
template<typename F, typename... Args>
auto delay_call(F&& f, Args&&... args) {
    return [f = std::forward<F>(f), ...f_args=std::forward<Args>(args)]()
            -> decltype(auto) {
        return f(f_args...);
    };
}

void f(){
    delay_call(g, 1, 2)();
}
```

---

## Constant expressions {#constexpr-features}

### Immediate functions(`consteval`) {#consteval}

While `constexpr` implies that function *can* be evaluated at compile-time,
`consteval` specifies that function *must* be evaluated at compile-time(only).
`virtual` functions are allowed to be `consteval` but they can override and be
overridden by another `consteval` function only, i.e., mix of `consteval` and
non-`consteval` is not allowed.
Destructors and allocation/deallocation functions can't be `consteval`.

```cpp
consteval int GetInt(int x){
    return x;
}

constexpr void f(){
    auto x1 = GetInt(1);
    constexpr auto x2 = GetInt(x1); // error x1 is not a constant-expression
}
```

---

### `constexpr` virtual function {#constexpr-virtual}

Virtual functions can now be `constexpr`. `constexpr` function can override
non-`constexpr` one and vice-versa.


```cpp
struct Base{
    constexpr virtual ~Base() = default;
    virtual int Get() const = 0;    // non-constexpr
};

struct Derived1 : Base{
    constexpr int Get() const override {
        return 1;
    }
};

struct Derived2 : Base{
    constexpr int Get() const override {
        return 2;
    }
};

constexpr auto GetSum(){
    const Derived1 d1;
    const Derived2 d2;
    const Base* pb1 = &d1;
    const Base* pb2 = &d2;

    return pb1->Get() + pb2->Get();
}

static_assert(GetSum() == 1 + 2);   // evaluated at compile-time
``` 

---

### `constexpr` try-catch blocks {#constexpr-try-catch}

*try-catch* blocks are now allowed inside `constexpr` functions but `throw` is not,
so, the `catch` block is simply ignored. This can be useful, for example, in
combination with `constexpr new`, we can have single function that works at
run/compile time:

```cpp
constexpr void f(){
    try{
        auto p = new int;
        // ...
        delete p;
    }
    catch(...){     // ignored at compile-time
        // ...
    }
}
```

---

### `constexpr` `dynamic_cast` and polymorphic `typeid` {#constexpr-dyn-cast}

Since virtual functions can now be `constexpr`, there's no reason not to allow
`dynamic_cast` and polymorphic `typeid` in `constexpr`. Unfortunately,
`std::type_info` has
no `constexpr` members yet so there's a little use of it now(thanks to Peter
Dimov for clarifying this for me).


```cpp
struct Base1{
    virtual ~Base1() = default;
    constexpr virtual int get() const = 0;
};

struct Derived1 : Base1{
    constexpr int get() const override {
        return 1;
    }
};

struct Base2{
    virtual ~Base2() = default;
    constexpr virtual int get() const = 0;
};

struct Derived2 : Base2{
    constexpr int get() const override {
        return 2;
    }
};

template<typename Base, typename Derived>
constexpr auto downcasted_get(){
    const Derived d;
    const Base& upcasted = d;
    const auto& downcasted = dynamic_cast<const Derived&>(upcasted);

    return downcasted.get();
}

static_assert(downcasted_get<Base1, Derived1>() == 1);
static_assert(downcasted_get<Base2, Derived2>() == 2);

// compile-time error, cannot cast Derived1 to Base2
static_assert(downcasted_get<Base2, Derived1>() == 1);
```

---

### Changing the active member of a `union` inside `constexpr` {#constexpr-union}

Another relaxation for constant expressions. One can change an active member of
a `union` but can't read an inactive member since it's UB and UB is not allowed
in `constexpr` context.

```cpp
union Foo {
  int i;
  float f;
};

constexpr int f() {
  Foo foo{};
  foo.i = 3;    // i is an active member
  foo.f = 1.2f; // valid since C++20, f becomes an active member

//   return foo.i;  // error, reading inactive union member
  return foo.f;
}
```

---

### `constexpr` allocations {#constexpr-alloc}

C++20 lays foundation for `constexpr` containers. First, it allows `constexpr` 
and 
even `virtual constexpr` destructors for *literal* types(types that can be used as 
a `constexpr` variable). Second, it allows calls to 
`std::allocator<T>::allocate()` and `new-expression` which 
results in a call to one of the global `operator new` if allocated storage is 
deallocated at compile time. That is, memory can be allocated at compile-time
but it must be freed at compile-time also. This creates a bit of friction
if final data has to be used at run-time. There's no choice but to store it in
some non-allocating container like `std::array` and get compile-time value twice:
first, to get its size, and second, to actually copy it(thanks to 
**arthur-odwyer**, 
**beached** and **luke** from [cpplang slack](cpplang.slack.com) for explaining 
this to me):

```cpp
constexpr auto get_str()
{
    std::string s1{"hello "};
    std::string s2{"world"};
    std::string s3 = s1 + s2;
    return s3;
}

constexpr auto get_array()
{
    constexpr auto N = get_str().size();
    std::array<char, N> arr{};
    std::copy_n(get_str().data(), N, std::begin(arr));
    return arr;
}

static_assert(!get_str().empty());

// error because it holds data allocated at compile-time
constexpr auto str = get_str();

// OK, string is stored in std::array<char>
constexpr auto result = get_array();
```

---

### Trivial default initialization in `constexpr` functions {#constexpr-trivial-def-init}

In C++17 `constexpr` constructor, among other requirements, must initialize all
non-static data members. This rule has been removed in C++20. But, because UB is 
not allowed in `constexpr` context, you can't read from such uninitialized members,
only write to them:

```cpp
struct NonTrivial{
    bool b = false;
};

struct Trivial{
    bool b;
};

template <typename T>
constexpr T f1(const T& other) {
    T t;        // default initialization
    t = other;
    return t;
}

template <typename T>
constexpr auto f2(const T& other) {
    T t;
    return t.b;
}

void test(){
    constexpr auto a = f1(Trivial{});   // error in C++17, OK in C++20
    constexpr auto b = f1(NonTrivial{});// OK

    constexpr auto c = f2(Trivial{}); // error, uninitialized Trivial::b is used
    constexpr auto d = f2(NonTrivial{}); // OK
}
```

---

### Unevaluated `asm`-declaration in `constexpr` functions {#constexpr-asm}

asm-declaration now can appear inside `constexpr` function in case it's not 
evaluated at compile-time. This allows to have both compile and run time(with 
asm now) code inside a single function:

```cpp
constexpr int add(int a, int b){
    if (std::is_constant_evaluated()){
        return a + b;
    }
    else{
        asm("asm magic here");
        //...
    }
}
```

---

### std::is_constant_evaluated() {#is-const-eval}

With `std::is_constant_evaluated()` you can check whether current invocation 
occurs within a constant-evaluated context. I would like to say "during 
compile-time" but, as the authors said, "C++ doesn't make a clear distinction 
between 
compile-time and run-time". Instead, C++20 declares a [list](https://en.cppreference.com/w/cpp/types/is_constant_evaluated)
of expressions that are *manifestly constant-evaluated* and this function returns
`true` during their evaluation and `false` otherwise.  
Be careful not to use this function directly in such *manifestly constant-evaluated*
expressions(e.g. `if constexpr`, array size, template arguments, etc.).
By definition, in such cases 
`std::is_constant_evaluated()` returns `true` even if the enclosing function
is not constant evaluated. Thanks to user **destroyerrocket** from [/r/cpp](https://www.reddit.com/r/cpp/)
for bringing up this issue.


```cpp
constexpr int GetNumber(){
    if(std::is_constant_evaluated()){   // should not be `if constexpr`
        return 1;
    }
    return 2;
}

constexpr int GetNumber(int x){
    if(std::is_constant_evaluated()){   // should not be `if constexpr`
        return x;
    }
    return x+1;
}

void f(){
    constexpr auto v1 = GetNumber();
    const auto v2 = GetNumber();

    // initialization of a non-const variable, not constant-evaluated
    auto v3 = GetNumber();

    assert(v1 == 1);
    assert(v2 == 1);
    assert(v3 == 2);

    constexpr auto v4 = GetNumber(1);
    int x = 1;

    // x is not a constant-expression, not constant-evaluated
    const auto v5 = GetNumber(x);

    assert(v4 == 1);
    assert(v5 == 2);    
}

// pathological examples
// always returns `true`
constexpr bool IsInConstexpr(int){
    if constexpr(std::is_constant_evaluated()){ // always `true`
        return true;
    }
    return false;
}

// always returns `sizeof(int)`
constexpr std::size_t GetArraySize(int){
    int arr[std::is_constant_evaluated()];  // always int arr[1];
    return sizeof(arr);
}

// always returns `1`
constexpr std::size_t GetStdArraySize(int){
    std::array<int, std::is_constant_evaluated()> arr;  // std::array<int, 1>
    return arr.size();
}
```

---

## Aggregates {#aggregates}

### Prohibit aggregates with user-declared constructors {#aggr-no-ctor}

Now aggregate types
can't have *user-declared* constructors. Previously, aggregates were allowed to
have only deleted or defaulted constructors. That 
resulted in a weird behavior for aggregates with defaulted/deleted constructors
(they're *user-declared* but not *user-provided*).

```cpp
// none of the types below are an aggregate in C++20
struct S{
    int x{2};
    S(int) = delete; // user-declared ctor
};

struct X{
    int x;
    X() = default;  // user-declared ctor
};

struct Y{
    int x;
    Y();            // user-provided ctor
};

Y::Y() = default;

void f(){
    S s(1);     // always an error
    S s2{1};    // OK in C++17, error in C++20, S is not an aggregate now
    X x{1};     // OK in C++17, error in C++20
    Y y{2};     // always an error
}
```

---

### Class template argument deduction for aggregates {#ctad-aggr}

In C++17 to use aggregates with CTAD we need explicit deduction guides,
that's unnecessary now:

```cpp
template<typename T, typename U>
struct S{
    T t;
    U u;
};
// deduction guide was needed in C++17
// template<typename T, typename U>
// S(T, U) -> S<T,U>;

S s{1, 2.0};    // S<int, double>
```

CTAD isn't involved when there are user-provided deduction guides:

```cpp
template<typename T>
struct MyData{
    T data;
};
MyData(const char*) -> MyData<std::string>;

MyData s1{"abc"};   // OK, MyData<std::string> using deduction guide
MyData<int> s2{1};  // OK, explicit template argument
MyData s3{1};       // Error, CTAD isn't involved
```

Can deduce array types:

{% raw %}
```cpp
template<typename T, std::size_t N>
struct Array{
    T data[N];
};

Array a{{1, 2, 3}}; // Array<int, 3>, notice additional braces
Array str{"hello"}; // Array<char, 6>
```
{% endraw %}

Brace elision doesn't work for dependent non-array types or array types of
dependent bound.

{% raw %}
```cpp
template<typename T, typename U>
struct Pair{
    T first;
    U second;
};

template<typename T, std::size_t N>
struct A1{
    T data[N];
    T oneMore;
    Pair<T, T> p;
};

template<typename T>
struct A2{
    T data[3];
    T oneMore;
    Pair<int, int> p;
};

// A1::data is an array of dependent bound and A1::p is a dependent type, thus,
// no brace elision for them
A1 a1{{1,2,3}, 4, {5, 6}};  // A1<int, 3>
// A2::data is an array of non-dependent bound and A1::p is a non-dependent type,
// thus, brace elision works
A2 a2{1, 2, 3, 4, 5, 6};    // A2<int>
```
{% endraw %}

Works with pack expansions.
Trailing aggregate element that is a pack expansion corresponds to all 
remaining elements:

```cpp
template<typename... Ts>
struct Overload : Ts...{
    using Ts::operator()...;
};
// no need for deduction guide anymore

Overload p{[](int){
        std::cout << "called with int";
    }, [](char){
        std::cout << "called with char";
    }
};     // Overload<lambda(int), lambda(char)>
p(1);   // called with int
p('c'); // called with char
```

Non-trailing element that is a pack expansions corresponds to no elements:

```cpp
template<typename T, typename...Ts>
struct Pack : Ts... {
    T x;
};

// can deduce only the first element
Pack p1{1};         // Pack<int>
Pack p2{[]{}};      // Pack<lambda()>
Pack p3{1, []{}};   // error
```

Number of elements in the pack is deduced only once but types should match 
exactly if repeated:

```cpp
struct A{};
struct B{};
struct C{};
struct D{
    operator C(){return C{};}
};

template<typename...Ts>
struct P : std::tuple<Ts...>, Ts...{
};

P{std::tuple<A, B, C>{}, A{}, B{}, C{}}; // P<A, B, C>

// equivalent to the above, since pack elements were deduced for
// std::tuple<A, B, C> there's no need to repeat their types
P{std::tuple<A, B, C>{}, {}, {}, {}}; // P<A, B, C>

// since we know the whole P<A, B, C> type after std::tuple initializer, we can
// omit trailing initializers, elements will be value-initialized as usual
P{std::tuple<A, B, C>{}, {}, {}}; // P<A, B, C>

// error, pack deduced from first initializer is <A, B, C> but got <A, B, D> for
// the trailing pack, implicit conversions are not considered
P{std::tuple<A, B, C>{}, {}, {}, D{}};
```

---

### Parenthesized initialization of aggregates {#aggr-paren-init}

Parenthesized initialization of aggregates now works in the same way as braced 
initialization except that narrowing conversions are permitted, designated 
initializers are not allowed, no lifetime extension for temporaries and no brace 
elision. Elements without initializer are value-initialized. This allows
seamless usage of factory functions like `std::make_unique<>()/emplace()` with
aggregates.

```cpp
struct S{
    int a;
    int b = 2;
    struct S2{
        int d;
    } c;
};

struct Ref{
    const int& r;
};

int GetInt(){
    return 21;
}

S{0.1}; // error, narrowing
S(0.1); // OK

S{.a=1}; // OK
S(.a=1); // error, no designated initializers

Ref r1{GetInt()}; // OK, lifetime is extended
Ref r2(GetInt()); // dangling, lifetime is not extended

S{1, 2, 3}; // OK, brace elision, same as S{1,2,{3}}
S(1, 2, 3); // error, no brace elision

// values without initializers take default values or value-initialized(T{})
S{1}; // {1, 2, 0}
S(1); // {1, 2, 0}

// make_unique works now
auto ps = std::make_unique<S>(1, 2, S::S2{3});

// arrays are also supported
int arr1[](1, 2, 3);
int arr2[2](1); // {1, 0}
```

---

## Non-type template parameters {#nttp}

### Class types in non-type template parameters {#class-types-nttp}

Non-type template parameters now can be of literal class types(
types that can be used as a `constexpr` variable)
with all bases and non-static members being `public` and non-`mutable`(literally, 
there should be no `mutable` specifier). Instances of such classes are stored as
`const` objects and you can even call their member functions. There's a new
kind of non-type template parameter: *placeholder for a deduced
class type*. In the example below, `fixed_string` is a template name, not a type 
name, but we can use it to declare template parameter `template<fixed_string S>`.
In such a case, the compiler will deduce template arguments for `fixed_string` 
before instantiating `f<>()` using an invented declaration in the form of 
`T x = template-argument;`. Here's how it can be used to create a simple
compile-time string class:

```cpp
template<std::size_t N>
struct fixed_string{
    constexpr fixed_string(const char (&s)[N+1]) {
        std::copy_n(s, N + 1, str);
    }
    constexpr const char* data() const {
        return str;
    }
    constexpr std::size_t size() const {
        return N;
    }

    char str[N+1];
};

template<std::size_t N>
fixed_string(const char (&)[N])->fixed_string<N-1>;

// user-defined literals are also supported
template<fixed_string S>
constexpr auto operator""_cts(){
    return S;
}

// N for `S` will be deduced
template<fixed_string S>
void f(){
    std::cout << S.data() << ", " << S.size() << '\n';
}

f<"abc">(); // abc, 3
constexpr auto s = "def"_cts;
f<s>();     // def, 3
```

---

### Generalized non-type template parameters {#nttp-gen}

Non-type template parameters are generalized to so-called *structural* types.
Structural type is one of:
- scalar type(arithmetic, pointer, pointer-to-member, enumeration, `std::nullptr_t`)
- lvalue reference
- literal class type with the following properties: all base classes and
non-static data members are public and non-`mutable`, and their types are
structural or array types.

This allows usage of floating-point and class types as a template parameters:

{% raw %}
```cpp
template<auto T>    // placeholder for any non-type template parameter
struct X{};

template<typename T, std::size_t N>
struct Arr{
    T data[N];
};

X<5> x1;
X<'c'> x2;
X<1.2> x3;
// with the help of CTAD for aggregates
X<Arr{{1,2,3}}> x4; // X<Arr<int, 3>>
X<Arr{"hi"}> x5;    // X<Arr<char, 3>>
```
{% endraw %}

Interesting moment here is that non-type template arguments are compared not 
with their `operator==()` but in a bitwise-*like* manner(the exact
rules are [here](https://en.cppreference.com/w/cpp/language/template_parameters#Template_argument_equivalence)). That is, their
bit representation is used for comparison. `union`s are exceptions because 
the compiler can track their active members. Two unions are equal if they both
have no active member or have the same active member with equal value.

```cpp
template<auto T>
struct S{};

union U{
    int a;
    int b;
};

enum class E{
    A = 0,
    B = 0
};

struct C{
    int x;
    bool operator==(const C&) const{    // never equal
        return false;
    }
};

constexpr C c1{1};
constexpr C c2{1};
assert(c1 != c2);                           // not equal using operator==()
assert(memcmp(&c1, &c2, sizeof(C)) == 0);   // but equal bitwise
// thus, equal at compile-time, operator==() is not used
static_assert(std::is_same_v<S<c1>, S<c2>>);

constexpr E e1{E::A};
constexpr E e2{E::B};
// equal bitwise, enum's identity isn't taken into account
assert(memcmp(&e1, &e2, sizeof(E)) == 0);
static_assert(std::is_same_v<S<e1>, S<e2>>); // thus, equal at compile-time

constexpr U u1{.a=1};
constexpr U u2{.b=1};
// equal bitwise but have different active members(a vs. b)
assert(memcmp(&u1, &u2, sizeof(U)) == 0);
// thus, not equal at compile-time
static_assert(!std::is_same_v<S<u1>, S<u2>>);
```

---

## Structured bindings {#struct-bindings}

### Lambda capture and storage class specifiers for structured bindings {#structbind-specs}

Structured bindings are allowed to have `[[maybe_unused]]` attribute, `static`
and `thread_local` specifiers. Also, it's possible now to capture them by-value or
by-reference in lambdas. Note that bound bit-fields can be captured only
by-value.

```cpp
struct S{
    int a: 1;
    int b: 1;
    int c;
};

static auto [A,B,C] = S{};

void f(){
    [[maybe_unused]] thread_local auto [a,b,c] = S{};
    auto l = [=](){
        return a + b + c;
    };

    auto m = [&](){
        // error, can't capture bit-fields 'a' and 'b' by-reference
        // return a + b + c;
        return c;
    };
}

```

---

### Relaxing the structured bindings customization point finding rules {#fix-structbind-cp}

One of ways for a type to be decomposed for structured bindings is through
a tuple-like API. It consists of three "functions": `std::tuple_element`, 
`std::tuple_size` and two options for `get`: `e.get<I>()` or `get<I>(e)` where the 
first has priority over the second. That is, the member `get()` is preferred over
non-member one. Imagine a
type that has `get()` but it's not for a tuple-like API, for example 
`std::shared_ptr::get()`. Such a type can't be decomposed because the compiler
will try to
use member `get()` and it won't work. Now this rule has been fixed in a way that 
the member version is
preferred only if it's a template and its first template parameter is a non-type
template parameter.

```cpp
struct X : private std::shared_ptr<int>{
    std::string payload;
};

// due to new rules, this function is used instead of std::shared_ptr<int>::get
template<int N>
std::string& get(X& x) {
    if constexpr(N==0) return x.payload;
}

namespace std {
    template<>
    class tuple_size<X> 
        : public std::integral_constant<int, 1>
    {};
    
    template<>
    class tuple_element<0, X> {
    public:
        using type = std::string;
    };
}

void f(){
    X x;
    auto& [payload] = x;
}
```

---

### Allow structured bindings to accessible members {#fix-structbind-access}

This fix allows structured bindings not only to `public` members but to
*accessible* members in the context of structured binding declaration.

```cpp
struct A {
    friend void foo();
private:
    int i;
};

void foo() {
    A a;
    auto x = a.i;   // OK
    auto [y] = a;   // Ill-formed until C++20, now OK
}
```

---

## Range-based for-loop {#range-based-for}

### init-statements for range-based for-loop {#init-range-for}

Similar to if-statement, range-based for-loop now can have init-statement. It
can be used to avoid dangling references:

```cpp
class Obj{
    std::vector<int>& GetItems();
};

Obj GetObj();

// dangling reference, lifetime of Obj return by GetObj() is not extended
for(auto x : GetObj().GetCollection()){
    // ...
}

// OK
for(auto obj = GetObj(); auto item : obj.GetCollection()){
    // ...
}

// also can be used to maintain index
for(std::size_t i = 0; auto& v : collection){
    // use v...
    i++;
}
```

---

### Relaxing the range-based for-loop customization point finding rules {#fix-range-for-cp}

This one is similar to [structured bindings customization point fix](#fix-structbind-cp).
To iterate over a range, range-based for-loop needs either free or member
`begin`/`end` functions.
Old rules worked in a way that if *any* member(function or variable) named
`begin`/`end` was found then the compiler would try to use member functions.
This creates a problem for types that have a member `begin` but no `end` or vice
versa. Now member functions are used only if both names exist, otherwise free
functions are used.

```cpp
struct X : std::stringstream {
  // ...
};

std::istream_iterator<char> begin(X& x){
    return std::istream_iterator<char>(x);
}

std::istream_iterator<char> end(X& x){
    return std::istream_iterator<char>();
}

void f(){
    X x;
    // X has member with name `end` inherited from std::stringstream
    // but due to new rules free begin()/end() are used
    for (auto&& i : x) {
        // ...
    }
}
```

---

## Attributes {#attributes}

### `[[likely]]` and `[[unlikely]]` {#attr-likely}

`[[likely]]` and `[[unlikely]]` attributes give a hint to the
compiler about likeliness of execution path so it can better optimize the code.
They can be applied to statements(e.g. `if/else`-statements, loops) or
labels(`case/default`).

```cpp
int f(bool b){
    if(b) [[likely]] {
        return 12;
    }
    else{
        return 10;
    }
}
```

---

### `[[no_unique_address]]` {#attr-no-uniq-addr}

`[[no_unique_address]]` can be applied to a non-static non-bitfield data member to 
indicate that it doesn't need a unique address. In practice, it's applied to
a potentially empty data member and the compiler can optimize it to occupy no
space(like empty base optimization for members). Such a member can share the
address of another member or base class.

```cpp
struct Empty{};

template<typename T>
struct Cpp17Widget{
    int i;
    T t;
};

template<typename T>
struct Cpp20Widget{
    int i;
    [[no_unique_address]] T t;
};

static_assert(sizeof(Cpp17Widget<Empty>) > sizeof(int));
static_assert(sizeof(Cpp20Widget<Empty>) == sizeof(int));
```

---

### `[[nodiscard]]` with message {#discard-msg}

Like `[[deprecated("reason")]]`, `nodiscard` now can have a reason too.

```cpp
// test whether it's supported
static_assert(__has_cpp_attribute(nodiscard) == 201907L);

[[nodiscard("Don't leave me alone")]]
int get();

void f(){
    get(); // warning: ignoring return value of function declared with 
           // 'nodiscard' attribute: Don't leave me alone
}
```

---

### `[[nodiscard]]` for constructors {#fix-nodiscard-ctor}

This fix explicitly allows applying `[[nodiscard]]` to constructors(compilers
were not required to support it prior to C++20).

```cpp
struct resource{
    // empty resource, no harm if discarded
    resource() = default;
    
    [[nodiscard("don't discard non-empty resource")]]
    resource(int fd);
};

void f(){
    resource{};     // OK
    resource{1};    // warning
}
```

---

## Character encoding {#encoding}

### `char8_t` {#char8t}

C++17 introduced the `u8` character literal for UTF-8 string but its type was plain 
`char`. The inability to distinguish encoding by a type resulted in a code that
had to use various tricks to handle different encodings. A new `char8_t` type was 
introduced to represent UTF-8 characters. It has the same size, signedness, 
alignment, etc, as `unsigned char` but it's a distinct type, not an alias.

```cpp
void HandleString(const char*){}
// distinct function name is required to handle UTF-8 in C++17
void HandleStringUTF8(const char*){}
// now it can be done using convenient overload
void HandleString(const char8_t*){}

void Cpp17(){
    HandleString("abc");        // char[4]
    HandleStringUTF8(u8"abc");  // C++17: char[4] but UTF-8, 
                                // C++20: error, type is char8_t[4]
}

void Cpp20(){
    HandleString("abc");    // char
    HandleString(u8"abc");  // char8_t
}
```

---

### Stronger Unicode requirements {#stronger-unicode}


Types `char16_t` and `char32_t` are now explicitly required to represent UTF-16
and 
UTF-32 string literals correspondingly. Universal character names(`\Unnnnnnnn` and 
`\uNNNN`) must correspond to ISO/IEC 10646 code points (0x0 - 0x10FFFF inclusive) 
and not to a surrogate code points (0xD800 - 0xDFFF inclusive), otherwise the 
program is ill-formed.

```cpp
char32_t c{'\U00110000'};   // error: invalid universal character
```

---

## Sugar {#sugar}

### Designated initializers {#designated-init}

Now it's possible to initialize specific(designated) aggregate members and skip 
others. Unlike C, initialization order must be the same as in aggregate
declaration:

```cpp
struct S{
    int x;
    int y{2};
    std::string s;
};
S s1{.y = 3};   // {0, 3, {}}
S s2 = {.x = 1, .s = "abc"};    // {1, 2, {"abc"}}
S s3{.y = 1, .x = 2};   // Error, x should be initialized before y
```

---

### Default member initializers for bit-fields {#bitfield-def-init}

Until C++20, to provide default value for a bit-field one had to create a default
constructor, now that can be achieved using convenient default member
initialization syntax:

```cpp
// until C++20:
struct S{
    int a : 1;
    int b : 1;
    S() : a{0}, b{1}{}
};

// since C++20:
struct S{
    int a : 1 {0},
    int b : 1 = 1;
};
```

---

### More optional `typename` {#less-typename}

`typename` can be omitted in contexts where nothing but a type name can 
appear(type in casts, return type, type aliases, member type, argument type of a member function, etc.):

```cpp
template <class T>
T::R f();  // OK, return type of a function declaration at global scope

template <class T>
void f(T::R);   // Ill-formed (no diagnostic required), attempt to declare a
                // void variable template

template<typename T>
struct PtrTraits{
    using Ptr = void*;
};

template <class T>
struct S {
  using Ptr = PtrTraits<T>::Ptr;  // OK, in a defining-type-id
  T::R f(T::P p) {                // OK, class scope
    return static_cast<T::R>(p);  // OK, type-id of a static_cast
  }
  auto g() -> S<T*>::Ptr; // OK, trailing-return-type

  T::SubType t;
};

template <typename T>
void f() {
  void (*pf)(T::X); // Variable pf of type void* initialized with T::X
  void g(T::X);     // Error: T::X at block scope does not denote a type
                    // (attempt to declare a void variable)
}
```

---

### Nested `inline` namespaces {#nested-inline-ns}

`inline` keyword is allowed to appear in nested namespace definitions:

```cpp
// C++20
namespace A::B::inline C{
    void f(){}
}
// C++17
namespace A::B{
    inline namespace C{
        void f(){}
    }
}
```

---

### `using enum` {#using-enum}

Scoped enumerations are great, the only problem with them is their verbose usage
(e.g. `my_enum::enum_value`). For example, in a switch-statement that checks
every possible enum value, `my_enum::` part should be repeated for each case-label.
*Using enum declaration* introduces all enumeration's names into the
current scope so they are visible as unqualified names and `my_enum::` part can
be omitted. It can be applied to unscoped enumerations and even to a single
enumerator.

```cpp
namespace my_lib {
enum class color { red, green, blue };
enum COLOR {RED, GREEN, BLUE};
enum class side {left, right};
}

void f(my_lib::color c1, my_lib::COLOR c2){
    using enum my_lib::color;   // introduce scoped enum
    using enum my_lib::COLOR;   // introduce unscoped enum
    using my_lib::side::left;   // introduce single enumerator id

    // C++17
    if(c1 == my_lib::color::red){/*...*/}
    
    // C++20
    if(c1 == green){/*...*/}
    if(c2 == BLUE){/*...*/}

    auto r = my_lib::side::right;   // qualified id is required for `right`
    auto l = left;                  // but not for `left`
}
```

---

### Array size deduction in new-expressions {#fix-arr-size}

This fix allows the compiler to deduce array size in new-expressions just like it
does for local variables.

```cpp
// before C++20
int p0[]{1, 2, 3};
int* p1 = new int[3]{1, 2, 3};  // explicit size is required

// since C++20
int* p2 = new int[]{1, 2, 3};
int* p3 = new int[]{};  // empty
char* p4 = new char[]{"hi"};
// works with parenthesized initialization of aggregates
int p5[](1, 2, 3);
int* p6 = new int[](1, 2, 3);
```

---

### Class template argument deduction for alias templates {#ctad-alias}

CTAD works with type aliases now:

```cpp
template<typename T>
using IntPair = std::pair<int, T>;

double d{};
IntPair<double> p0{1, d};   // C++17
IntPair p1{1, d};   // std::pair<int, double>
IntPair p2{1, p1};  // std::pair<int, std::pair<int, double>>
```

---

## `constinit` {#constinit}

C++ has infamous "static initialization order fiasco" when order of 
initialization of static storage variables from different translation units is 
undefined. Variables with zero/constant initialization avoid this problem because 
they are initialized at compile-time. `constinit` enforces that variable is
initialized at compile-time and unlike `constexpr` it allows non-trivial
destructors. Second use-case for `constinit` is with non-initializing 
`thread_local` declarations. In such a case, it tells the compiler that the 
variable is
already initialized, otherwise the compiler usually adds code to check and 
initialize it if required on each usage.

```cpp
struct S {
    constexpr S(int) {}
    ~S(){}; // non-trivial
};

constinit S s1{42};  // OK
constexpr S s2{42};  // error because destructor is not trivial

// tls_definitions.cpp
thread_local constinit int tls1{1};
thread_local int tls2{2};

// main.cpp
extern thread_local constinit int tls1;
extern thread_local int tls2;

int get_tls1() {
    return tls1;  // pure TLS access
}

int get_tls2() {
    return tls2;  // has implicit TLS initialization code
}
```

---

## Signed integers are two's complement {#int-twos-compl}

That is, signed integers are now guaranteed to be [two's complement](https://en.wikipedia.org/wiki/Two%27s_complement).
This removes some undefined and implementation-defined behavior because
the binary representation is fixed. Overflow for signed integers is still UB but 
these are well-defined now:

```cpp
int i1 = -1;
// left-shift for signed negative integers(previously undefined behavior)
i1 <<= 1;    // -2

int i2 = INT_MAX;
// "unrepresentable" left-shift for signed integers(previously undefined behavior)
i2 <<= 1;   // -2

int i3 = -1;
// right shift for signed negative integers, performs sign-extension(previously 
// implementation-defined)
i3 >>= 1;   // -1
int i4 = 1;
i4 >>= 1;   // 0

// "unrepresentable" conversions to signed integers(previously implementation-defined)
int i5 = UINT_MAX;  // -1
```

---

## `__VA_OPT__` for variadic macros {#va-opt}

Allows more simple handlining of variadic macros. Expands to nothing if 
`__VA_ARGS__` is empty and to its content otherwise. It's especially useful
when macro calls a function with some predefined argument(s) followed be optional
`__VA_ARGS__`. In such a case, `__VA_OPT__` allows to omit the trailing comma when
`__VA_ARGS__` are empty(thanks to Jérôme Marsaguet for bringing up this issue).

```cpp
#define LOG1(...)                   \
    __VA_OPT__(std::printf(__VA_ARGS);) \
    std::printf("\n");

LOG1();                      // std::printf("\n");
LOG1("number is %d", 12);    // std::printf("number is %d", 12); std::printf("\n");

#define LOG2(msg, ...) \
    std::printf("[" __FILE__ ":%d] " msg, __LINE__, __VA_ARGS__)
#define LOG3(msg, ...) \
    std::printf("[" __FILE__ ":%d] " msg, __LINE__ __VA_OPT__(,) __VA_ARGS__)

// OK, std::printf("[" "file.cpp" ":%d] " "%d errors.\n", 14, 0);
LOG2("%d errors\n", 0);

// Error, std::printf("[" "file.cpp" ":%d] " "No errors\n", 17, );
LOG2("No errors\n");

// OK, std::printf("[" "file.cpp" ":%d] " "No errors\n", 20);
LOG3("No errors\n");
```

---

## Explicitly defaulted functions with different exception specifications {#diff-except-spec}

This fix allows exception specification of an explicitly defaulted function to 
differ from such specification of implicitly declared function. Until C++20 such 
declarations made the program ill-formed. Now it's allowed and, of course,
the provided exception specification is the actual one. This is useful when you
want to enforce `noexcept`-ness of some operations. For example, due to
strong exception guarantee, `std::vector` *moves* its elements into a new storage
only if their move constructors are `noexcept`, otherwise elements are *copied*.
Sometimes it's desirable to allow this faster implementation even if elements
can actually throw during move. As usual, when a function marked `noexcept` throws,
`std::terminate()` is called.

```cpp
struct S1{
    // ill-formed until C++20 because implicit constructor is noexcept(true)
    S1(S1&&)noexcept(false) = default; // can throw
};

struct S2{
    S2(S2&&) noexcept = default;
    // implicitly generated move constructor would be `noexcept(false)`
    // because of `s1`, now it's enforced to be `noexcept(true)`
    S1 s1;
};

static_assert(std::is_nothrow_move_constructible_v<S1> == false);
static_assert(std::is_nothrow_move_constructible_v<S2> == true);

struct X1{
    X1(X1&&) noexcept = default;
    std::map<int, int> m;   // `std::map(std::map&&)` can throw
};

struct X2{
    // same as implicitly generated, it's `noexcept(false)` because of `std::map`
    X2(X2&&) = default;
    std::map<int, int> m;   // `std::map(std::map&&)` can throw
};

std::vector<X1> v1;
std::vector<X2> v2;
// ... at some point, `push_back()` needs to reallocate storage

// efficiently uses `X1(X1&&)` to move the elements to a new storage,
// calls `std::terminate()` if it throws
v1.push_back(X1{});

// uses `X2(const X2&)`, thus, copies, not moves elements to a new storage
v2.push_back(X2{});
```

---

## Destroying `operator delete` {#destr-delete}

C++20 introduces a class-specific `operator delete()` that takes a special
`std::destroying_delete_t` tag. In such a case, the compiler will not call the 
object's
destructor before calling `operator delete()`, it should be called manually. This 
might be useful if object members should be used to extract information needed
to free memory it occupies, for example to extract its valid size and call sized
version of `delete`.

```cpp
struct TrickyObject{
    void operator delete(TrickyObject *ptr, std::destroying_delete_t){
        // without destroying_delete_t object would have been destroyed here
        const std::size_t realSize = ptr->GetRealSizeSomehow();
        // now we need to call the destructor by-hand
        ptr->~TrickyObject();
        // and free storage it occupies
        ::operator delete(ptr, realSize);
    }
    // ...
};
```

---

## Conditionally `explicit` constructors {#explicit-conditional}

Just like `noexcept(bool)` we now have `explicit(bool)` to make
constructor/conversion conditionally `explicit`.

```cpp
template<typename T>
struct S{
    explicit(!std::is_convertible_v<T, int>) S(T){}
};

void f(){
    S<char> sc = 'x';           // OK
    S<std::string> ss1 = "x";   // Error, constructor is explicit
    S<std::string> ss2{"x"};    // OK
}
```

---

## Feature-test macros {#feature-test-macros}

C++20 defines a set of preprocessor macros for testing various language and library
features, the full list is [here](https://en.cppreference.com/w/cpp/feature_test).

```cpp
#ifdef __has_cpp_attribute  // check __has_cpp_attribute itself before using it
#   if __has_cpp_attribute(no_unique_address) >= 201803L
#       define CXX20_NO_UNIQUE_ADDR [[no_unique_address]]
#   endif
#endif

#ifndef CXX20_NO_UNIQUE_ADDR
#   define CXX20_NO_UNIQUE_ADDR
#endif

template<typename T>
class Widget{
    int x;
    CXX20_NO_UNIQUE_ADDR T obj;
};
```

---

## Known-to-unknown bound array conversions {#array-conv}

Allows conversion from array of known bound to the reference to array of unknown
bound. Overload resolution rules have also been updated so that overload with
matching size is better than overload with unknown or non-matching size.

```cpp
void f(int (&&)[]){};
void f(int (&)[1]){};

void g() {
  int arr[1];

  f(arr);       // calls `f(int (&)[1])`
  f({1, 2});    // calls `f(int (&&)[])`
  int(&r)[] = arr;
}
```

---

## Implicit move for more local objects and rvalue references {#more-impl-moves}

In certain cases the compiler is allowed to replace copy with move. But it turned 
out that rules were too restrictive. C++17 didn't allow to move rvalue references 
in `return` statements, function parameters in `throw` expressions, and various 
forms of conversions unreasonably prevented moving. C++20 fixed these issues but some problems
are still here, see
[P2266R0 Simpler implicit move](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p2266r0.html).

```cpp
std::unique_ptr<T> f0(std::unique_ptr<T> && ptr) {
    return ptr; // copied in C++17(thus, error), moved in C++20, OK
}

std::string f1(std::string && x) {
    return x;   // copied in C++17, moved in C++20
}

struct Widget{};

void f2(Widget w){
    throw w;    // copied in C++17, moved in C++20
}

struct From {
    From(Widget const &);
    From(Widget&&);
};

struct To {
    operator Widget() const &;
    operator Widget() &&;
};

From f3() {
    Widget w;
    return w;  // moved (no NRVO because of different types)
}

Widget f4() {
    To t;
    return t;// copied in C++17(conversions were not considered), moved in C++20
}

struct A{
    A(const Widget&);
    A(Widget&&);
};

struct B{
    B(Widget);
};

A f5() {
    Widget w;
    return w;  // moved
}

B f6() {
    Widget w;
    return w; // copied in C++17(because there's no B(Widget&&)), moved in C++20
}

struct Derived : Widget{};

std::shared_ptr<Widget> f7() {
    std::shared_ptr<Derived> result;
    return result;  // moved
}

Widget f8() {
    Derived result;
    // copied in C++17(because there's no Base(Derived)), moved in C++20
    return result;
}
```

---

## Conversion from `T*` to `bool` is narrowing {#narrowing-ptr-bool-conv}

Conversions from pointer or pointer-to-member types to `bool` are narrowing now 
and can't be used in places where such conversions are not allowed. `nullptr` is
OK when used with direct initialization.

```cpp
struct S{
    int i;
    bool b;
};

void f(){
    void* p;
    S s{1, p};          // error
    bool b1{p};         // error
    bool b2 = p;        // OK
    bool b3{nullptr};   // OK
    bool b4 = nullptr;  // error
    bool b5 = {nullptr};// error
    if(p){/*...*/}      // OK
}
```

---

## Deprecate some uses of `volatile` {#depr-volatile}

Deprecates `volatile` in various contexts:
- built-in prefix/postfix increment/decrement operators on volatile-qualified variables
- usage of the result of an assignment to volatile-qualified object
- built-in compound assignments in form of `E1 op= E2`(e.g. `a += b`) when E1 is volatile-qualified
- volatile-qualified return/parameter type
- volatile-qualified structured binding declarations

Note that `volatile-qualified` means top-level qualification, not just any
`volatile` in a type. Something like `volatile int* px` is actually
pointer-to-volatile-int, thus, not volatile-qualified.

```cpp
volatile int x{};
x++;            // deprecated
int y = x = 1;  // deprecated
x = 1;          // OK
y = x;          // OK
x += 2;         // deprecated

volatile int            //deprecated
    f(volatile int);    //deprecated
```

---

## Deprecate comma operator in subscripts {#depr-comma-subs}

Comma operator inside subscripts is deprecated to allow a multidimensional
(variadic) subscript operator [in the future](http://wg21.link/P2128R3). Current 
approach for this is to have
a custom `path_type` with overloaded `path_type::operator,()` and `operator[](path_type)`.
Variadic `operator[]` will eliminate the need for such dirty tricks.

```cpp
// current approach
struct SPath{
    SPath(int);
    SPath operator,(const SPath&);  // store path somehow
};

struct S1{
    int operator[](SPath); // use path
};

S1 s1;
auto x1 = s1[1,2,3];    // deprecated
auto x2 = s1[(1,2,3)];  // OK

// future approach
struct S2{
    int operator[](int, int, int);
    // or, as a variadic template
    template<typename... IndexType>
    int operator[](IndexType...);
};

S2 s2;
auto x3 = s2[1,2,3];
```

---

## Fixes {#fixes}

Here I put minor fixes. Some of them have been implemented by compilers for a
while but were not reflected in the Standard. Perhaps, you won't notice any
major changes in practice.

### Initializer list constructors in class template argument deduction {#fix-init-list-ctad}

```cpp
// C++17
std::tuple t{std::tuple{1, 2}};     // std::tuple<int, int>
std::vector v{std::vector{1,2,3}};  // std::vector<std::vector<int>>
```

In this example, two syntactically similar initializations result
in surprisingly different CTAD-deduced types. That's because `std::vector` has
and prefers `std::initializer_list` constructor, `std::tuple` doesn't have one
so it prefers copy constructor.  
With this fix, copy constructor is preferred to list constructor when
initializing from a single element whose type is a specialization or a 
child of specialization of the class template under construction.

```cpp
// C++20
std::tuple t{std::tuple{1, 2}};     // std::tuple<int, int>
std::vector v{std::vector{1,2,3}};  // std::vector<int>

// this example is from "C++17" book by N. Josuttis, section 9.1.1
// now it has consistent behavior across compilers
template<typename... Args>
auto make_vector(const Args&... elems)
{
    return std::vector{elems...};
}

auto v2 = make_vector(std::vector{1,2,3});  // std::vector<int>
```

---

### `const&`-qualified pointers to members {#fix-const-qual}

The problem was that using `.*` with rvalue with reference qualified pointer to 
member function was not allowed. Now it's fine.

```cpp
struct S {
    void f() const& {}
};

S{}.f();        // OK
(S{}.*&S::f)(); // could be an error on some old compilers
```

---

### Simplifying implicit lambda capture {#fix-impl-capture}

This simplifies wording for lambda capture. Lambdas within default member 
initializers now officially can have capture list, their enclosing scope is the class scope:

```cpp
struct S{
    int x{1};
    int y{[&]{ return x + 1; }()};  // OK, captures 'this'
};
```

Entities are implicitly captured even within discarded statements and `typeid`:

```cpp
template<bool B>
void f1() {
    std::unique_ptr<int> p;
    [=]() {
        if constexpr (B) {
            (void)p;        // always captures p
        }
    }();
}
f1<false>();    // error, can't capture unique_ptr by-value

void f2() {
    std::unique_ptr<int> p;
    [=]() {
        typeid(p);  // error, can't capture unique_ptr by-value
    }();
}

void f3() {
    std::unique_ptr<int> p;
    [=]() {
        sizeof(p);  // OK, unevaluated operand
    }();
}
```

---

### `const` mismatch with defaulted copy constructor {#fix-const-mismatch}

This fix allows type to have defaulted copy constructor that takes
its argument by `const` reference even if some of its members or base classes has
copy constructor that takes its argument by non-`const` reference until that
constructor is actually needed:

```cpp
struct NonConstCopyable{
    NonConstCopyable() = default;
    NonConstCopyable(NonConstCopyable&){}   // takes by non-const reference
    NonConstCopyable(NonConstCopyable&&){}
};

// std::tuple(const std::tuple& other) = default;   // takes by const reference

void f(){
    std::tuple<NonConstCopyable> t; // error in C++17, OK in C++20
    auto t2 = t;                    // always an error
    auto t3 = std::move(t);         // OK, move-ctor is used
}
```

---

### Access checking on specializations {#fix-spec-access-check}

Allows usage of `protected/private` type to be used as template arguments for 
partial specialization, explicit specialization and explicit instantiation.

```cpp
template<typename T>
void f(){}

template<typename T>
struct Trait{};

class C{
    class Impl; // private
};

template<>
struct Trait<C::Impl>{};    // OK

template struct Trait<C::Impl>; // OK

class C2{
    template<typename T>
    struct Impl;    // private
};

template<typename T>
struct Trait<C2::Impl<T>>;   // OK
```

---

### ADL and function templates that are not visible {#fix-adl}

Unqualified-id that is followed by a `<` and for
which name lookup finds nothing or finds a function is treated as a
template-name in order to potentially cause argument dependent lookup to be 
performed.

```cpp
int h;
void g();

namespace N {
	struct A {};
	template<class T> int f(T);
	template<class T> int g(T);
	template<class T> int h(T);
}

// OK: lookup of `f` finds nothing, `f` treated as a template name
auto a = f<N::A>(N::A{});
// OK: lookup of `g` finds a function, `g` treated as a template name
auto b = g<N::A>(N::A{});
// error: `h` is a variable, not a template function
auto c = h<N::A>(N::A{};
// OK, `N::h` is qualified-id
auto d = N::h<N::A>(N::A{});
```

In rare cases, this can break existing code if there's `operator<()` for
functions but it was considered as a pathological case by committee:

```cpp
struct A {};
bool operator <(void (*fp)(), A);
void f(){}
int main() {
    A a;
    f < a;      // OK until C++20, now error
    (f) < a;    // OK
}
```

---

### Specify when `constexpr` function definitions are needed for constant evaluation {#fix-constexpr-inst}

This fix specifies when `constexpr` functions are instantiated.
These rules are
pretty tricky but most of the time everything works as expected. Instead of 
copy-pasting them here I will only show a couple of examples to demonstrate the 
problem.

```cpp
struct duration {
    constexpr duration() {}
    constexpr operator int() const { return 0; }
};

// duration d = duration(); // #1
int n = sizeof(short{duration(duration())});    // always OK since C++20
```

Remember that special member functions are defined only when they are *used*.
In C++17 terms move constructor is not used and not 
defined here so the program should be ill-formed. But, if line `#1` would be uncommented,
move constructor would become *used* and defined so the program would be OK. It makes no
sense and rules have been changed to reflect this.

Another example:

```cpp
template<typename T> constexpr int f() { return T::value; }

template<bool B, typename T> void g(decltype(B ? f<T>() : 0));
template<bool B, typename T> void g(...);

template<bool B, typename T> void h(decltype(int{B ? f<T>() : 0}));
template<bool B, typename T> void h(...);

void x() {
    g<false, int>(0); // OK
    h<false, int>(0); // error
}
```

Here we have `constexpr` template function that will potentially be instantiated
with type `int` and should lead to an error because `int::value` is wrong. Then
there are two functions that use `B ? f<int>() : 0` where `B` is always `false`
so `f<int>()` is never needed.
The question is: should `f<int>` be instantiated here?  
New rules clarify
what's *needed for constant evaluation*, template variables or functions in such
expressions are always instantiated
even if they are not required to evaluate an expression. One of such cases is braced
initializer list, thus, in expression `int{B ? f<T>() : 0}` `f<T>` is always
instantiated which leads to an error.

---

### Implicit creation of objects for low-level object manipulation {#fix-impl-creation}

In C++17 an object can be created by a definition, by a new-expression or by changing
the active member of a `union`. Now, consider this example:

```cpp
struct X { int a, b; };
X *make_x() {
    X* p = (X*)malloc(sizeof(struct X));
    p->a = 1;   // UB in C++17, OK in C++20
    return p;
}
```

Although it looks natural, in C++17 this code has undefined behavior because `X` 
is not created according
to the language rules and write to a member of a nonexistent entity is UB.
Rules for such cases have been clarified by specifying what types can be created 
implicitly and what operations can create such objects implicitly.
Types that can be created implicitly(implicit-lifetime types):
- scalar types
- aggregate types
- class types with any eligible trivial constructor and trivial destructor

Operations that can create implicit-lifetime objects implicitly:
- operations that begin the lifetime of an array of `char`, `unsigned char`, 
`std::byte`
- `operator new` and `operator new[]`
- `std::allocator<T>::allocate(std::size_t n)`
- C library allocation functions: `aligned_alloc`, `calloc`, `malloc`, and 
`realloc`
- `memcpy` and `memmove`
- `std::bit_cast`

Also, the rule for pseudo-destructor(destructor for built-in types) has been 
changed. Until C++20 it has no effect, now it ends object's lifetime:

```cpp
int f(){
    using T = int;
    T n{1};
    n.~T();     // no effect in C++17, ends n's lifetime in C++20
    return n;   // OK in C++17, UB in C++20, n is dead now
}
```

You can find more detailed explanation in this post: [Objects, their lifetimes and pointers](https://blog.panicsoftware.com/objects-their-lifetimes-and-pointers/)
by Dawid Pilarski.

---

## References

[C++20 feature list](https://en.cppreference.com/w/cpp/20)  
[Complete and grouped list of all papers for each feature](https://gcc.gnu.org/projects/cxx-status.html#cxx20)  
[C++ Weekly](https://www.youtube.com/channel/UCxHAlbZQNFU2LgEtiqd2Maw)  
[CppCon 2019: Jonathan Müller "Using C++20's Three-way Comparison ＜=＞"](https://www.youtube.com/watch?v=8jNXy3K2Wpk&t)  
[CppCon 2019: Timur Doumler “C++20: The small things”](https://www.youtube.com/watch?v=Xb6u8BrfHjw)  
[C++ standard draft](http://eel.is/c++draft/)  