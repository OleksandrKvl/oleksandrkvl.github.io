---
layout: post
title:  "Allow only pure data structs with clang-tidy"
date:   2020-11-04 18:33:00 +0200
post_url: 2020-11-04-pure-data-structs-clang-tidy
---
## Introduction

`struct` is for *data*, `class` is for *invariant*. This is what guides told us([Core guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-class), 
[Google C++ style guide](https://google.github.io/styleguide/cppguide.html#Structs_vs._Classes), 
[Fluent C++ blog](https://www.fluentcpp.com/2017/06/13/the-real-difference-between-struct-class/)).
Sounds like a good candidate for another simple `clang-tidy` check. While 
implementation of such a check is more or less simple, defining the actual 
constraints
for it is not. In this article I'll discuss what could *pure data* `struct` mean 
and flavors it could have. Then I'll briefly show my implementation of a 
corresponding `clang-tidy` check.

## Pure data struct

Naively, `pure data struct` means that there are only data members and nothing
more, just like C `struct`:

```cpp
struct Point{
    int x;
    int y;
};
```

But in C++ world we can have much more in it and still consider that as a *data*.
It turned out that it can have several levels of strictness, that's why I decided
to make this tool configurable instead of hardcoding one *the only* way.

#### Static members

Because `static` members are not part of an object's state, I decided to completely
ignore them. Also, I don't care whether they are `public` or not.

#### Stateless structs

`struct`s are often used for various TMP tricks and for simple callables:

```cpp
struct less<Widget>{
    bool operator()(const Widget&, const Widget&){
        //...
    }
};
```

In such context they are simply a shorthand for a stateless `class` with all 
members being
`public`. However, in some codebases, especially old ones, you can find this 
trick(which I also consider as a non-data `struct`):

```cpp
struct WidgetList : std::vector<Widget>{};
```

Thus, instead of always skipping stateless structs, I added an option 
`SkipStateless` to control it. By default it's `true`.

#### Data members

Obviously, only `public` data members are allowed. What about default member
initialization, do you think it violates *pure data* notion? Consider this simple 
example:

```cpp
struct Point{
    int x{};
    int y{};
};
```

It doesn't have invariants but it has a kind of a contract, specifically 
postcondition:

```cpp
Point p;
assert((p.x == 0) && (p.y == 0));
```

It binds magic values(`0`s in this particular case) to its members. As opposed to
`class`, `struct` should model a *collection* of values without any control or
logic over them. Not everyone will share it, so I made 
`AllowDefaultMemberInit` option for it, it's `false` by default.

#### Member functions

At first sight no member functions should be allowed. Since all data is `public`
there's no need for them. How about special members? Google guide says:
> Constructors, destructors, and helper methods may be present; however, these 
> methods must not require or enforce any invariants.

I feel that `struct` should not add *behavior*, only collect related data. Thus,
 this tool doesn't allow destructors and *helper methods*(whatever that means).
Hand-written copy/move constructors are also not allowed but you can still 
`=default` or `=delete` them. What's left I called *primary* constructors, that 
is, non-copy non-move constructors.    
Default constructor can be used instead of 
default member initialization(before C++20 default values for bit-fields could 
be set only in constructors) and they also can have a body. Having a body means
that constructor does something beyond trivial initialization and should be 
considered suspicious. Nevertheless, it's not
uncommon, hence another option: `AllowNonEmptyCtorBody` which is `false` by
default.    
First kind of a non-default constructor is what I called *memberwise* constructor,
it enforces the initialization of all members at once and looks like a good 
practice:

```cpp
struct Point{
    int x;
    int y;
    Point(int x, int y) : x{x}, y{y} {}
};
Point p1;       // error
Point p2{1};    // error
Point p3{1, 2}; // OK
```

Although there's no invariant nor contract, I consider this a poor interface.
It guarantees that initial `Point` contains valid and related coordinate values
but there's no guarantee that this relation will always be respected. The better
way is to have two separate types, one for coordinate values and 
another one to manipulate them:

```cpp
// pure data
struct Coordinate{
    int x;
    int y;
};
// manipulation interface
class Point{
public:
    Point(Coordinate c) : c{c}{}    // or Point(int x, int y);
    void Update(Coordinate c) { this->c = c; }
    //...
private:
    Coordinate c;
};
```

Second kind of a non-default constructor is purely custom one. It can have any
kind of parameters, related or not to `struct`'s data members. I
consider this a bad design because it either implies invariant or works as 
a conversion from another set of values, which should be implemented as a free 
function.    
With that in mind, I added another option: `AllowedCtors` which can have three
values: `none` - no constructors are allowed, `default` - only default constructors
are allowed, `primary` - all non-copy non-move constructors are allowed.

#### Inheritance

Things are simple here: only `public` inheritance is allowed and only another 
`struct` could be used as a base.

#### Conclusion

So what's a *pure data* `struct`? It's a `struct` that implies no invariant nor
contract on its members beyond theirs own ones, and doesn't add any kind of 
behavior to them. It should be used just to pack a set of logically related values. 
It should *not* be used to model a new type. In practice
it means good old C-`struct`s  with rare static members and inheritance.

## Implementation

Let's summarize options introduced above:
- `SkipStateless` - allows skip(`0`) or not(`1`) checking of `struct`s without
direct data members
- `AllowDefaultMemberInit` - allows(`1`) or not(`0`) default member initialization
- `AllowNonEmptyCtorBody` - allows(`1`) or not(`0`) non-empty body of allowed 
constructors
- `AllowedCtors` - specifies what kind of constructors are allowed. Possible values: 
`none` - no constructors are allowed, `default` - only default constructors
are allowed, `primary` - all non-copy non-move constructors are allowed    

Our matcher should detect bad `struct`s finding some bad parts in them. The 
top-level structure of a matcher is:

```cpp
cxxRecordDecl(
    isStruct(),
    // proceed only if SkipStateless == false or has any data member
    anyOf(boolean(!SkipStateless), has(fieldDecl())),
    anyOf(
        // bad member method
        // OR bad data member
        // OR bad base specifier
    ))
```

We want to check only user-provided methods(hand-written, non-defaulted,
non-deleted), and it's easier to specify what's allowed(static methods and
several kinds of constructors):

```cpp
// helper to check whether given constructor matches AllowedCtors requirements
AST_MATCHER_P(CXXConstructorDecl, shouldAllowCtor,
              NonDataStructsCheck::AllowedCtorKind, AllowedCtors) {
  if (Node.isCopyOrMoveConstructor() ||
      (AllowedCtors == NonDataStructsCheck::AllowedCtorKind::None)) {
    return false;
  } else if (AllowedCtors == NonDataStructsCheck::AllowedCtorKind::Primary) {
    return true;
  } else {
    return Node.isDefaultConstructor();
  }
}

// it should have trivial(empty) body or be explicitly allowed through config
const auto ShouldAllowNonEmptyCtorBody =
    anyOf(boolean(AllowNonEmptyCtorBody), hasTrivialBody());

cxxMethodDecl(isUserProvided(),
    unless(anyOf(isStaticStorageClass(),
                    cxxConstructorDecl(
                        shouldAllowCtor(AllowedCtors),
                        ShouldAllowNonEmptyCtorBody))))
```

Bad data members are either non-public or the ones with default member 
initializers(in case they are not allowed):

```cpp
// returns true when either AllowDefaultMemberInit is set or there's no default
// member initializer
const auto ShouldAllowDefaultMemberInit =
    anyOf(boolean(AllowDefaultMemberInit), unless(has(initListExpr())));

fieldDecl(
    unless(allOf(isPublic(), ShouldAllowDefaultMemberInit)))
```

And finally, bad base specifier is the one that's non-public or non-`struct`:

```cpp
anyOf(hasNonPublicBase(cxxRecordDecl()),
    hasDirectBase(
        cxxRecordDecl(unless(isStruct()))))
```

The full matcher:

```cpp
cxxRecordDecl(
    isStruct(), anyOf(boolean(!SkipStateless), has(fieldDecl())),
    anyOf(
        has(cxxMethodDecl(isUserProvided(),
                        unless(anyOf(isStaticStorageClass(),
                                        cxxConstructorDecl(
                                            shouldAllowCtor(AllowedCtors),
                                            ShouldAllowNonEmptyCtorBody))))
                .bind("method")),
        has(fieldDecl(
                unless(allOf(isPublic(), ShouldAllowDefaultMemberInit)))
                .bind("field")),
        anyOf(hasNonPublicBase(cxxRecordDecl().bind("np_base")),
            hasDirectBase(
                cxxRecordDecl(unless(isStruct())).bind("ns_base")))
    .bind("record"),
```

That's it, you can find the full source code [here](https://github.com/OleksandrKvl/clang-tidy-playground/blob/master/misc/NonDataStructsCheck.cpp). Maybe it doesn't
cover all possible cases but upgrading it to meet specific guide requirements
should not be hard.