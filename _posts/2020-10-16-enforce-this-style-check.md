---
layout: post
title:  "Enforce explicit/implicit 'this' with custom clang-tidy check"
date:   2020-10-16 21:34:00 +0300
post_url: 2020-10-16-enforce-this-style-check
---

### Introduction

Recently, I've discovered an interesting topic: `clang-tidy`-based tools. The idea
is that you get an AST representing all the details of your C++ code, what you
can do with it is limited mostly by your imagination: detect bugs, calculate
some code metrics, refactor, etc. You can take your old legacy codebase and convert
it into a modern one. This idea of making changes at scale really fascinates me.
To learn something you have to use it in a real-world. As I've recently made
several contributions to `CMake` codebase, the idea quickly popped up in my mind for
a refactoring tool. `CMake` has one strange part in its coding conventions which I've never seen in any other C++ codebase: explicit `this` usage. That is, 
like in JS or Python:

```cpp
class Widget{
    void Increment(){
        this->x++;
    }
    int x;
};
```

Because there's no way to check it automatically, there are places where it's not 
respected. I don't know how this
style was adopted, maybe it's just some old artifact that is hard to get rid of
by hand. So, I decided to write a tool that can do the both:
- add explicit `this` if it's missed
- remove explicit `this` wherever possible

### Workflow outline

Writing this `clang-tidy` tool involves several steps:
- understand what C++ code you want to fix
- find out how to detect it using `Clang AST API`
- fix it

