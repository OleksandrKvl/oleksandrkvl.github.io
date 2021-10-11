---
layout: post
title:  "From range projections to projected ranges"
date:   2021-10-11 18:22:00 +0200
post_url: 2021-10-11-projected-ranges
---

## Table of contents

* [Introduction](#intro)
* [What a projection is](#what-proj-is)
* [Problems with existing design](#problems)
* [Projected ranges to the rescue](#projected-ranges)
* [Implementation story](#implementation)
  * [C++20 iterators overview](#cpp20-iterators)
  * [Need for a better design](#need-for-better-design)
  * [The next iteration of iterators](#next-iter-iter)
  * [views::projection](#views-projection)
  * [views::narrow_projection](#views-narrow-projection)
  * [Impact on algorithms](#impact-on-algos)
  * [Reducing number of dereferences](#reducing-derefs)
  * [root() method](#root-method)
* [Other use-cases](#other-use-cases)
* [The role of std::views::transform](#transform)
* [Demo](#demo)
* [Wrap-up](#wrap-up)

---

## Introduction {#intro}

When I first watched range-related talks, I liked the idea of projections. I
played with them a bit and still liked them. However, after trying to write
range-based algorithms I found them not good enough and not pleasant to work with.
In this post I'll explain why I don't like range projections in their current
form and how I propose to fix them(demo implementation is provided).

---

## What a projection is {#what-proj-is}

If you are not familiar with projections, here's a brief explanation. 
Projection is an *invocable* entity which is applied to a range element before the
algorithm's logic will use it. It can be a lambda, pointer-to-member(either data
or function) or just a function pointer.
Along this article I will use these two structures for examples:

```cpp
struct Y
{
    int a;
    int b;
    auto operator<=>(const Y&) const = default;
};

struct X
{
    int x;
    Y y;
    auto operator<=>(const X&) const = default;
};
```

If you don't know what `operator<=>` is, don't worry, in the context of this
article you only need to know that it provides all the comparison operations(
`==, !=, <, <=, >, >=`) for both `X` and `Y`, they operate in a member-wise 
fashion. Ok, back to the subject, imagine that we 
want to sort
a vector of `X` based on `X::x`. Here's how this can be done
with the pre-ranges STL:

```cpp
std::vector<X> v;

std::sort(std::begin(v), std::end(v), [](const auto& lhs, const auto& rhs){
    return lhs.x < rhs.x;
});
```

And here's how it can be done using projection and range-based algorithm:

```cpp
std::ranges::sort(v, std::less{}, &X::x);
```

Now, `std::less` operates on `X::x` values but it's important to understand that
the algorithm itself sorts original `X` elements, not just their `X::x` parts. 
Roughly:

```cpp
auto sort(auto range, auto compare, auto projection){
    // `it1` and `it2` are iterators from `range`
    // comparator is invoked on projected values
    if(compare(std::invoke(projection, *it1), std::invoke(projection, *it2))){
        // but moving/swapping is done on non-projected values
        std::ranges::iter_swap(it1, it2);
    }
    // ...
}
```

Projection provides clear separation of comparison logic from the element
manipulation. These things are really orthogonal, it's nice that now we can 
keep them separate. And while the idea behind projection is great, its 
implementation has unpleasant side-effects which sometimes make developer
lives harder.

---

## Problems with existing design {#problems}

Several months ago I became involved in the
[P1708 Simple Statistical Functions](https://wg21.link/p1708r5) proposal. I needed
those functions for my hobby project and started implementing them. This was my
first experience in writing range-based API and that's how I got most of my
unpleasant experience working with current projections design.

### Projections uglify function signatures

In range-based API you usually have at least one range for which you need to
support and hence provide a projection. For example, simplified signature of 
`copy_if` with removed return type and `O` type requirements:

```cpp
template<ranges::input_range R, typename O,
    class Proj = std::identity,
    std::indirect_unary_predicate<
        std::projected<ranges::iterator_t<R>, Proj>> Pred>
constexpr auto copy_if(R&& r, O result, Pred pred, Proj proj = {});
```

All range-based algorithms must have this additional function and template
parameter that defaults to a
no-op `std::identity` projection. Looks innocent? In P1708 we have
weighted statistics so we use two ranges: one for values and one for weights,
thus, we need one projection per range:

```cpp
template<
    typename Values,
    typename ValuesProj = std::identity,
    typename Weights,
    typename WeightsProj = std::identity>
constexpr auto mean(
    Values&& v, Weights&& w, ValuesProj proj1 = {}, WeightsProj proj2 = {});
```

Add to it more algorithm specific parameters like comparators and you'll get
something like `std::ranges::merge()`:

```cpp
template<
    ranges::input_range R1,
    ranges::input_range R2,
    std::weakly_incrementable O,
    class Comp = ranges::less,
    class Proj1 = std::identity,
    class Proj2 = std::identity>
constexpr auto merge(R1&& r1, R2&& r2, O result,
        Comp comp = {}, Proj1 proj1 = {}, Proj2 proj2 = {});
```

I believe that in a good API default function arguments should be rare and
their number should be small. Here, we have 6 parameters and 3 of them have
default arguments. This signature is not good at all, we also will discuss
usability of such API in following sections.

Another issue, though not so critical, is access to projected value type in 
order, for example, to constrain it. Recall the constraint from the `copy_if()`:
`std::indirect_unary_predicate<std::projected<ranges::iterator_t<R>, Proj>> Pred`.
It ensures that predicate `Pred` can be called with the result of applying
projection `Proj` to the value of the iterator of a range `R`. It's understandable
but still quite complex. In P1708R5 functions are supposed to work only on
standard arithmetic types, a way to achieve it:

```cpp
template<typename Range, typename Proj = std::identity>
requires std::is_arithmetic_v<
    std::remove_cvref_t<
        std::indirect_result_t<Proj&, std::ranges::iter_value_t<Range>>>>
double mean(Range&& r, Proj = {});
```

I mean, OK, it works and with some effort you can do it properly. But I don't
like its complexity. Writing your own algorithms in the classic STL style was
simple, writing them for ranges is not if you want to support projections
properly.

---

### Projections are not easily composable

Imagine that you're implementing a range-based algorithm and you need to call
another algorithm but with one more additional projection. For example, [geometric
mean](https://en.wikipedia.org/wiki/Geometric_mean#Relationship_with_logarithms)
is usually implemented in terms of arithmetic mean of logarithms and final
`std::exp()` of it. This requires combination of two projections, original one
and `std::log()`:

```cpp
template<typename R, typename P = std::identity>
constexpr double geometric_mean(R&& r, P proj = {})
{
    const auto logs_mean = mean(
        std::forward<R>(r),
        [&](const auto& value)
        {
            return std::log(std::invoke(proj, value));
        });

    return std::exp(logs_mean);
}
```

It would be nice to move `std::log()` part to a separate independent projection
but such a projection wouldn't be really independent because it needs to know
about the preceding one:

```cpp
// to make it a reusable function-like object we again need this additional
// parameter everywhere
template<typename P = std::identity>
class log_proj{
public:
    explicit log_proj(P proj = {}): p{std::move(proj)}{}

    auto operator()(const auto& value){
        return std::log(std::invoke(p, value));
    }
private:
    P p;
};
// with the above it's possible to write:
// const auto logs_mean = mean(r, log_proj{proj});

// and this is what I want as a client:
struct nice_log_proj{
    auto operator()(const auto& value){
        return std::log(value);
    }
};
```

Of course, it's possible to create another utility to chain projections together
like `mean(r, chain(std::move(proj), nice_log_proj{}))` but at the moment
there's no standard tool for that.
This problem also occurs when you want to sort `std::vector<X>` by `Y::a` member of
`X::y`. In C++ it's not possible to get a pointer to a member of a member, `&X::y::a`
doesn't work, something like `chain(&X::y, &Y::a)` is needed.

---

### Projections complicate caller's code

Imagine a function with several default arguments:

```cpp
template<typename R, typename P = std::identity>
void f(R&& range, int x = 1, int y = 2, P p = {});
```

Because the projection is usually placed at the end of signature, if you need to
use it, you have to specify all the default arguments by hand:

```cpp
f(v);               // without projection
f(v, 1, 2, &X::x);  // with projection
```

What if the author of `f()` decides to change default arguments? Clients will be
forced to rewrite the code to preserve "default" behavior. It's less painful
with something like `std::ranges::sort()`:

```cpp
std::ranges::sort(v);   // without projection
std::ranges::sort(v, std::less{}, &X::x);   // with projection
std::ranges::sort(v, {}, &X::x);    // less verbose but less readable too
```

But now it's either too verbose with explicit `std::less{}` or less readable
with `{}`. So the client is either forced to
explicitly write arguments by hand or use less readable constructions if that's
possible at all. Going back to weighted stats:

```cpp
template<
    typename Values,
    typename Weights,
    typename ValuesProj = std::identity,
    typename WeightsProj = std::identity>
constexpr auto mean(
    Values&& v, Weights&& w, ValuesProj proj1 = {}, WeightsProj proj2 = {});

mean(values, weights);    // no projections, great
mean(values, weights, &X::x); // only value projection, OK
mean(values, weights, {}, &X::x); // only weight projection, ugly :(
```

---

### Root cause of all the problems

From the interface point of view it's pretty simple, the problem is that range
and projection represent a logically single entity but are passed to functions 
separately via distinct
parameters. It's the same as for
error-prone `f(const char* str, std::size_t len);` and we all know it's a bad
way of doing things. Clients are forced to separate things, developers are 
forced to combine them back together, I want something better.

---

## Projected ranges to the rescue {#projected-ranges}

I had quite a simple idea: range and projection should be combined into a single
thing using some kind of view, e.g., `views::projection`. This would make all
those projection-related parameters redundant, algorithms wouldn't care about
them at all, they would only operate on a range itself, just like in classic
STL. Here's what I wanted:

```cpp
// no projection-related parameters
auto sort(auto&& range, auto cmp = std::ranges::less{});

// sorts elements by `X::x` member, analog of current sort(v, {}, &X::x);
sort(v | projection(&X::x));

const auto log_proj = [](const auto value)
{
    return std::log(value);
};

// nested projections, actually, it's a nested range now
constexpr double geometric_mean(auto&& r)
{
    const auto logs_mean = mean(r | projection(log_proj{}));
    return std::exp(logs_mean);
}
```

Isn't it great? No more projection-related parameters, signatures are clean,
everything is perfectly composable. It
simplifies projections in the same way as ranges simplified usage and
composition of iterator-based algorithms.

I call it a *projected range* because it combines range and projection.
Such a range
has very important property: its `operator*()` returns projected
value, while copy/move/swap/assign operations should be performed on the whole
underlying object. Immediately, another type of projection
came to my mind, the so-called `narrow_projection`. All of its operations are
performed on the projected part only. It's *narrow* in a sense that it
represents only a narrow part of the object while *wide* `projection`
represents a wider object behind it:

{% raw %}
```cpp
std::vector<X> v{{3, {30, 300}}, {2, {20, 200}}, {1, {10, 100}}};

// sorts the whole X objects using &X::x member
std::sort(v | projection(&X::x));
// {{1, {10, 100}}, {2, {20, 200}}, {3, {30, 300}}}

// sorts only X::x
std::sort(v | narrow_projection(&X::x), std::ranges::greater{});
// {{3, {10, 100}}, {2, {20, 200}}, {1, {30, 300}}}

// sorts X::y by Y::a
std::sort(
    v | narrow_projection(&X::y) | projection(&Y::a), std::ranges::greater{});
// {{3, {30, 300}}, {2, {20, 200}}, {1, {10, 100}}}
```
{% endraw %}

Delighted, I started to think how to implement it and it turned out to be a
bit harder than I expected.

---

## Implementation story {#implementation}

If you don't know how range views work, here's the basic idea: all work is
done inside custom "smart" iterators. For example, iterator for the most relevant
to projections `views::transform` has `operator*()` which looks like this:

```cpp
class transform_view_iterator{
private:
    Iterator it;    // underlying iterator
    F f;            // transform function

public:
    decltype(auto) operator*(){
        return std::invoke(f, *it);
    }
    // ...
};
```

Other operations mostly take care of proper `it` moving. To implement
`views::projection` we will need to implement a custom iterator, thus, we need
first to understand how  iterators work in C++20.

---

### C++20 iterators overview {#cpp20-iterators}

Here's the brief overview of iterator-related types and operations. It's
heavily based on articles/papers by Eric Niebler
([0](http://ericniebler.com/2015/01/28/to-be-or-not-to-be-an-iterator/),
[1](https://ericniebler.com/2015/02/03/iterators-plus-plus-part-1/),
[2](https://ericniebler.com/2015/02/13/iterators-plus-plus-part-2/),
[3](https://ericniebler.com/2015/03/03/iterators-plus-plus-part-3/),
[4](https://ericniebler.github.io/std/wg21/D0022.html)).
Read them if you want more details and reasoning behind current design.

`iter_value_t/value_type` - the type of a value which the iterator represents. The
value of this type can be copied/moved from the iterator.

`iter_reference_t operator*()` - dereference operator, usually returns lvalue
reference to `value_type` (but not required), must be convertible to `iter_value_t`.

`iter_rvalue_reference_t iter_move(it)` - customization
point for moving value out of iterator, usually returns rvalue reference to
`value_type`(but not required). Also must be convertible to `iter_value_t`. If
not defined by iterator, `std::move(*it)` is used.

`void iter_swap(it1, it2)` - customization point for swapping values between
two iterators. If not defined, performs `std::ranges::swap(*it1, *it2)` if 
possible, otherwise uses `iter_move()` to swap elements "by-hand".

`common_reference` requirements for readable iterators. Now comes the tricky
part. As you might have
noticed, `iter_value_t`, `iter_reference_t`, `iter_rvalue_reference_t` are not
required to be as simple as `int`, `int&` and `int&&` correspondingly. But there
must be pairwise `common_reference`s to represent relationships between them.
Basically, `common_reference<T,U>` is a type to which both `T` and `U` can be
converted or bound, it's not required to be a true reference type.

```cpp
static_assert(std::same_as<std::common_reference_t<int&, const int&>, const int&>);
static_assert(std::same_as<std::common_reference_t<int&&, int&&>, int&&>);
static_assert(std::same_as<std::common_reference_t<int&&, int&>, const int&>);
static_assert(std::same_as<std::common_reference_t<int&, int>, int>);
```

You can find
these requirements in the [`std::indirectly_readable`](https://en.cppreference.com/w/cpp/iterator/indirectly_readable) concept:

```cpp
template<class In>
concept __IndirectlyReadableImpl = // exposition only
requires(const In in) {
    typename std::iter_value_t<In>;
    typename std::iter_reference_t<In>;
    typename std::iter_rvalue_reference_t<In>;
    { *in } -> std::same_as<std::iter_reference_t<In>>;
    { ranges::iter_move(in) } -> std::same_as<std::iter_rvalue_reference_t<In>>;
} &&
std::common_reference_with<
    std::iter_reference_t<In>&&, std::iter_value_t<In>&> &&
std::common_reference_with<
    std::iter_reference_t<In>&&, std::iter_rvalue_reference_t<In>&&> &&
std::common_reference_with<
    std::iter_rvalue_reference_t<In>&&, const std::iter_value_t<In>&>;
```

Interestingly, there's no requirement that all these `common_reference`s must
be the same type. In fact, they are not even required to be used and hence
defined but they must be declared. Eric shows one example when `common_reference`
might be useful, `unique_copy()` comparator parameter types. `unique_copy()`
needs to copy `value_type` and then call comparator with this copy and the
result of `operator*()` which is `iter_reference_t`. But the order of arguments
is not specified. If for whatever reason your comparator cannot have templated
parameters, you need to use `common_reference` for parameter types:

```cpp
auto unique_copy(Iterator first, Iterator last, auto d_first, auto comparator){
    // somewhere inside `unique_copy()`
    Iterator it = first;
    std::iter_value_t<It> copy = *it;   // copy current element
    ++it;
    comparator(copy, *it);  // compare it to the next one, it can be one way
    comparator(*it, copy);  // or the other
}

// client's code
auto generic_comparator = [](auto& lhs, auto& rhs){};   // no problems

// but if you need specific types, use common_reference
template<std::indirectly_readable T>
using iter_common_reference_t = std::common_reference_t<
    std::iter_reference_t<T>,std::iter_value_t<T>&>;

auto non_generic_comparator = [](
    iter_common_reference_t<T> lhs, iter_common_reference_t<T> rhs){};
```

Note that since C++20, `iter_reference_t` is not required to be a true reference
for any kind of iterator which effectively allows random-access proxy iterators.

Range-based versions of existing algorithms must be changed like this(current
`libstdc++` still doesn't use `iter_move()`/`iter_swap()` in its range algorithms):

```cpp
Iterator it1, it2;

// pre C++20 algorithms:
auto copied = *it1;             // copy
auto moved = std::move(*it1);   // move
std::iter_swap(it1, it2);       // swap

// C++20 algorithms:
using value_type = std::iter_value_t<Iterator>;
value_type copied = *it1;                       // copy
value_type moved = std::ranges::iter_move(it);  // move
std::ranges::iter_swap(it1, it2);               // swap
```

The main purpose of this design(as I understand it) is to allow proxy iterators
of any kind, which, in theory, allows more "indirect" iterators and their usage
with standard algorithms. Imaginary proxy-iterator must implement:
- corresponding to its category functions(`operator++()`, `operator[]`, etc.)
- custom proxy-reference type which must have read/write/conversions to/from 
`value_type`, itself and `iter_rvalue_reference_t`
- custom `iter_move()` and `iter_swap()`
- specialize necessary `basic_common_reference`(a helper for `common_reference`
described above) between its `value_type`, 
`iter_reference_t`, `iter_rvalue_reference_t` to a type which is at least
declared

---

### Need for a better design {#need-for-better-design}

Now, when you have a basic idea of what iterators can do in C++20, we can start to
think about how `views::projection` should work. Recall usage example:

```cpp
sort(v | projection(&X::x));    // sorts `v` by `X::x` member
```

For this to work, `operator*()` must return a reference-like thing which points to
the projected value(`X::x` member in this
case) so that the comparator will use it instead of the whole object. On the other
hand, copy/move/swap/assign operations must operate on the non-projected object(`X`).
In other words, we have two distinct types:
- `value_type` - the type exposed through `operator*()`, projected type
- `iter_root_t` - the type of underlying object, root type

and there's no logical relationship between them, i.e., there's no connection
between `int x;` and `struct X;` types. One can argue that in fact we have `X`
and `&X::x` types and there is a `member-of` relation but in reality projection
can also be a pure transformation, e.g.,
from `std::string` to `int` so any kind of relationship doesn't make sense here.  
In contrast, the existing design doesn't leave space for a second type(`iter_root_t`).
It allows proxy-reference as `iter_reference_t` but it enforces strict
relationships between it and `value_type` in terms of `common_reference`
requirements. At most, it allows representing logically single `value_type`
with two different types, like an advanced form of pointer. That's why related
concepts are named like `indirectly_readable/writable/etc`, it's all about
indirection mechanics, not true abstraction from one type to another.  
And even this indirection mechanism is over-complicated, I'd say it's
expert- ~~friendly~~ only utility. I mean, when Eric
Niebler [says it's hard](https://twitter.com/ericniebler/status/1363892564831711235)(
you can check his implementation [here](https://github.com/ericniebler/range-v3/blob/master/include/range/v3/iterator/basic_iterator.hpp)), how can you expect people
to write their own iterators using it? It's hard because if you need to use
proxy-reference, you need first to check and understand algorithm requirements
on operations/conversions proxy-reference(`iter_reference_t`), `value_type` and
`iter_rvalue_reference_t` should support and only then *try* to implement it.

To summarize, there are two main problems: over-complicated design and it's
inability to support true abstraction between two unrelated types. Now, let's
fix it.

---

### The next iteration of iterators {#next-iter-iter}

In [Projected ranges to the rescue](#projected-ranges) section I said that
algorithms must operate on the `iter_root_t` values only, `value_type` should be 
used only when it's passed to customizable logic like comparators. Thus, we need
to separate `value_type` API from `iter_root_t` API. Let's summarize what we
have so far:
- `operator*()` to get an lvalue reference or copy of `value_type`
- `operator*()` to write `value_type`
- `iter_move(it)` to get rvalue reference to `value_type`
- `iter_swap(it1, it2)` to swap whatever we want, in our case it's `iter_root_t`

Now we need similar functions for `iter_root_t`:
- `iter_copy_root(it)` to get an lvalue reference to `iter_root_t`, 
`iter_root_reference_t`
- `iter_move_root(it)` to get an rvalue reference to `iter_root_t`, 
`iter_root_rvalue_reference_t`

And to simplify assignment:
- `iter_assign_from(it, value)` to assign whatever is needed

All these new functions are customization point objects(CPO) which means
they are not required to be implemented if the iterator is happy with the default
behavior. One of my goals was to preserve backward compatibility with all
existing iterators so default implementations mostly forward to the old
API. If you are not familiar with typical CPO implementation, the idea is quite
simple: you call customized for a specific type function or the default 
implementation. The
presence of a customized function is detected via ADL check(`has_adl_[cpo_name]`
below). Implementation is located inside `struct` that's why in the code below
`operator()(...)` is used instead of a plain function. `stdf` is a namespace
name where I put all the new stuff, not a typo.

---

#### iter_copy_root()

Returns lvalue reference to `iter_root_t`. Default behavior is to return
the result of `operator*()`. I deliberately omit return type, `noexcept`-ness
and constraint specifications since they are trivial, interested readers can
find them in demo implementation.

```cpp
template<typename From>
constexpr decltype(auto) operator()(From&& from) const
{
    if constexpr(has_adl_iter_copy_root<From>)
    {
        return iter_copy_root(static_cast<From&&>(from));
    }
    else
    {
        return *from;
    }
}

// helper aliases
template<typename T>
using iter_root_t =
    std::remove_cvref_t<decltype(stdf::iter_copy_root(std::declval<T>()))>;

template<typename T>
using iter_root_reference_t = decltype(stdf::iter_copy_root(std::declval<T>()));

// usage example:
auto& ref = stdf::iter_copy_root(it);
auto copy = stdf::iter_copy_root(it);
```

---

#### iter_move_root()

Returns rvalue reference to underlying object. When not customized, can forward
to `iter_move()` or to `iter_copy_root()`. Reason for this is simple: 
`iter_move_root()` is supposed to return rvalue reference to root type, if
`iter_copy_root()` is not customized, it operates in terms of value type and 
`iter_move()` is responsible for moving it. This also preserves backward
compatibility, for existing iterators `iter_copy_root()` is forwarded
to `operator*()` and `iter_move_root()` to `iter_move()`.

```cpp
constexpr decltype(auto) operator()(From&& from) const
{
    if constexpr(has_adl_iter_move_root<From>)
    {
        return iter_move_root(static_cast<From&&>(from));
    }
    else if constexpr(
        iter_move_cpo::has_adl_iter_move<From> &&
        !iter_copy_root_cpo::has_adl_iter_copy_root<From>)
    {
        return stdf::iter_move(static_cast<From&&>(from));
    }
    else if constexpr(std::is_lvalue_reference_v<
                            iter_root_reference_t<From>>)
    {
        return std::move(
            stdf::iter_copy_root(static_cast<From&&>(from)));
    }
    else
    {
        return stdf::iter_copy_root(static_cast<From&&>(from));
    }
}

template<typename T>
using iter_root_rvalue_reference_t =
    decltype(stdf::iter_move_root(std::declval<T>()));

// usage example:
auto moved = stdf::iter_move_root(it);
```

---

#### iter_assign_from()

It is responsible for assignment. Developer has full control over supported
types. It's possible to introduce `iter_assign_value` and `iter_assign_root` but
I don't know any use-case where it might be useful. Default behavior assigns to
root:

```cpp
template<typename To, typename From>
constexpr void operator()(To&& to, From&& from) const
{
    if constexpr(has_adl_iter_assign_from<To, From>)
    {
        iter_assign_from(static_cast<To&&>(to), static_cast<From&&>(from));
    }
    else
    {
        stdf::iter_copy_root(static_cast<To&&>(to)) = static_cast<From&&>(from);
    }
}

// helper concept
template<typename To, typename From>
concept iter_assignable_from = requires(To&& to, From&& from)
{
    stdf::iter_assign_from(static_cast<To&&>(to), static_cast<From&&>(from));
};

// usage example:
stdf::iter_assign_from(it, T{});
```

---

#### iter_swap()

`iter_swap()` behaves almost like `std::ranges::iter_swap()` but it operates on
root values, i.e., it uses `iter_copy_root()` instead of `operator*()` and
`iter_move_root()`/`iter_assign_from()` instead of `iter_move()`/`operator=()`. I
don't show implementation here because it's not so short, you can find it in the
demo.

---

### views::projection {#views-projection}

Now, when we have full control over the iterator's behavior, we can finally
implement `views::projection` and `views::narrow_projection` and see how the new
API simplifies custom iterator implementation.
I will show only core parts of the iterator. We need to store current underlying
iterator and a pointer to a parent view where projection function is stored:

```cpp
class Iterator
{
private:
    BaseIter current{};
    ParentView* parent{};
};
```

Core parts:

```cpp
constexpr decltype(auto) operator*() const
{
    return std::invoke(parent->fun, *current);
}

friend constexpr decltype(auto) iter_copy_root(const Iterator& it)
{
    return stdf::iter_copy_root(it.current);
}

// enabled only if `BaseIter` has custom `iter_move_root`
friend constexpr decltype(auto) iter_move_root(const Iterator& it)
{
    return stdf::iter_move_root(it.current);
}

// enabled only if `BaseIter` has custom `iter_swap`
friend constexpr void iter_swap(const Iterator& x, const Iterator& y)
{
    return stdf::iter_swap(x.current, y.current);
}

// enabled only if `BaseIter` has custom `iter_assign_from`
template<typename T>
friend constexpr void iter_assign_from(const Iterator& it, T&& val)
{
    stdf::iter_assign_from(it.current, std::forward<T>(val));
}
```

As you can see, it's trivial, all of them are one-liners.
`operator*()` returns the result of applying projection to the iterator's value. We
don't need custom `iter_move()` because default implementation operates on the
result of `operator*()`. Since we want copy/move/assign/swap operations to 
operate on 
the root value, we simply forward these calls to it. Note that the last three
functions are enabled(using `requires`-clause) only in case when the underlying 
iterator 
customizes them. Otherwise, their default versions will operate on
the basis of `iter_copy_root()` which is exactly what's needed. There's another
reason why it's better to avoid customized versions of CPO-s when possible, 
it's described later in section [Reducing number of dereferences](#reducing-derefs).

---

### views::narrow_projection {#views-narrow-projection}

It's even simpler, all we need is:

```cpp
constexpr decltype(auto) operator*() const
{
    return std::invoke(parent->fun, *current);
}
```

Because we don't want to expose the underlying root value to copy/move/swap/assign 
operations, everything else works by default.

---

### Impact on algorithms {#impact-on-algos}

Just like it was with C++20 iterator API, this one also requires algorithm authors
to update implementations. Their requirements have to be updated to
reflect usage of the new API. Changes to algorithms code are trivial, `operator*()`
is still used for customizable logic like comparators but copy/move/swap/assign
must be replaced with new functions. In the demo I implemented a couple of simple
algorithms to test how the new design fits in and found no major problems.

```cpp
// read projected value
auto v1 = std::invoke(proj, *it);   // before
auto v2 = *it;                      // after

// copy underlying value, now copies `iter_root_t`
iter_value_t<It> copy1 = *it;               // before
iter_root_t<It> copy2 = iter_copy_root(it); // after

// move underlying value, now moves `iter_root_t`
iter_value_t<It> moved1 = iter_move(it);    // before
iter_root_t<It> moved = iter_move_root(it); // after

// assign to iterator
*it = val;                  // before
iter_assign_from(it, val);  // after
```

#### Iterator-based versions of algorithms

For some reason, all range-based algorithms also have iterator-based
counterparts, e.g., `copy_if(Range r, Out o, Pred pred, Proj proj)` and `copy_if(I begin, S end, Out o, Pred pred, Proj proj)`.
I don't know why they are needed at all when
a pair of iterators can be converted into a range using `std::ranges::subrange`
but they are here.
Described `projection`/`narrow_projection` combine projection and a *range*. To
remove projections from iterator-based signatures we need something like
`projection_iterator`. It should work just like `projection_view::iterator`
with addition of comparison functions with its root iterator to support cases
like `std::ranges::sort(make_projection_iterator(std::begin(r), some_projection), std::end(r))`. Or this issue can be ignored at all, 
projections can be removed without introducing `projection_iterator`. It will 
force usage of `std::ranges::subrange` and projection on the resulting
range.

---

### Reducing number of dereferences {#reducing-derefs}

While implementing algorithms, I found one interesting issue.
Consider `copy_if()` algorithm. Here's `libstdc++` implementation:

```cpp
void copy_if(auto first, auto last, auto result, auto pred, auto proj)
{
    for (; first != last; ++first)
    {
        if (std::invoke(pred, std::invoke(proj, *first)))   // #1
        {
            *result = *first;   // #2
            ++result;
        }
    }
}
```

The subtle issue here is that `first` is dereferenced twice per iteration, first,
to call
the predicate, second, to copy its value to the output `result` iterator. As I told
you before, `libstdc++` still uses old implementations for range-based
algorithms and probably this version is OK for old-school iterators. But in a
`ranges` world `operator*()` might do non-trivial things. For example, it might
be a range which uses `views::transform` with `int -> string` transformation.
`range-v3` handles it better, whenever
possible it stores and reuses the result of dereference:

```cpp
void copy_if(auto first, auto last, auto result, auto pred, auto proj)
{
    for (; first != last; ++first)
    {
        auto&& x = *first;     // dereference is done only once now
        if (std::invoke(pred, std::invoke(proj, x)))
        {
            *result = (decltype(x) &&)x;    // analog of std::forward<...>(x)
            ++result;
        }
    }
}
```

With that in mind, I wrote initial implementation using the new API:

```cpp
constexpr void copy_if(auto&& in, auto out, auto pred)
{
    auto first = std::ranges::begin(in);
    auto last = std::ranges::end(in);
    for(; first != last; ++first)
    {
        if(std::invoke(pred, *first))
        {
            iter_assign_from(out, iter_copy_root(first));
            ++out;
        }
    }
}
```

Explicit dereference is done only once here but recall that when
`iter_copy_root()` is not customized by client, it falls back to `operator*()` so
the above code transforms to:

```cpp
if(std::invoke(pred, *first))       // first dereference
{
    iter_assign_from(out, *first);  // second dereference
    ++out;
}
```

Taking into account that now `operator*()` might contain projection, I want to
avoid calling it whenever possible. Also, standard algorithms guarantee a
specific number of projection calls, any approach which cannot fulfill
them would be useless.
The fixed version would be:

```cpp
constexpr void copy_if(auto&& in, auto out, auto pred)
{
    auto first = std::ranges::begin(in);
    auto last = std::ranges::end(in);
    for(; first != last; ++first)
    {
        auto&& x = *first;
        if(std::invoke(pred, x))
        {
            if constexpr(has_adl_iter_copy_root<decltype(first)>)
            {
                // call customization point
                iter_assign_from(out, iter_copy_root(first));
            }
            else
            {
                // reuse `x`
                iter_assign_from(out, std::forward<decltype(x)>(x));
            }
            ++out;
        }
    }
}
```

Now when there's no customized `iter_copy_root()`, the dereferenced value can
safely be reused.
Obviously, having such an `if` statement in all algorithms for each call of
`iter_copy_root`, `iter_move_root` and `iter_assign_from` would be too verbose.
To
simplify it, I added a second version for each CPO with additional `dereferenced`
parameter at the end. Now `copy_if()` is shorter and dereferences only once:

```cpp
// second version of iter_copy_root()
template<typename From>
constexpr decltype(auto) operator()(
    From&& from, std::iter_reference_t<From>& dereferenced) const
{
    if constexpr(has_adl_iter_copy_root<From>)
    {
        return iter_copy_root(static_cast<From&&>(from));
    }
    else if constexpr(std::is_lvalue_reference_v<std::iter_reference_t<From>>)
    {
        return dereferenced;
    }
    else
    {
        return std::move(dereferenced);
    }
}

constexpr void copy_if(auto&& in, auto out, auto pred)
{
    auto first = std::ranges::begin(in);
    auto last = std::ranges::end(in);
    for(; first != last; ++first)
    {
        auto&& x = *first;
        if(std::invoke(pred, x))
        {
            stdf::iter_assign_from(out, stdf::iter_copy_root(first, x));
            ++out;
        }
    }
}
```

As a side-effect, it reduces the number of dereferences for all currently
existing iterators because they don't customize new CPOs. Usually, an optimizer
is able to eliminate them but I like that now it's guaranteed by design with or
without optimizations. The same problem exists for `iter_move()` and
`iter_swap()` because when not customized, they dereference. At the end, I 
added a second version for them too.
That's why in `views::projection` it's important to enable custom
`iter_move_root()`, `iter_swap()`, `iter_assign_from()` only if they are
customized by the underlying iterator. Customizing them unconditionally prevents
reuse of dereferenced value.

---

### root() method {#root-method}

Sometimes we need to use the result of a generic algorithm with member function
of a container. One such example is `remove_if()`. It
returns a range of removed elements which are then `erase()`d using member
function. The signature in `std::vector` is `constexpr iterator erase(const_iterator first, const_iterator last);`.
The problem is that it takes `std::vector::const_iterator` and when we do:

```cpp
auto pv = v | projection(&X::x);
auto removed = stdf::remove_if(pv, less_than<int{3}>{});
```

`removed` contains a range of `projection_view::iterator` so we need a way to get
the underlying iterator from it. It's possible to provide an implicit conversion
for it but implicit conversions are always dangerous so for now I made it a
normal member function. While the existing `base()` method of view iterators returns 
the last wrapped iterator, the new `root()` method returns the very first iterator
in the projection chain. To make it generic, I added a `stdf::root(it)` free function
which falls back to `it.root()` or just returns `it`. Now we can do:

```cpp
auto pv = v | projection(&X::x);
auto removed = stdf::remove_if(pv, less_than<int{3}>{});
v.erase(stdf::root(removed.begin()), stdf::root(removed.end()));
```

---

## Other use-cases {#other-use-cases}

Introduced design significantly simplifies creation of non-trivial iterators.
Because each aspect is handled separately, there's no need for tricky proxy
reference objects. `common_reference` requirements are still there, now for
both `value_type` and `iter_root_t`, but it's almost impossible to break them so
clients shouldn't care or even know about their existence. For example, here's
how infamous `std::vector<bool>::iterator` can be implemented:

```cpp
class Iterator
{
public:
    bool operator*();   // no need for proxy reference type
    // swaps bits
    friend void iter_swap(const Iterator& lhs, const Iterator& rhs);
    // assigns bit from bool value
    friend void iter_assign_from(const Iterator& lhs, bool val);
};
```

Because there's no sense in true copy/move of a single bit, `iter_move()`,
`iter_copy_root()`, `iter_move_root()` work in terms of `bool` value returned by
`operator*()`. But `iter_swap()` needs to actually swap bit values and
`iter_assign_from()` should assign `bool` to a specific bit, thus, they are 
customized. It's possible to achieve it with the existing design but it
requires a custom proxy reference type and `basic_common_reference`
specializations.

Another use-case might be various wrappers. Once I wanted to
write a wrapper for [rapidjson](https://rapidjson.org/) library. The main part
was a wrapper class around
`rapidjson::Value` and `rapidjson::Document::AllocatorType`
which provided a more convenient interface similar to
[nlohmann/json](https://github.com/nlohmann/json). Writing the wrapper itself
was easy but I failed at
the point when I needed to provide a random-access iterator which returns my 
wrapper by-value. In C++17 it was 
impossible to achieve simply because `operator*()` returns value instead of
true reference and such iterator couldn't be a random-access one. In C++20 it
should be possible, but again, requires a good understanding of proxy reference
and `common_reference` requirements to implement it. With 
the proposed 
design it's straightforward: root type is `rapidjson::Value`, value type is
a wrapper:

```cpp
class MyWrapper{
public:
    // interface methods...
private:
    rapidjson::Value* value;
    rapidjson::Document::AllocatorType* allocator;  // required for write ops
};

class MyIterator{
public:
    auto operator*(){
        return MyWrapper{*origIterator, allocator};
    }

    decltype(auto) iter_copy_root(){
        return *origIterator;
    }

    // other iterator methods...

private:
    rapidjson::Value::ValueIterator origIterator;
    rapidjson::Document::AllocatorType* allocator;
};
```

---

## The role of std::views::transform {#transform}

Someone might think that `std::views::transform` can be used to combine
projection and a range but currently it's mostly useless for that purpose.
Its `iter_move()` operates on transformed value while `iter_swap` operates on
the underlying non-transformed value so you can't use it with any algorithm that
might use
them both(like `sort()`, see the
[issue 3520](https://cplusplus.github.io/LWG/issue3520)). The
proposed fix is to remove customized `iter_swap()` so that the default version
will operate on transformed value. With that fix,
`views::transform` will become almost the same as
`narrow_projection`(the only difference is that, for unknown reason,
`views::transform` has [customized](https://eel.is/c++draft/range.transform.iterator) 
`iter_move()` which behaves exactly like [the
default one](https://eel.is/c++draft/iterator.cust.move#1.2)). But should it
be used instead of `narrow_projection`? It's more like a naming question, I
think that the name `transform` corresponds to a case when that's the real
purpose of the code, just like classic `std::transform()`. The name `projection`
better fits cases when you don't want to transform a range but only change its
representation for an algorithm. Of course such a thing can still be called a
*transformation*, it's hard to get a clear answer here.

---

## Demo {#demo}

You can find the implementation of new CPO-s, `projection`, `narrow_projection`
and a few test algorithms [here](https://github.com/OleksandrKvl/projected_ranges/blob/master/src/main.cpp).
It's just a single file which you can [copy-paste to godbolt](https://godbolt.org/z/685Pcavb3),
currently it works only with GCC-11 because Clang hasn't implemented ranges yet.

---

## Wrap-up {#wrap-up}

The main benefit of introduced design is the support of true abstraction behind
the iterator. Projection is only one of its use cases and
I believe that it can fully replace and enhance them, at the same time simplify 
creation of new iterators. Important point
is that it's backward compatible, no need to change existing iterators, only
the algorithms.
Let me know what you think about it. Do you like this design? Would you like to
see it in the standard? Can it solve some of your problems or will create new
ones instead? Have I missed something else? Any meaningful feedback is welcome.