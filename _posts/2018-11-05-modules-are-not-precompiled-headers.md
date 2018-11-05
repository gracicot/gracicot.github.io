---
layout: default
title:  "Modules are not precompiled headers"
date:   2018-11-05 3:00:00 +0500
categories: modules
excerpt_separator: <!--more-->
---

# Modules Are Not Precompiled Headers

Modules have been the subject of many controversies in the C++ community in the recent past and there seem to be some misconception floating around recently about modules. Important decisions will soon be made, and I wanted to clear some of the facts and raise potentially good questions about modules and the path to their adoption.

<!--more-->

## What A Time To Be A C++ Programmer!

I'm writing this ~~November 4th, it's about 19:00~~ the November 5, it's about 3 in the morning, and today is gonna be the start of an important event: the 2018 San Diego meeting. There is about 276 papers in the pre-meeting mailing, which is an absurdly large amount.

Another thing from the San Diego meeting worth noting is that it's the last meeting to add big features into C++20. After that, the language enters a feature freeze and mostly fixes and minor things will be considered to be added into C++20. In this meeting, a lot of impactful decisions will be made and many will try to make their favourite feature into the next C++ revision. One of the main big feature awaited for C++20 is modules. It's one of the biggest change to be added and may be our chance rejuvenate the language.

## The Module Interface

What's the module interface? Something really different from a header, much more similar to a translation unit. In fact, the module interface is a translation unit, like any other. There is a subtle difference: it contains everything the compiler needs to know about what entities should be available in a module, and there can be only one module interface by modules.

The key point here is that translation units that contain exported declarations _are translation units_. They actually can be compiled. Here's a simple module example:

```c++
/// hello.cpp
module;
#include <cstdio>
export module hello;

export namespace hello {
    int global_data;
    
    void say_hello() {
        int data = global_data;
        std::printf("hello modules! data is %d\n", data);
    }
}

/// main.cpp
import hello;

int main() {
    hello::global_data = 3;
    hello::sayhello();
}
```

I can compile this all at once and run it:

    g++ -fmodules-ts hello.cpp main.cpp && ./a.out

This outputs two files actually: `a.out` and `hello.nms`, our BMI (Binary Module Interface) for `hello.cpp`.

I can also compile hello independently because again, `hello.cpp` is a translation unit, not a header:

    g++ -fmodule-ts -c hello.cpp -o hello.o

This also outputs two files: `hello.o` and `hello.nms`. The file `hello.o` contains the compiled code from our `hello.cpp` translation unit. Additionally, this file `hello.o` compiled using modules and exported fancy stuff can be used to link against non-modular code, since the `hello.o` file is the same as it is were compiled as non-modular code.

## The BMI And Recompilation

One could argue that such code as above is a bad design. If one would change ever so slightly the implementation of `hello::say_hello`, it would cause a runaway recompilation of all file that transitively imports it!

The reality is that it's may not true. As you know, recompilation is defined by the build system when it judges it necessary. When editing the module interface the BMI may change or not. Some compiler, such as GCC and MSVC chose to output information in a way that even if the implementation changed, the BMI stay untouched not.

Here's a hash of the BMI of `hello.cpp`:

```
> sha1sum hello.nms
36859455fa50c5455429ddca674b38bd5483e8f6  hello.nms
```

Let's change our function to something like this:

```c++
void say_hello() {
    int data = global_data + 1;
    std::printf("hello modules! data is %d (it's a lie!)\n", data);
}
```

Let's see if the BMI changed at all:

```
> sha1sum hello.nms
36859455fa50c5455429ddca674b38bd5483e8f6  hello.nms
```
Hurray! No recompilation of `main.cpp`!

Since imporing module do not need to see the definition of `hello::say_hello`, the compiler can simply export the declaration in the BMI. Minus some bugs that caused random changes, the compiler always yielded the same BMI reguardless of the implementation.

> What about inline functions?

I suspect the compiler would output the definition in the BMI, but I cannot be sure. In that case, a recompilation would be required but isn't much different from the current situation. Sadly, GCC didn't allow me to export an inline function because of a crash, so I couldn't test these cases.

### In A World Without BMIs

Compilers implemented modules using BMI files, but it could happen otherwise in the future. Someday a compiler could fork itself into some sort of compilation server, which it would simply hold into memory the declaration data of each module interface and never serialize the interface into a binary file. In that case, the compiler server has all the knowledge required to tell if something needs recompilation or not, even for inline and template functions.

## Cool Kids Export Thier Templates

Exported templates were such a failure... we didn't imagine we would talking about exporting templates again. But in its `c++-modules` branch, GCC supports exported templates in modular code. Here's another example:

