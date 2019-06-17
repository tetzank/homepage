---
title: "Removing Unused Code"
date: 2019-01-05T23:25:03+01:00
---

I recently went on a little journey to get a better understanding of the linker used in C++.
Like many other C++ developers, my mental model of what the linker actually does is very limited.
But I'm always interested in learning new tricks about the tools I use regularly, especially new optimizations slumbering behind some obscure flag.
In this post, we will have a look at the removal of unused functions from the executable and how the linker can help us with this task.

Wait a minute. In a well-maintained project there shouldn't be any unused function, right?
Well, it happens more often than you think that some function was left behind after some refactoring and nobody notices that it is not required anymore.
And there's another important case: static linking.
Usually a program which statically links another library is not using every function of the library.
Only a subset of the functions is called.
All other functions are unused.

Surprisingly, a lot of C++ developers assume that today's optimizing compilers just take care of it.
The compiler should be able to see that a function is never called and, therefore, exclude it from the executable.
Let's put that to a test and see for ourselves.


# Example Program

The following tiny C/C++ program shall be our test case.

{{< highlight "C++" >}}
#include <stdio.h>

void unused(){
	puts("This is dead code which is never called.");
}

int main(){
	return 0;
}
{{< / highlight >}}

The program does not do much.
It just returns the return code of 0, nothing more.
Obviously, the function `unused()` is never called.
Let's compile it with optimizations turned on.

```
$ g++ -O2 -march=native deadcode.cpp -o deadcode
```

I will focus on "standard" tools running on Linux, mostly tools from GCC and binutils.
You might get other results with different toolchains.
I will usually check if clang+LLVM have a different behavior and mention it, if they do.
Here we use the C++ frontend `g++` of GCC.
Compiling the program as C or C++ makes no real difference.


# Inspecting Executable

Now that we have an executable, let's have a look if `unused()` survived all the optimization passes of the compiler and made its way into the executable.

```
$ objdump -tC deadcode | grep 'unused'
0000000000001150 g     F .text  000000000000000c              unused()
```

`objdump` should be your goto tool if you want to inspect an executable.
We are interested in the functions contained in the executable.
The option `-t` dumps all of them by looking at the symbol table in the executable
and `-C` demangles all symbols so that we get C++ function names (a step not required for C programs).
We pipe the output through `grep` to only get the entry we are looking for, or nothing if the function was removed.

As we can see, `unused()` ended up in the executable.
The question is: why?
Why was the compiler unable to see that the function is never called?
The answer is simple: it can't.
We did not provide enough information to the compiler.
For example, we know that there is just one .cpp-file and that we link it to a final program and not a library.
Both things we did not tell the compiler explicitly.
It gets clearer when we expand the one-liner we used for compilation to the commands which get run (simplified, pass `-v` to see the real commands).

```
$ g++ -O2 -march=native deadcode.cpp -o deadcode.o
$ g++ deadcode.o -o deadcode
```

First, all .cpp-files are compiled to object files, isolated from each other in their own process.
Afterwards, all object files are linked together to the executable.
The function `unused()` in deadcode.cpp has [external linkage][linkage].
Hence, it can be called potentially from other .cpp-files.
The compiler processing deadcode.cpp does not know that there is no external call to `unused()`.


# Internal Linkage

One simple solution is to change `unused()` to internal linkage.
Static functions are only visible inside the same .cpp-file and, therefore, have internal linkage.

{{< highlight "C++" >}}
static void unused(){
	puts("This is dead code which is never called.");
}
{{< / highlight >}}

C++ has another possibility: anonymous namespaces.
An unnamed namespace is also just visible inside the same .cpp-file.
Every function or class declared inside of it has internal linkage.

{{< highlight "C++" >}}
namespace{

void unused(){
	puts("This is dead code which is never called.");
}

} // anonymous namespace
{{< / highlight >}}

The compiler can now see that `unused()` is only visible inside deadcode.cpp and there is no call to it in deadcode.cpp.
Hence, the function is unreachable and removed as dead code.

Instead of changing the file by hand, we can use a special flag of GCC.

```
$ g++ -O2 -march=native -fwhole-program deadcode.cpp -o deadcode
```

`-fwhole-program` lets the compiler assume that there is just one .cpp-file and, therefore, set every function except main to internal linkage.
This will remove `unused()` without any source changes.


# Garbage Collection with the Linker

Obviously, internal linkage cannot help with static libraries.
The whole point of functions in a static library is to be called from other .cpp-files.
They need to have external linkage.
Different .cpp-files can call different subsets of functions from a static library.
The compiler processing each .cpp-file in isolation always sees just one subset.
The only tool which has the full picture is the linker.
It knows all object files and all libraries involved in the compilation.
We have to instruct it explicitly to remove unused code.

```
$ g++ -O2 -march=native -ffunction-sections -Wl,--gc-sections deadcode.cpp -o deadcode
```

`-ffunction-sections` puts every function in its own ELF section.
The linker only sees sections as a black box.
`--gc-sections` tells the linker to track calls into sections and remove unused ones.
`-Wl,<linker-flag>` passes a flag to the linker.
With both flags enabled, the linker can construct a call graph for the whole program and remove `unused()` even when it has external linkage.

There are some caveats attached to this solution.
Putting every function in its own section adds quite some size overhead to the executable when it is not just a toy program.
In case of static linking, the static library has to be compiled with `-ffunction-sections` too.


# Link-Time Optimization

As we learned, the linker is in a nice position to see every part of a program.
Many optimization passes, not just dead code elimination, benefit from a bigger picture of the program.
Therefore, modern compilers and linkers provide link-time optimization (LTO).

```
$ g++ -O2 -march=native -flto deadcode.cpp -o deadcode
```

`-flto` lets the compiler and its optimization passes see the whole program during linking.
It sees that `unused()` is not called in any .cpp-file and removes it as dead code even with external linkage.

Same as before, static libraries have to be compiled with LTO enabled, otherwise the compiler cannot reason about it and remove unused code.
But the main problem of LTO hindering its widespread adoption is high memory consumption and increased compilation time.
The sequential step of linking is now also used to run optimization passes on bigger parts of the program.
Linking is not trivially parallelizable like compiling .cpp-files to object files.
Both, GCC and LLVM are working on improving scalability of LTO, e.g., LLVM with [ThinLTO][thinlto].


# Conclusion

Coming back to our initial question if the compiler can safely remove unused code: yes, the compiler can do it if you ask in the right way.
It is not done by simply enabling optimizations.
LTO is the best variant in my opinion as it also enables a lot of other optimizations to be applied to your whole program.
If you can tolerate the scalability issues of the compilation, LTO is the way to go.


[linkage]: https://en.cppreference.com/w/cpp/language/storage_duration#Linkage
[thinlto]: http://blog.llvm.org/2016/06/thinlto-scalable-and-incremental-lto.html
