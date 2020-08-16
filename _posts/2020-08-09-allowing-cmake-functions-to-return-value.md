---
layout: post
title:  "Allowing CMake functions to return(value)"
date:   2020-08-09 13:55:09 +0300
post_url: 2020-08-09-allowing-cmake-functions-to-return-value
---

## Introduction

It's a story of implementing CMake feature that I call `command reference`
(similar to existing `variable reference`), i.e., using result of command 
invocation as an argument. Having this idea for a long time I never had enough 
time to dig into it. Now, being unemployed I decided at least to try it before 
looking for a next job. It was not as easy as I expected but I'm pretty satisfied 
with the result.

It consists of two parts:
1. First part contains motivation, design and results.
2. Second part explains some implementation details, such as why new lexer and 
parser is needed.

## Motivation

Most part of my career I used Visual Studio
and when switched to Linux I was slightly shocked. Compared to MSVS, makefiles 
felt like bows and arrows against machine gun. Then I discovered CMake and it felt
much better, instead of cryptic makefiles we got a distinct language with 
commands and variables. And since it's just another language, the same rules 
apply to its code: meaningful names, small functions, separation of abstractions, 
etc. Unfortunately, many CMake files look like one very big function 
that mixes everything in it. CMake allows us to handle almost all those things 
right except one - it doesn't have return values, thus, limiting the usefulness 
of function abstraction. As a result, some parts of your 
CMakeLists look bad.

Let's look at some examples.

```cmake
if(${CMAKE_CURRENT_LIST_DIR} STREQUAL ${CMAKE_SOURCE_DIR})    # is top level list?
if(WIN32)   # surprisingly short name comparing to other CMake vars
if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
if((CMAKE_CXX_COMPILER_ID STREQUAL "Clang") 
    OR (CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang"))
if(CMAKE_SIZEOF_VOID_P EQUAL 8)    # my favorite one, x64 check
```
The problem with such code is that it doesn't express logic, only implementation 
details. It requires you to remember all those long and tricky variable names,
magic values and relation between them.

We can move that into a function but it doesn't solve the problem. We are lazy,
nobody wants to write two lines instead of one:
```cmake
check_is_top_level_list(is_top_level_list)
if(is_top_level_list)...

# is it better than one-liner?
if(${CMAKE_CURRENT_LIST_DIR} STREQUAL ${CMAKE_SOURCE_DIR})
```

Almost all feature tests become:
```cmake
check_feature_available(RESULT is_feature_available)
if(is_feature_available)
```
It's better than direct values manipulation but it's ugly. Now you need to
check documentation for the name of this output argument, intuitive 
candidates are 
`RESULT`, `RESULT_VAR`, `OUTPUT`, or just absence of such argument at all: 
`check_feature_available(is_feature_available)`(or `check_feature_available()` with `FATAL_ERROR`).
It also forces you to think about names for variables, many 
of whom are used only once. In any popular language we can write all of the above 
in a clear manner:

```cpp
if(IsTopLevelProject()){}
if(IsWindowsBuild()){}
if(IsClang() || IsGcc()){}
if(IsX64Build()){}
if(IsFeatureAvailable()){}
```

Why should I know how those checks are performed?

Again, you can handle all that stuff, CMake has been successively used for years. But why not make it simpler? CMake is a big part of C++ so why 
shouldn't we make it easier for new people as we do with C++ itself?


## Design

### Why f(h()) isn't possible

Initial idea was to allow something like `get_name_by_id(get_id())` but it quickly
turned out to be wrong because of two reasons:
1. CMake syntax is too simple, it doesn't have keywords, command names are not 
restricted, anything is a string(including parens).
For example expression `if(x AND (y OR z))` means
call `if_impl("x", "AND", "(", "y", "OR", "z", ")");` where `AND`, `OR` and parens 
are just plain strings that are handled in a specific way by `if_impl`. 
The only requirement here is that parens should match, e.g. `if(x AND (y))))` 
isn't allowed. Because of that this is ambiguous:

    ```cmake
    function(AND a b c)
    endfunction()

    if(x AND (y OR z))  # if_impl(x, AND, (, y, OR, z, )) or
                        # if_impl(x, AND_impl(y, OR, z)) ?
    ```
