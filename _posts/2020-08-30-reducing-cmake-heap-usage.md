---
layout: post
title:  "Reducing CMake heap usage with Heaptrack"
date:   2020-08-30 23:02:00 +0300
post_url: 2020-08-30-reducing-cmake-heap-usage
---
### Introduction

While working on [previous CMake experiment](https://oleksandrkvl.github.io/2020/08/09/allowing-cmake-functions-to-return-value.html) 
I noticed several places that looked suspiciously suboptimal. As in case of any
performance considerations everything should be measured so I decided to run it
through a [Heaptrack](https://github.com/KDE/heaptrack) and check what's going on.

### Clean run

I used Heaptrack 1.2.80, CMake 3.18.1 with debug symbols and this barely empty 
CMakeLists.txt:

```cmake
cmake_minimum_required(VERSION 3.17)
project(empty)
```

Let's take a look at what heaptrack gave us:

![heaptrack_clean](/assets/images/heaptrack_clean.jpg)

I don't expand all entries but you can see that top two heap consumers(1 and 2) 
are related to `std::vector<cmListFileFunction>` and 
`std::vector<cmListFileArgument>` operations which in turn stem from 
`cmFunctionBlocker::IsFunctionBlocked()`(3 and 4). This is exactly what I 
expected. Now let's look at these types and their role.

### Representation

These types are used to represent parsed code. Here are their simplified 
definitions:

```cpp
// command argument representation
struct cmListFileArgument
{
    std::string value;  // argument itself
    Delimiter delim;    // argument's type: quoted/unquoted/bracket
    long line;          // argument's position
};

// command representation
struct cmListFileFunction
{
    std::string nameLower;
    std::string nameOriginal;
    long line;
    std::vector<cmListFileArgument> arguments;
};

// file representation
struct cmListFile
{
    std::vector<cmListFileFunction> functions;
};
```

For example, representation for this code
```cmake
Find_Package(Boost 1.41.0 COMPONENTS system filesystem iostreams)
```

would be

```cpp
cmListFile
{
    std::vector<cmListFileFunction>
    {
        cmListFileFunction
        {
            std::string{"find_package"},
            std::string{"Find_Package"},
            long{1},
            std::vector<cmListFileArgument>
            {
                {std::string{"Boost"}, Unquoted, long{1}},
                {std::string{"1.41.0"}, Unquoted, long{1}},
                {std::string{"COMPONENTS"}, Unquoted, long{1}},
                {std::string{"system"}, Unquoted, long{1}},
                {std::string{"filesystem"}, Unquoted, long{1}},
                {std::string{"iostreams"}, Unquoted, long{1}},
            }
        }
    }
};
```

When the file is parsed, `cmListFile` contains all its commands. After, they
will be executed in a row.

#### Function blocker

Everything in CMake is a command including things like functions, conditionals 
and cycles. Hence, it needs a way to represent a *scope*, 
e.g. it can't just execute a function body on its first occurence. Instead, it 
needs to collect function's
inner commands as its body and create a new function definition. To achieve that CMake
has *function blocker* which has that `IsFunctionBlocked()` function:

```cpp
class cmFunctionBlocker
{
public:
    bool IsFunctionBlocked(cmListFileFunction const& function)
    {
        //...
        function.push_back(function);   // copy!
        return true;
    }
    //...
private:
    std::vector<cmListFileFunction> functions;
};
```

When *starting* command(e.g. *function()*, *if()*) is executed, it starts
to collect scope body using `IsFunctionBlocked()` and when *ending* command(*endfunction()*, *endif()*) is 
met, it does something with its body. The important moment here: once the body is 
collected, it will never be modified.

### The problem

As you can see it copies functions inside.
Consider how this simple code would be represented:

```cmake
function(f)
    # function() block
    if(${var})
        # if() block
        message("hello world")
    endif()
endfunction()

f()
```

Here's how it's stored in memory:
- `cmListFile` - stores commands at lines [1; 6] (ranges are closed)
- `function() blocker` - stores commands at lines [2; 6]
- `if() blocker` - stores commands at lines [4; 5]

That's the problem, blockers copy same commands multiple times depending on 
the code structure. It's quite expensive because each command contains two strings
(for its name) and a vector of strings(for arguments).

#### Solution

My first thought was to store raw pointers in blockers but it doesn't work. When 
it reads dependent file(via *include()*) it really needs to copy `cmListFileFunction` 
because corresponding `cmListFile` is destroyed after parsing but we still need
to use included functions. So the actual
solution is to store each command in a `std::shared_ptr` to make copy cheap:

```cpp
// old cmListFileFunction
struct cmListFileFunctionImpl
{
    std::string nameLower;
    std::string nameOriginal;
    long line;
    std::vector<cmListFileArgument> arguments;
};

using cmListFileFunction = std::shared_ptr<cmListFileFunctionImpl>;
```

#### Another little problem

Second place where it does unnecessary copy of commands is in 
`ExecuteCommand()` function. Although it's not critical, I still think obviously
unneeded copy operations should be avoided.

```cpp
using Command = std::function<bool(std::vector<cmListFileArgument> const&)>;

std::map<std::string, Command> RegisteredCommands;

Command GetCommandByExactName(std::string const& name);

bool ExecuteCommand(const cmListFileFunction& function)
{
    //...
    if (auto command = GetCommandByExactName(function.nameLower))   // copy!
    {
        // execute
        command(function.arguments);
    }
    //...
}
```

As you can see, commands are stored in a `std::function` and during 
execution that object is copied. Why? Because commands could be redefined, thus
currently executing function object might be reassigned and the following 
execution would be UB if it's not stored anywhere.
Copy of `std::function` usually involves copy of its control block where the 
actual Callable is stored. Most built-in commands in CMake are stored as a raw 
function pointers so their copy is relatively cheap. But user-defined functions
(*function()/endfunction()*) contains a vector of commands in its blocker and its copy is not cheap.
Again, solution is to use `std::shared_ptr`:

```cpp
using CommandPtr = std::shared_ptr<Command>;
std::map<std::string, CommandPtr> RegisteredCommands;
CommandPtr GetCommandByExactName(std::string const& name);
```
No more excessive copies of vectors of strings. Both solutions could be even 
better with something like `boost::local_shared_ptr` which avoids synchronization
overhead.

### Optimized run

Let's measure those changes.

![heaptrack_optimized](/assets/images/heaptrack_optimized.jpg)

Now top heap consumers(1 and 2) are related to `std::string` operations. Some of 
them also could be fixed but it requires more effort to get significant 
improvement.
Total bytes allocation decreased from 65MB to 39MB. Number of allocations 
decreased from 394k to 280k.

For more complex configurations the economy is of course lower. Partly because 
there's an old parsing routine that allocates a lot and becomes the major memory 
consumer. Here are results for heaptrack itself and Google benchmark 
configuration step (total allocated bytes (number of allocations)).

Heaptrack:
- before 305 MB (1308k)
- after 268 MB (1148k)

Google benchmark:
- before: 233 MB (1344k)
- after: 196 MB (1190k)

### Conclusion

I'm not a fan of deep optimizations in a project like CMake where resources
are not scarce and actual execution is rare. However, in this case it's
not really an *optimization*, just avoidance of unneeded copy
operations, just the C++ way of doing things. 