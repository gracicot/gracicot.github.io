---
layout: default
title:  "Modules And Macros: A Reasonable Compromise"
date:   2018-05-14 14:00:00 +0500
categories: modules
excerpt_separator: <!--more-->
---

# Modules And Macros: A Reasonable Compromise

The debate on whether we should support macro in modules has always been very polarized. This is a response to the recent Reddit post
[Really think that the macro story in Modules is doing more harm than good](https://www.reddit.com/r/cpp/comments/8j1edf/really_think_that_the_macro_story_in_modules_is/)
and the paper [p1052r0](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p1052r0.html).
I want to make a quick post showing what is possible to do, and how we can solve the need for modular macros and find common ground in this story.

 <!--more-->

## The Survival Of Macros 

Modules are not the end of macros. I think modules as currently proposed by the TS (not mixing macro in them) has good interactions with macros for most needs. I'll show you what I mean with examples of how we can use.

The current proposal makes it explicit when macros are included with a module. Indeed, it even enforce a separation between preprocessor
code and C++ code, and I think this is important to keep our interfaces clean. Importing a module with macro will look like this:

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
#include "logging.h" // imports lib.logging with macros A and B
```

In this example, C++ symbols and preprocessor directives are separated.
The C++ symbols come from a module, but macros are included using the preprocessor.
Of course, one can also separate them completely, and make the user to explicitly both import and include, a
which is not bad either.

> It breaks the goal of modules! You have to include!

I think that even in a modular world, `#include`'ing should not be banned. There is nothing wrong with using `#include`.
Just like when smart pointers have been added, we didn't ban raw pointers, we are still using them, there is still a need for it.
With modules, there is still a need for including: interacting with the preprocessor across multiple files. And there is nothing wrong with it.

> But... Now you have duplicated code across TU, you are including everywhere! Imagine if you have many many large headers for other parts of your code or even external libraries. Compilation will be slowed down.

This is a fine answer. The small header above don't have a huge impact on compile time, the header only contained the directives to change
the preprocessor state by defining some macros.
For whole libraries, compile time will be negatively affected... right?
Well, making every modular translation unit including the same code over and over kind of break the purpose of modules indeed.
It may work, but would not be optimal.
A better solution for external libraries would be to create a module interface for the library, take SDL for example:

```c++
// sdl.ixx
module;
#include "SDL2/SDL.h"
export module sdl;

export using SDL_Init = ::SDL_Init;
```

Then if you really need to also export macros, add a header that only defines those macros.

> But this is repetitive and tedious! I don't want to do this, I use 10 libraries like that! I won't maintain this!

Good! This is actually a good point. Fortunately, repetitive and tedious stuff like that can be done by a script.
Using the clang tooling API, you can make a tool can generate your module interface and your defining header for you.

This should be even easier if the library exposes a "modularizable header", as defined by
the clang folks. I mean, the Google proposal initially proposed the `import "legacy.h"` syntax that would transform a modularizable legacy header inclusion to an importation. If that is possible within clang, I cannot imagine why it won't be possible to generate a module interface and a separated header that applies the required preprocessor state.

I didn't try making a tool with this particular API, but I don't think it would be impossible to do. Invoking a generator like that can be done by the build system automatically, making the integration of legacy code and C code seamless.

This script won't magically make every code modular, but can greatly help the current situation and make the transition easier.

## Config Macros

Config macros are a common pattern in some libraries. We use it to enable or disable some features of a library by defining a macro before including it. Consider this code:

```c++
#define LIBA_NO_EXCEPTIONS
#include "liba.h"
```

Familiar? But what happens if somewhere you forget to define it at one place of many?
Yes, you get an ODR violation. It can be sometimes hard to debug such mistakes.
Sure, the best solution would be to add the compile definition for all your translation unit,
but some prefer to do it this way.

So... What's happening with modules? Of course, defining a macro before importing has no effect,
and adding the compile definition for all the TU either won't have an effect, how could we do this?

When compiling the module interface, we can add a compile definition, but only for the specific module interface.
If you add the `LIBA_NO_EXCEPTIONS` definition for the `liba` module interface,
the BMI will only contain the functions without exceptions.

The great thing about this, is we now cable forget to define it before including.
You can only import! No trace of the option in the code, and no added definition for the entirety of the code base.
Modules enforce a more robust solution for config macros, and without spilling macros in the code.

## A Reasonable Compromise

Do I believe macros support in modules should *never* be added?
My personal option is *No*. I'm not enthusiastic about supporting macros in modules, but I still believe that if something reasonable is proposed, we should consider it.

We could in the future add features that would allow exporting macros from modules.

I don't want to transform the `import` directive into a preprocessor directive.
I don't think a C++ statement should influence in any way the state of the preprocessor.
But new preprocessor directives to cope with the problem can be added.

I presume my concerns are valid and are shared by many developers, and I would be okay if we end up with something like this:

```c++
//Module libD
export module lib.d;
export void f();
#define D_MACRO 46
#export D_MACRO // export preprocessor directive
```
```c++
// Module B
import lib.d; // make `f` visible
#import lib.d // defines D_MACRO
```

I think this is a reasonable compromise since the import C++ statement will not change the preprocessor state,
and yet macro could be exported, and imported by other modules. I don't claim it to be the ultimate solution, but it shows that if we put efforts into it, maybe we could have something acceptable.

This is great because the feature exposed above is, I believe, implementable as a preprocessor only solution that won't directly change how C++ works, and the impact on build system should be reduced.

Do I believe this is ideal? No. From my point of view, preprocessor and modules are two separate things that
should stay separated. But a lot of big players are pushing for this feature, and there is no reason why we should exclude their needs.

## Compiler Extensions

Another compromise would be to ship macro support in modules as an extension.

Clang has always been friendly for macros in modules. Their BMI is more or less a precompiled header. If we want to gradually
use modules without breaking anything and if you happen to use clang, why not develop an extension? I mean, clang is open source.
Their codebase has been friendly toward modular macros.

If a company cannot afford to make a transition to modules without breaking their 200 million codebase and are using clang,
the extension could be activated temporarily and would allow giant codebases to make a smooth transition. Later on, we could disable the extension when it's not needed anymore.
This is great because it won't impact the language or the community as a whole, but still allow the smooth transition large codebases are requiring to be modernized.

 #conclude "blogpost.md"
========================

Modules are indeed a huge transition, and probably the biggest transition the C++ community has seen, bigger than C++11 in my opinion.
We can't simply drop all the old code that exists, but I think that if we provide the required tool and resources, the transition
can be done elegantly.

The perfect solution doesn't exist. This is why we are engineers, we work out solutions to difficult problems and find common ground between multiple constraints. When it affects the tool we usually use to solve problems, we tend to be polarized in our opinions. The above compromises and solutions are just *examples*, and I'm sure even better solutions exists if we take the time to find them. If we unite our work to move the community forward, find good enough solutions and stop dividing it I think we might find something acceptable for everybody, that would impact the C++ ecosystem positively.