You can extend this case to named arguments which, unlike bool operations,
can have non-trivial names.

2. But the main reason is that the above form isn't flexible enough. How to use it
within the quoted argument or to mix it with plain strings?

    ```cmake
    function(h)
    endfunction()

    f(a_h() "b h()_c")  # not possible
    ```

### Meet the command reference

Syntax mimics variable reference: `${command_name( args... )}`(notice, there's no 
spaces before command name and after final paren). It works 
just like you expect:

```cmake
function(get_name)
    return("Alex")
endfunction()

message(${get_name()})    # prints "Alex"
```

More generally:

```cmake
function(f)
    return(return-value-expr)
endfunction()
use_f(${f()})

# is equal to
set(__ret_var_name return-value-expr)
use_f(${__ret_var_name})
```

It can be used wherever variable reference can.
Comments, nested calls and lists are also allowed, let's mix it all together:

```cmake
function(format_name first last)
    return("First: ${first}, last: ${last}")
endfunction()

function(get_first_name)
    return("John")          # return quoted
endfunction()

function(get_last_name)
    return(Doe)             # return unquoted
endfunction()

function(get_first_and_last)
    return([[John]] Doe)    # return list
endfunction()

message(
    ${format_name(      # pass separate args
        ${get_first_name()}     # comments
        ${get_last_name()}      #[[ inside 
                                    command
                                    reference ]]
    )}
)                       # First: John, last: Doe

message(
    ${format_name(      # pass as a list, expands in two arguments
        ${get_first_and_last()}
    )}
)                       # First: John, last: Doe

# return() becomes a function that returns its arguments
message(${CMAKE_${return("VERSION")}})  # 3.18.1-...
```

## Downloads

