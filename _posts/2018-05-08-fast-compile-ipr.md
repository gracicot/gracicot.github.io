---
layout: default
title:  "IPR May make modules faster"
date:   2018-05-08 18:11:01 +0500
categories: general
excerpt_separator: <!--more-->
---

# IPR May make modules faster

Module. I just cannot wait to move all my code to them. I'm sick of headers.
Well, they won't disappear in an instant, but a whole bunch of them will when module will be official. I see modules as a
general, clean way to componentize my code and expose clear interfaces. Some see then primarly as a tool to make compilation faster,
and indeed it will most of the time, make your code compile faster... wait... *most* of the time? Can module really slow down compilation?

In this post we will see how module can slow down compilation in some case, and how I see IPR as a viable solution to fix it.

## From parallel To Serial

Compiling a large mode base is slow. Right now, each cpp file is preprocessed. Every file a compilation unit includes is textually
appended at the inclusion point. Do that for 800 compilation unit, and you'll compile 800 times the same code. What a waste of resource!

Even if it seems stupid, it has an advantage: each compilation unit is completely independent. I could compile each processed unit
one by one or... all at once! This is the advantage of the current model: one compilation unit is not waiting for another one to be compiled.

> What modules have to do with that?

Well, module don't have that same independence between compilation unit.
And this is by desing: You must compile interfaces in order to consume them. And that's okay, 
this is one of it's primary goal: not to parse the same source code again. So if a module interface is parsed one time, every
consumer of that module must wait for that interface to finish being parsed.

What's the effect in the long run, for a large project? Well, interfaces need to be compiled first. Good, but these interface
themselves consumes other interfaces, so they need to be be compiled too.

Also, this problem can easily be amplified with some coding style. In the interface of a module, you can put definition for
template functions, global data and even implement inline functions. Since the interface is compiled into an object file like any
other translation unit, it can compile the definitions, and consumers are importing, not including, so they know an implementation
is provided by the object file of the interface. This enable writing a complete codebase in term of module interfaces.

Personally I like this style, I don't like repeating function signatures and definitions. But this can really slow down compilation
because since implementations are in the interface, they must be compiled before getting out the compiled interface, and
there will inevitably be more interface dependency since all other modules we need to compiled the implementation will also be
a dependency of our interface.

This seem really bad. Adopting a particular style may make your compilation serial instead of parallel. There must be a way out
of that! Is there a solution to keep my favourite style and keep compilation fast?

## What's IPR?

Fist we'll discuss a bit about IPR, for those not aware on what it is. let's read what their readme says:

> The IPR, short for *Internal Program Representation*, is an open source project originally developed as the core data structures of a framework for semantics-based analysis and transformation of C++ programs. 

So IPR is a binary format that can be used to represent C++ in a concise and efficient way, such as declaration and definitions. It was developed by Gabriel Dos Reis and Bjarne Stroustrup. Lately, it was proposed to be a candidate for a common binary format for module interfaces that would work cross-compiler, and could be shipped alongside or embedded in binaries.

This seem great! Right now, every compilers have their own binary format that can change from version to versions. They are not shippable and they serve as a kind of cache for binary interfaces. If we don't have a common binary format, we will have to ship module interfaces just like we ship headers today.

As Gabriel Dos Reis said, IPR won't be forced in compilers to be the *one format to rule them all*. Instead, if IPR is adopted, compilers will simply transform IPR into their own, optimized format. Their own binary format won't be shippable, but if every compiler (or an external tool) can output IPR and every compilers can translate it, we basically have a common format for compiled module interface.

## IDEs and IPR

IDEs are great tools that help us write code in many ways. Take auto completion, highlighting, fixit, find use and other goodies the offers. All these features need to understand the code for then to work. It must understand auto and even sfinae to give accurate information about the code. To do that, they must parse the code and store a representation of the code in memory. Does that sound familiar to you? Indeed, it sound like a compiler that store information about the code, it's interface and implementation, and other helpful informations. 
