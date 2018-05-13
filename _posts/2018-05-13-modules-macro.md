

The debate on whether we should support macro in module has always been very polarized. This is a response to the recent reddit post
[Really think that the macro story in Modules is doing more harm than good.](https://www.reddit.com/r/cpp/comments/8j1edf/really_think_that_the_macro_story_in_modules_is/)
and the paper [p1052r0](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p1052r0.html).
I want to make a small post showing what is possible to do, and how we can solve the need for modular macros and find common ground in this story.

## Macros: A New Start

Modules is not the end of macro. I think modules as currently proposed by the TS (not mixing macro in them) has good interactions with macros for
most needs. I'll show you what I mean with examples of how we can use.

It makes explicit when macros are included with a module. Indeed, it even enforce a separation between preprocessor
code and C++ code. Importing a module with macro will look like this:

```c++
// Module and macros "logging.h"
import lib.logging;
#define MACRO_A ...
#define MACRO_B ...
```
```c++
// Importer "user.cpp"
module user;
import lib.graphic; // traditional import
#include "logging.h" // import with macros
```

In this example, C++ symbols and preprocessor directives are separated.
The C++ symbols come from a modules, but macro are included using the preprocessor.
Of course, one can also separate them completely, and forcing the user to explicitly both import and include,
which is not bad either.

> It breaks the goal of modules! You have to include!

I think that even in a modular world, `#include`'ing should not be banned. There is nothing wrong by using `#include` .
Just like when smart pointers has been added, we didn't ban raw pointers, we are still using them, there is still a need for it.
With modules, there is still a need for including: interacting with the preprocessor across multiple files. And there is nothing wrong with it.

> But... Now you have duplicated code across TU, you are including everywhere! Imagine of you have many many large headers for other parts of your code or even external libraries.

Ok. The small header above don't have a huge impact on compile time, the header only contained preprocessor directive to change
it's state by defining some macros.
For whole libraries, compile time will be negatively affected.
It will break the purpose of modules by making every modular translation unit including the same code over and over.
It may work, but would not by optimal.
A better for external libraries would be to do something like that:

```c++
// sdl.ixx
module;
#include "SDL2/SDL.h"
export module sdl;

extern "C" { // can I do this? No really, I'm curious
export namespace sdl {
    using SDL_Init = ::SDL_Init;
}
}
```

Then if you really need to also export macros, add a header that only define those macros.

> But this is repetitive and tedious! I don't want to do this, I use 10 libraries like that!

Good! This is actually a good point. Repetitive and tedious stuff like that can be done by a script.
Using the clang tooling API, you can make a tool can generate your module interface and your defining header for you.
This should be even easier of the header for which you create such module interface is a "modularizable header", as defined by
the clang fork. I mean, if it's easy to write a magic `import "legacy.h"` that work like a module,
it would be fairly easy to generate a module interface alongside a preprocessor state header.
I didn't tried making a tool with this particular API, but it I don't think it would be impossible to do.

## Config Macros

Config macros are a common pattern in some libraries. Consider this code:

```c++
#define LIBA_NO_EXCEPTIONS
#include "liba.h"
```

Familiar? But what happen if somewhere you forget to define it?
Yes, you get an ODR violation. It can be sometimes hard to debug such mistakes.
Sure, the best solution would be to add the compile definition for all your translation unit,
but some prefer to do it this way.

So... What's happening with modules? Of course, defining a macro before importing has no effect,
and adding the compile definition for all the TU either won't have effect, how could we do this?

When compiling the module interface, we can add a compile definition, but only for the specific module interface.
If you add the `LIBA_NO_EXCEPTIONS` definition for the `liba` module interface,
the BMI will only contain the functions without exceptions.

The great thing about this, is we now cable forget to define it before including.
You can only import! No trace of the option in the code, and no added definition for the entirety of the code base.
Modules enforce a more robust solution for config macros, and without spilling macros in the code.

## A Resonable Compromise

Do I believe macros support in module should never be added?
No. I'm not enthusiastic about supporting macros in modules, but I still believe a reasonable compromise can be made.

I don't want to transform the `import` directive into a preprocessor directive.
I don't think a C++ statement should influence in any way the state of the preprocessor.
But new preprocessor directives to cope with the problem can be added.

Imagine something like this:

```c++
//Module A
export module a;
export void f();
#define A_MACRO 46
#export A_MACRO // export preprocessor directive
```
```c++
// Module B
import a; // make `f` visible
#import a // defines A_MACRO
```

I think this is a reasonable compromise since the import C++ statement will not change the preprocessor state,
and yet macro coulis be exported, and imported by other modules.

Do I believe this is ideal? No. From my point of view, preprocessor and modules are two separated things that
should stay separated. But a lot of big player are pushing for this feature.

## Compiler Extensions

Another compromise would be to ship macro support in modules as an extension.

Clang has always been friendly for macros in modules. Their BMI is actually a precompiled header. If we want to gradually
use module without breaking anything, if you happen to use clang, why not develop an extension? I mean, clang is open source.
Their codebase has been friendly toward modular macros.
If a company cannot affort to make a transition to modules without breaking their 200 million codebase and are using clang,
the extension could be activated temporarely and would allow giant codebases to make a smooth transition. Later on, we could disable the extension when it's not needed anymore.
This is great because it won't impact The language, or the community as a whole, but still allow the smooth transition large codebase are requiring.

 #conclude "blogpost.md"
========================

Modules 
