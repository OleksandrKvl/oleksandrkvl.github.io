---
layout: post
title:  "Creating your own clang-tidy checks without building LLVM"
date:   2020-10-06 17:10:00 +0300
post_url: 2020-10-06-building-clang-tidy-without-llvm
---

### What is clang-tidy

[clang-tidy](https://clang.llvm.org/extra/clang-tidy/) is a static analysis tool
based on `Clang`'s [LibTooling](https://clang.llvm.org/docs/LibTooling.html) 
library. It can find and sometimes fix subtle problems 
in your code or just make it look better. The [list of checks](https://clang.llvm.org/extra/clang-tidy/checks/list.html)
is pretty extensive.

### Create your own tool

What's more interesting is that you can create your own tool that can detect/fix
some problems with your code, enforce your custom coding style, refactor it and so 
on.

There are two options for that:
- use [LibTooling](https://clang.llvm.org/docs/LibTooling.html)
- [create](https://clang.llvm.org/extra/clang-tidy/Contributing.html) custom `clang-tidy` check

When I started, I chose the `LibTooling` way. But it turned out that it's
pretty low-level, you need to understand a lot more things, and write more 
boiler-plate code to create useful tool.
Custom `clang-tidy` check is a much better option, it's actively supported, you can 
use existing checks as an example for your own ones. It has the whole 
infrastructure 
like diagnostic messages, deduplication of fix-it replacements, `run-clang-tidy.py` 
to run your check in parallel and so on.

### Little problem

The only problem with `clang-tidy` check is that according to official manual you 
have to build it from sources which means you have to build the whole LLVM. Why 
should someone who wanted to play with simple checks to build LLVM? And I'm afraid
to imagine how long it will take on my 2013 2-core MBP laptop.

### Solution

So, I wanted to have full `clang-tidy` infrastructure without building LLVM.
Thankfully, LLVM has Debian/Ubuntu [prebuilt packages](https://apt.llvm.org/) 
which include
all required libraries for `clang-tidy`. The only remaining thing is to link
`clang-tidy` sources with it. It turned out to be pretty simple. We need to edit
three `CMakeLists.txt`s, replace parts responsible for building libraries from 
sources with parts that do link to static libraries. 
I've removed all checks to make it as light as possible. Now you can follow the
[official manual](https://clang.llvm.org/extra/clang-tidy/Contributing.html), use
`add_new_check.py` to create a new check, build it with prebuilt packages, and run 
it with `run-clang-tidy.py` in parallel.

The resulting repository is [here](https://github.com/OleksandrKvl/clang-tidy-standalone).
It requires LLVM 10 packages. Porting it to next versions should be simple but 
it's not in my plans right now.