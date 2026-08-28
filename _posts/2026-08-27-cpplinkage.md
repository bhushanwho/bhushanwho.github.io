---
layout: design-post
title: "cpp and linkages"
date: 2025-08-27 08:21:21 +0530
tags: [backend]
---

[google's c++ style guide](https://google.github.io/styleguide/cppguide.html#Internal_Linkage)

# the problem

every function at file scope gets external linkage by default. visible to the whole linker, not just the file it's in.

fine for stuff that need to be shared. bad for the small helpers every file collects that were never meant to leave it.

<pre><code class="language-cpp">
void fail(std::string_view msg) {
    std::cerr << "error: " << msg << "\n";
    std::exit(1);
}
</code></pre>

put this in main.cpp. put another fail() in scanner.cpp. now two files define the same symbol with external linkage. that's an ODR violation. linker either refuses to build, or does weird shit.

### fix 1: static (c style)

<pre><code class="language-cpp">
static void fail(std::string_view msg) { ... }
</code></pre>

gives internal linkage, one function at a time. works, but you have to remember it on every declaration. miss one out of thirty and it's still exposed.

### fix 2: named namespace

<pre><code class="language-cpp">
namespace ssgee_internal {
    void fail(std::string_view msg) { ... }
}
</code></pre>

doesn't actually solve it. still externally linked, just under a longer name. good for a real shared api (ssg::Scanner, ssg::Parser), not for hiding private helpers.

### fix 3: unnamed namespace

<pre><code class="language-cpp">
namespace {
    void fail(std::string_view msg) { ... }
    void printHelp() { ... }
}
</code></pre>

everything inside gets internal linkage. wrap the block once and forget about it.

unnamed namespaces are the modern idiomatic cpp equivalent of a million `static`s
