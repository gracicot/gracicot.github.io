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

Fist we'll discuss a bit about IPR, for those not aware on what it is. 