Current implementation based upon CMake 3.18.1 release. You can build it 
from [sources](https://github.com/OleksandrKvl/CMake/tree/command-reference-3.18.1) or use pre-built binaries:
1. [cmake-3.18.1-win64-x64.zip](https://github.com/OleksandrKvl/cmake_cmdref_builds/raw/master/cmake-3.18.1-win64-x64.zip)
2. [cmake-3.18.1-Linux-x86_64.tar.gz](https://github.com/OleksandrKvl/cmake_cmdref_builds/raw/master/cmake-3.18.1-Linux-x86_64.tar.gz)

### Warning

*Some CMake features or policies, especially related to syntax or variable 
expansion, might not work. One such policy I'm aware of is OLD part of
[CMP0053](https://cmake.org/cmake/help/v3.1/policy/CMP0053.html). Syntax related 
error messages are also slightly different. All other things should work, I've
successfully built Google Test, Google Benchmark and fmt, using it. I'm not a
CMake developer, integration is quite dirty in some places so don't expect it to 
be production ready right away.*

## Part 2. Implementation details

At the beginning I naively supposed that if CMake can already parse single 
command  invocation, it would be enough just to call that function recursively on 
every argument :) But it turned out to be a way more complex and required 
completely new lexer and parser. I've created it using Flex&Bison, you can find
separate project that does parsing and pseudo-evaluation [here](https://github.com/OleksandrKvl/cmake_parser).

### Existing CMake parser and why it's not enough

Current implementation is relatively simple(but not its code). It consists of 
Flex-based scanner and hand-written parser.
Scanner detects separated arguments and their kinds,
it's easy since we know how each argument starts and ends. Current parser 
mostly verifies basic syntax rules like valid separations, parens matching etc.
For example
`command(a "${b}")` is parsed as `call("command").with_args(unquoted_arg{"a"}, 
quoted_arg{"${b}"})`. Notice that variable reference `${b}` is passed as a plain text.
During command execution each argument is parsed again with another parser that
can detect, verify and evaluate variable references. If such command appears in a 
cycle it does this additional parsing on every iteration.
Also if you make a mistake inside reference, it won't be detected until 
expression is evaluated:

```cmake
if(${ALWAYS_TRUE_IN_YOUR_ENV})
    # no errors or warnings on your machine
	message("hello world")
else()
    # syntax error at run-time on another machine
    message(${@:-:@})
endif()
```
Now, when we allow another command appear inside argument, argument separation is not so easy:
```cmake
command("result: ${get_result("a" b)}")
```
You can see that highlighter marks "a" in black because it thinks that arguments are
`"result: ${get_result("`, `a`, `" b)}"`. Existing CMake parser sees it in 
the same way.
To separate arguments correctly we got to be able to parse recursively
when we meet command reference.

In terms of BNF existing syntax looks like(simplified):

```bnf
command_invocation ::= identifier '(' argument* ')'
```

with command reference we got:

```bnf
command_invocation ::= identifier '(' (argument | command_invocation)* ')'
```

with only difference that command reference might appear inside argument, not only
as a separate one.

As you can see, now we need to parse it much deeper than existing parser does,
there's no sense in trying to extend it, also writing parser for recursive rules 
by hand is not trivial so I have no choice but to write both scanner and parser 
from scratch. Flex and Bison were chosen because they're already used
in CMake.

### BNF for a new syntax

Let's slightly update [official BNF](https://cmake.org/cmake/help/latest/manual/cmake-language.7.html#syntax) accordingly to new syntax:

```bnf
command_invocation  ::=  identifier space* '(' arguments ')'

quoted_argument     ::= '"' (quoted_element | reference)* '"'
unquoted_argument   ::= (unquoted_element | reference)+

reference           ::= var_reference | command_reference
var_reference       ::= var_ref_open (variable_name | reference)* ref_close
command_reference   ::= cmd_ref_open command_invocation ref_close

var_ref_open        ::= "${" | "$ENV{" | "$CACHE{"
cmd_ref_open        ::= "${"
ref_close           ::= "}"

quoted_element      ::= <check official docs>
unquoted_element    ::= <check official docs>
variable_name       ::= <check official docs>
```

Unlike existing implementation, I want to avoid parsing during execution and get
all details in one pass. Now, each quoted/unquoted argument consists of 
string(`quoted/unquoted_element+`) and reference. To get its real value
at run-time we need to evaluate and concatenate all its parts. For example, 
`a_${b}_c` has 3 elements: `string("a_")`, `var_ref("b")`, `string("_c")`. At 
run-time we get the value of `b` and concatenate them together: `a_B_VALUE_c`.

### Expression representation and evaluation

Here's brief overview of key expressions:
- call expression is a list of arguments.
- quoted/unquoted argument expression is a list of strings and references.
- variable reference expression is a list of strings and references
- command reference expression is similar to call expression.

Now we need a good representation that can store and evaluate 
such expressions efficiently.

#### AST

First approach was to use classic [Interpreter pattern](https://en.wikipedia.org/wiki/Interpreter_pattern) and compose expressions into a tree. Since each expression is 
list-like we can represent them all as a  `std::vector<std::unique_ptr<IExpression>>`. 
It works but even simple command becomes quite
involved, `command(a b)` is represented roughly with

```cpp
vector{         // vector of arguments
    "command",  // command name
    vector{     // each argument is a vector itself
        "a"
    },
    vector{
        "b"
    }
}
```

Things got worse when we add reference, `command(a ${b}_c)`:

```cpp
vector{
    "command",
    vector{
        "a"
    },
    vector{
        vector{     // reference is also a vector
            "b"
        },
        "_c"
    }
}
```

Too much vectors %)

#### RPN

[Reverse Polish(or postfix) Notation](https://en.wikipedia.org/wiki/Reverse_Polish_notation) is a notation when arguments comes before 
operator. It shines when you need to represent "linear" expression without branches, also it doesn't need parens to express precedence:

```
Normal(infix) notation: a + b
RPN: a b +

Normal: (a + b) * c
RPN: a b + c *
```

Now `command(a ${b}_c)` is represented with:
```cpp
vector<IExpression>{
    StringExpr{"command"},           // command name
    StringExpr{"a"}, UnquotedArg{1}, // 1 means number of subexpressions to concat
    StringExpr{"b"}, VarRefExpr{1}, UnquotedArg{1}, // same for VarRefExpr
    CallExpr{3}                      // 2 means number of arguments including name
}
```
One vector instead of four with AST approach, regardless how complex expression 
is, win :)

It also fits nicely a bottom-up parser like Bison because of the order in which 
symbols are discovered.
In example above, Bison will discover symbols exactly in 
their order in that vector, you can just push expressions without any knowledge
about previous symbols or other context.

#### Evaluation

RPN is evaluated using stack. Each expression knows its arity
(number of arguments), it pops them from stack and pushes back the result. But 
there's a little problem here. CMake expands list strings into multiple arguments:

```cmake
set(my_list a;b;c)
command(${my_list})       # called with 3 args: a, b, c
```

It means that if our CallExpr has arity = 1, at run-time it might become any 
number including zero. Classical RPN evaluation doesn't work here. To overcome 
this we need to adjust definition of `arity`: now arity means *number of 
expressions whose results should be taken as arguments*. And we need additional 
stack to track this *results count*. Consider RPN representation of the above example:

```cpp
{
    StringExpr{"command"},
    StringExpr{"my_list"}, VarRefExpr{1}, UnquotedArgExpr{1},
    CallExpr{2}
}
```

Take a look at both stacks before CallExpr evaluation for two cases:
1. my_list expands into 3 arguments
    ```cpp
    results:        {"command", "a", "b", "c"}
    results_count:  {1, 3}
    ```
CallExpr arity is 2, thus actual arity is the sum of last two elements in
results_count stack
and that will be the final number of its arguments `1 +3 = 4`.

2. my_list expands into 0 arguments
    ```cpp
    results:        {"commands"}
    results_count:  {1, 0}
    ```
Here, actual arity is `1 + 0 = 1`.

### Another small benefits of this implementation

#### Easy to change

Writing syntax rules in Bison makes it much easier to change, 
understand, review and support, then hand-written parser.

#### Symbol locations

Bison makes symbol locations tracking almost automatic. With [simple action]((https://github.com/OleksandrKvl/cmake_parser/blob/master/src/scanner.l#L73))
you only need to track lines manually.

#### Error messages

Bison's out-of-the-box error messages are pretty good:
```cmake
f(${@}) # 1.5 : syntax error, unexpected invalid token, expecting command name or 
        # reference opening or reference closing or variable name
```

#### BOM and line breaks handling

CMake supports BOM header but only UTF-8
is allowed. Instead of reading it [by hand](https://github.com/Kitware/CMake/blob/master/Source/LexerParser/cmListFileLexer.in.l#L421) 
we can handle it easily with another rule in parser.

CMake converts all `\r\n` into `\n` during [file reading](https://github.com/Kitware/CMake/blob/0cd3b5d0ca8d541fc3769f467db71a07a95be7f6/Source/LexerParser/cmListFileLexer.in.l#L326) by [replacing](https://github.com/Kitware/CMake/blob/master/Source/LexerParser/cmListFileLexer.in.l#L59) Flex's input routine.
Honestly, I can't fully understand that code. Supposedly it just replaces `\r\n`
with `\n` and `memcpy()` the rest, I want something better. In many places we can 
just use `\r?\n` regexp endings in scanner rules. In theory it's possible 
that string literal might contain `\r\r\n` which should become `\r\n`(I'm talking
about raw bytes `0x0D 0x0A`, not escapes). To handle this I remove trailing 
`\r` (if any) when `\n` is met in string literal on the fly in scanner. Since 
rules are written to take input line-by-line it doesn't involve much overhead. 
These simple solutions allow to eliminate custom reading routines and tons of 
`memcpy()` calls.


## Aftenotes

It's not an official CMake feature of course. If you like it, let me or CMake 
devs know to increase chances of having it in future CMake versions.