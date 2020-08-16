---
layout: post
title:  "Looking for a good C++ highlighter for your Jekyll blog? Try Prism.js"
date:   2020-08-09 13:48:07 +0300
post_url: 2020-08-09-use-prismjs-with-jekyll
---
What a nice title for the first blog post :) Almost as good as "How to start your 
blog". Anyway, I faced this problem and want to share the solution with others.

Starting a blog with GitHub Pages and Jekyll is easy, however I found that 
built-in highlighter(Rouge) has very poor C++ support. You can see how it looks
in the nice post by Lewis Baker [C++ Coroutines: Understanding the promise type](https://lewissbaker.github.io/2018/09/05/understanding-the-promise-type). At 
most it makes some parts bold and others dark blue(which looks almost black for me). 
Thus, I looked for another option. There were two candidates: [highlight.js](https://highlightjs.org/)
and [Prism.js](https://prismjs.com/). Both do pretty decent job but latter looks 
slightly better for me.

Here's how definition of `my_promise_type` looks now:
```cpp
struct my_promise_type
{
  void* operator new(std::size_t size)
  {
    void* ptr = my_custom_allocate(size);
    if (!ptr) throw std::bad_alloc{};
    return ptr;
  }

  void operator delete(void* ptr, std::size_t size)
  {
    my_custom_free(ptr, size);
  }
  ...
};
```

### Installation

1. Disable built-in highlighter in your `_config.yml`:
    ```
    kramdown:
        syntax_highlighter_opts:
            disable : true
    ```
2. Take links to prism.js scripts and themes from [cdnjs](https://cdnjs.com/libraries/prism).
You need `prism.min.css`, `prism-line-numbers.min.css`, `prism-core.min.js`,
`prism-autoloader.min.js`(it will download highlighters for languages used on your page on-the-fly),
`prism-line-numbers.min.js`. You can omit `prism-line-numbers.*` if line numbers
aren't required.

3. Add each theme to `<head>` section of your blog like `<link href="CSS_LINK" rel="stylesheet" />`.
Add each script to the bottom of `<body>` like `<script src="SCRIPT_LINK"></script>`.
You can read how to edit and where to find those files [here](https://jekyllrb.com/docs/themes/#overriding-theme-defaults).

That's it. If you like me are little crazy about *don't pay for what you don't use*
principle, you can set it up only for posts. For this you need to make above steps
for `post` layout. Check my blog [repository](https://github.com/OleksandrKvl/oleksandrkvl.github.io) for an example.

### The final tip
The C++ language name in prism.js is `cpp`, not `c++`; thus, you
should use *\`\`\`cpp* code blocks, not *\`\`\`c++*.