I will describe explicit/implicit parts separately after describing common things.
I used `clang-tidy-standalone` as a base for this tool, you can build it without
build LLVM itself, more information in my [previous article](/2020/10/06/building-clang-tidy-without-llvm.html).    
Notice that this is not a complete `Clast AST` tutorial, you can find more
information in the [official documentation](https://clang.llvm.org/extra/clang-tidy/Contributing.html)
, various youtube talks, and `clang-tidy` [sources](https://github.com/llvm/llvm-project/tree/master/clang-tools-extra/clang-tidy).

#### Templates handling

Since template is only an outline for generated code, they are represented 
differently from non-templated code. In non-templated code all the types, 
variables and functions are known and checked. In template it's not always possible
before the actual instantiation. As a result, they are represented with different
AST nodes. You have a choice: deal with template definition which contains unknown 
or type-dependent entities, or deal with instantiations where everything is known.
You should know whether your check can produce different results for different 
template instantiations. Thankfully, that's not the case for this tool, so I will
deal only with instantiations. This in turn requires that all of your templates
are used somewhere in a project or in a test suite so they are actually instantiated.

#### Macros handling

Since macros are just text replacements, they can have very different meaning in 
different contexts. Final AST represents code after preprocessing, so your
tool can detect things that were composed from macro expansions. In most cases
you can just skip such code, macro usage should be rare nowadays. But there's at
least one macro which I want to handle - `assert()`. It naturally contains things
that I want to fix, for example:

```cpp
void Widget::Reset(){
    assert(this->ptr);
    *ptr = 0;
}
```

For this reason, I've added simple regex-based macro filter:

```cpp
  if (ThisLocation.isMacroID()) {
    const auto MacroName =
        Lexer::getImmediateMacroName(ThisLocation, SM, getLangOpts());
    if (!llvm::Regex(AllowedMacroRegexp).match(MacroName)) {
      return false; // skip
    }
  }
  // continue...
```

Another example of widely used macro is various loggers, my final macro-filter for 
CMake 
codebase looks like this: `^(assert|cm.*Log|cm.*Logger)$`. Keep in mind that
we only can handle things that are present after preprocessing, eliminated `#ifdef` 
blocks wouldn't be there, so run your tool on various configurations.

### Enforce explicit `this`

#### Target C++ code

Let's start with the easier case, enforcing explicit `this`. Here's our test case:

```cpp
class Widget{
    void Do(){
        DoConst();      // should become this->DoConst();
        x++;            // should become this->x++;
    }
    void DoConst() const{}
    int x{};
};
```

That is, every access to member should become explicit. Since we're dealing with
already valid code and adding explicit `this` wouldn't change its meaning, there's 
nothing more to consider, we just need to find such
places, check whether they have explicit `this` or not, and add it if missed.

#### Detecting it with Clang AST API

`clang-query` is a useful tool to examine generated AST, I left only important
parts:

```
clang-check-10 --ast-dump example.cpp --

  |-CXXMethodDecl 0x1ef3a88 <line:2:5, line:5:5> line:2:10 Do 'void ()'
  | `-CompoundStmt 0x1ef3e18 <col:14, line:5:5>
  |   |-CXXMemberCallExpr 0x1ef3d88 <line:3:9, col:17> 'void'
  |   | `-MemberExpr 0x1ef3d58 <col:9> '<bound member function type>' ->DoConst 0x1ef3ba8
  |   |   `-ImplicitCastExpr 0x1ef3da8 <col:9> 'const Widget *' <NoOp>
  |   |     `-CXXThisExpr 0x1ef3d48 <col:9> 'Widget *' implicit this
  |   `-UnaryOperator 0x1ef3e00 <line:4:9, col:10> 'int' postfix '++'
  |     `-MemberExpr 0x1ef3dd0 <col:9> 'int' lvalue ->x 0x1ef3c60
  |       `-CXXThisExpr 0x1ef3dc0 <col:9> 'Widget *' implicit this
```

You can see that our target parts are represented as `MemberExpr` and `CXXThisExpr`
with optional `ImplicitCastExpr`. Cast is there because we're calling `const`
function from `non-const` one, hence, casting `Widget*` to `const Widget*`. AST 
matcher for it is straightforward:

```cpp
memberExpr(has(
    ignoringImpCasts(
        cxxThisExpr().bind("thisExpr"))))
.bind("memberExpr")
```

`bind()` is needed to get access to the matched node, in our case we need
`MemberExpr` and `CXXThisExpr`, thus, we bind them to names. In the [CXXThisExpr documentation](https://clang.llvm.org/doxygen/classclang_1_1CXXThisExpr.html#a933d8a76b980a003a6bcfb7bb583aa5a)
we can see `isImplicit()` method that does exactly what we need:

```cpp
void EnforceThisStyleCheck::check(const MatchFinder::MatchResult &Result) {
  const auto ThisExpr = Result.Nodes.getNodeAs<CXXThisExpr>("thisExpr");
  const auto MembExpr = Result.Nodes.getNodeAs<MemberExpr>("memberExpr");
  // ...
  if (ThisExpr->isImplicit()) {
    addExplicitThis(*MembExpr);
  }
}
```

#### Fix

Fixing is really simple, `clang-tidy` has a lot of examples of it, we have to 
provide hint, location, and text for our fix:

```cpp
void EnforceThisStyleCheck::addExplicitThis(const MemberExpr &MembExpr) {
  const auto ThisLocation = MembExpr.getBeginLoc();
  diag(ThisLocation, "insert 'this->'")
      << FixItHint::CreateInsertion(ThisLocation, "this->");
}
```

We use `MemberExpr`'s location instead of `CXXThisExpr`'s because in case of
qualified names(`Base::Method();`) `CXXThisExpr::getBeginLoc()` points to the start of `Method`, not the start of a namespace.

### Enforce implicit `this`

#### Target C++ code

This case is a bit harder because in some cases removing explicit `this` could 
change the meaning of code due to name lookup rules, in other cases it could 
result in a compilation error.

#### Special members

We can't remove explicit `this` from a special member functions like destructors
or operators:

```cpp
void Widget::Do(){
    this->~Widget();
    this->operator=(Widget{});
}
```

This can happen only when member expression refers to a method, not to a variable.
Thus, we need to get member declaration, check whether it's a method, and then
check it's name:

```cpp
static bool isNonSpecialMember(const MemberExpr &MembExpr) {
  const auto MemberDecl = MembExpr.getMemberDecl();
  assert(MemberDecl);

  const auto MethodDecl = dyn_cast<CXXMethodDecl>(MemberDecl);
  // CXXMethodDecl::getIdentifier() returns nullptr for special members
  return !MethodDecl || MethodDecl->getIdentifier();
}
```

#### Name conflicts

Consider this case:

```cpp
void Widget::Do(int x){
    this->x++;  // increment member
    x++;        // increment argument
}
```

If we remove explicit `this` from the expression at line 2, it will increment 
argument instead of data member. Generally, any visible local name hides class member name during the lookup. Unfortunately,
`Clang` doesn't have the API to detect such conflicts, so I choose less precise
but easier to implement way(thanks to Nicolás Alvarez for this idea):

```cpp
static bool hasVariableWithName(const CXXMethodDecl &Function,
                                ASTContext &Context, const StringRef Name) {
  const auto Matches =
      match(decl(hasDescendant(varDecl(hasName(Name)))), Function, Context);

  return !Matches.empty();
}
```

This method enumerates all declared variables(including arguments) in the 
function, ignoring
their visibility. It means that this
code will be untouched even if it's safe:

```cpp
void Widget::Do(){
    this->x++;  // increment member
    x++;        // still increment member but confusing
    int x;
    x++;        // increment local variable
}
```

#### Dependent names

```cpp
template<typename Base>
class Derived : public Base{
    void Do(){
        this->baseCounter++;    // baseCounter is defined somewhere in Base
    }
};
```

C++ requires dependent member names to be prepended with explicit `this`, thus, 
removing
it here will yield a compile-time error. In our case it means is that if name is
provided by the base class, explicit `this` is required. So, removing explicit
`this` from a name is safe when this name is a direct(non-inherited) member of
a class:

```cpp
static bool hasDirectMember(const CXXRecordDecl &Class, ASTContext &Context,
                            const StringRef Name) {
  const auto Matches =
      match(cxxRecordDecl(has(namedDecl(hasName(Name)))), Class, Context);

  return !Matches.empty();
}
```

Now, we can create our final `isRedundantExplicitThis()` function:

```cpp
static bool isRedundantExplicitThis(const MemberExpr &MembExpr,
                                    const CXXMethodDecl &MethodDecl,
                                    ASTContext &Context) {
  return (isNonSpecialMember(MembExpr) &&
          !hasVariableWithName(MethodDecl, Context,
                               MembExpr.getMemberDecl()->getName()) &&
          !isDependentName(MethodDecl, MembExpr, Context));
}
```

And, because we need access to the corresponding `CXXMethodDecl`, our final 
matcher for both cases becomes:

```cpp
cxxMethodDecl(
    isDefinition(), unless(anyOf(isImplicit(), isDefaulted())),
    forEachDescendant(
        memberExpr(has(ignoringImpCasts(cxxThisExpr().bind("thisExpr"))))
            .bind("memberExpr")))
    .bind("methodDecl")
```

`unless(anyOf(isImplicit(), isDefaulted())` part is needed to skip 
compiler-generated function definitions.

#### Fix

Again, fixing is mostly simple. We have to provide hint and range for removal.
Qualified names require special handling because we don't want to remove 
namespace part.

```cpp
void EnforceThisStyleCheck::removeExplicitThis(const SourceManager &SM,
                                               const MemberExpr &MembExpr) {
  const auto ThisStart = MembExpr.getBeginLoc();
  auto ThisEnd = MembExpr.getMemberLoc();
  if (MembExpr.hasQualifier()) {
    ThisEnd = MembExpr.getQualifierLoc().getBeginLoc();
  }

  const auto ThisRange = Lexer::makeFileCharRange(
      CharSourceRange::getCharRange(ThisStart, ThisEnd), SM, getLangOpts());

  diag(ThisStart, "remove 'this->'") << FixItHint::CreateRemoval(ThisRange);
}
```

### Results

Applying to CMake codebase:
- enforce explicit `this`: 129 files changed, 4689 insertions
- enforce implicit `this`: 406 files changed, 23237 insertions

Full source code is [here](https://github.com/OleksandrKvl/clang-tidy-playground).    
CMake branch with explicit `this` is [here](https://gitlab.kitware.com/OleksandrKvl/cmake/-/tree/explicit-this-fix).    
CMake branch with implicit `this` is [here](https://gitlab.kitware.com/OleksandrKvl/cmake/-/tree/implicit-this-fix).

I'm pretty satisfied with the result. The whole tool takes  <170 lines of code.
Hope that in future there will be more good tutorials to make this framework 
more available to more people.