```c++
/// size.cpp
module;
#include <cstdio>
export module hello.size;

void print(std::size_t size) {
    std::printf("The size of arg is %u\n", size);
}

export namespace hello {
    template<typename T>
    void print_size(T const& arg) {
        print(sizeof(arg));
    }
}

/// main.cpp
import hello.size;

int main() {
    int many_ints[] = {1, 2, 3, 4};
    hello::print_size(many_ints); // prints "The size of arg is 16"
}

```

Here we have a _template in a cpp file_ that is effectively exported. The `main.cpp` can see that declaration and instantiate it. The compiler knows how to instantiate the template because it serialized the template and its definition in the BMI.

> Hey BTW, you leaked into the global namespace!

No, I did not :-) This is a function with _module linkage_. Kind of static linkage mangled with a unique name. It is said that such entity with module linkage is _owned_ by the module.

Trying to use it directly in the `main.cpp` would result in a compiler error since the print function is not exported. This could not be possible if modules were precompiled headers in disguise: print would need to be visible since precompiled header does not differentiate between exported and non-exported entities.

## Tooling... In The Scope Of The Near Future?

Tooling is the bane of modules it this time of writing. All the stuff I presented above seem good, but it's all useless if we don't have a way to build our thousand file projects with them.

Since it changes how our code is compiled, of course, our tools need to change to adapt to this new model.

Our build system faces two main challenge:

 1. Discovering where the file is and which module they export
 2. Extracting the dependencies of a translation unit

I am **far from an expert in tooling and build systems**, but I'll try making sense of what I found and explore possibilities.

### File Vs Export Name

My example above was quite simple. A more complex project may have different naming conventions for file or classes and module names. I can easily imagine many projects having slightly different idea about how files should be named for a particular module interface. This leads to a problem: how does the build system link translation units to module names? The build system will need to compile some required interface on demand when compiling some other importing translation unit. [An article from cor3ntin](https://cor3ntin.github.io/posts/modules) explains that problem really well.

### Building The Graph

Getting what file imports which module is currently one of the great problems with modules. A build system will need to read cpp files to inspect which module import which ones. However, we don't want to turn our build system into compilers! The [build2](https://build2.org/) build system actually parses source files with a simple regex to extract every string of character following import. However, with the addition of legacy headers, getting the preprocessed file may be a bit more complicated since they allow exported macros. At that point, I think we should simply ask the compiler for the imports of a file. The compiler can go read imported headers since the import statement reveals the file name of those headers. That would simplify build systems but would require compiler support.

All of this still need file parsing before even the build starts. That can slow down first build considerably. However, [the author of build2 claims](https://build2.org/faq.xhtml#ninja) that the parsing overhead extracts valuable data that can speed up the build afterwards and also has other advantages. If legacy header uses are infrequent and only used for special cases like exporting symbols from legacy or C code, then the lack of almost any preprocessor directives can make the extracting process really fast.

Also, Nathan Sidwell [open up the possibility](https://youtu.be/4yOZ8Zp_Zfk?t=1216) to reuse it's build server idea to construct the dependency graph on the fly.

## What About CMake?

The CMake situation is a bit problematic. I think supporting modules in CMake will be really hard, but still possible. If CMake chooses to build the dependency graph before compiling, it has to do that in the target build system, ask the target build system to save the graph and module file association. With build system like GNU make, as far as I know it's possible too add custom build commands. When generating Visual Studio solutions, adding another meta-target to do that should not be a problem. Ninja may need to be patched to support that kind of commands from what I heard.

After extracting the module information, CMake has other choices to do: should it implement the build server approach described by the GCC design? Should it output the whole thing into neat little files so the compiler can read them? Something else maybe?

I once [asked the principal engineer at Kitware](https://www.reddit.com/r/cpp/comments/8sie4b/i_manage_the_release_cycle_for_cmake_the_build/e0zrad4), a leading CMake developer. There some pathways to add support for midules since CMake already supports Fortran's ones, but they really need to see how module actually work when they are finished. At that time the macro situation wasn't clear enough to draw any good decision, and he also concluded that build systems of today will need to be updated to support modules since they are not ready yet.

CMake could also add a build2 generator. It would basically add modules support for free and provide some other interesting feature and tools.

## What Should We Do With Tools

There are many interesting problems with interesting solutions. I think they all are really interesting problems and may be hard to solve. I don't think we should delay modules because tools are not supporting them. It's kind of a chicken and egg problem. Why should build system implement something as big as modules if they are not even standard yet? What if they change and their implementation doesn't work anymore? If we want our tools to support modules, I think we need them to be standardized.

## Good Luck Modules, And Nice Work Everyone

With all that said, the amount of effort and time the community and the cometee has put into the recent and next C++ revisions is incredible. Looking back to C++11, I would have never guessed to see that amount investement into the evolution of the language. I wish everyone a nice stay at San Diego if they actually find thier way through that mountain of papers. I'm looking forward those highly detailed trip report we have been so lucky to have and hope for the best!
