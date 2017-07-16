---
layout: default
title:  "Compiler Tricks: SFINAE and nice messages"
date:   2017-07-01 18:11:01 +0500
categories: tricks
excerpt_separator: <!--more-->
---

# Compiler Tricks: SFINAE and nice messages

C++ templates is often blamed of horrible errors. Diagnostics can be painfully large for users of heavily templated libraries. And indeed, there can be pretty horrible errors only by using the STL.

Library writers often are confronted with a choice: being sfinae friendly, or output a nicely crafted compiler error with `static_assert`.

While experimenting ways to output errors in the cleanest way possible, I have found a trick to both enable sfinae while providing custom error messages that may depend on multiple conditions.

<!--more-->

## Do you even error?

Yeah... do you?

No, I didn't meant you, I meant the compilers. Do they. I felt like necessary to compare some solutions I have found to provide nicer messages from the compiler when having errors in templates. 

First, we will start with a non-sfinae enabled, simple template function that yield a compilation error:

```c++
template<typename A, typename B>
auto add(A a, B b) {
    return a + b;
}

int main() {
    struct {} someVar;

    add(someVar, 7);
}
```

Now let's listen to the screams of pain of multiple compilers at the sight of this ill-formed program.

#### GCC

Ah! The good old GCC. The great pillar of GNU, that lighthouse of freedom in this proprietary sea, that everlasting source of... ok, let's cut the crap. Here's the output:

    add.cpp: In instantiation of ‘auto add(A, B) [with A = main()::<anonymous struct>; B = int]’:
    add.cpp:9:19:   required from here
    add.cpp:3:14: error: no match for ‘operator+’ (operand types are ‘main()::<anonymous struct>’ and ‘int’)
     return a + b
            ~~^~~

#### Clang

I like to use clang sometime when GCC's template bracktrace are not enough. It's nice to see some complex metaprogramming errors from another perspective. It's doing quite similar to GCC in this particular case:

    add.cpp:3:14: error: invalid operands to binary expression ('(anonymous struct at add.cpp:7:5)' and 'int')
        return a + b;
               ~ ^ ~
    add.cpp:9:5: note: in instantiation of function template specialization 'add<(anonymous struct at add.cpp:7:5), int>' requested here
    add(someVar, 7);
    ^
    1 error generated.


#### MSVC
Oh, MSVC. That compiler. It always had some quirks and bugs, but it has gotten a lot better recently. Let's see how it goes.

    add.cpp(3): error C2676: binary '+': 'main::<unnamed-type-someVar>' does not define this operator or a conversion to a type acceptable to the predefined operator
    add.cpp(9): note: see reference to function template instantiation 'auto add<main::<unnamed-type-someVar>,int>(A,B)' being compiled
            with
            [
                A=main::<unnamed-type-someVar>,
                B=int
            ]

We can spot easily the difference in style compared to the two other competitors. However, it's not *that* bad, no big deal.

## static_assert

Static assertations are a very handful feature of C++11. It allowed us to make custom compilation errors, and allow library writers to provide useful diagnostics to help the user fix the code. The most important thing about it is that the message in the compilation error is written by the library writer. Here's the same example as above with a static_assert, with gcc:

```c++
#include <type_traits>

template<typename...>
using void_t = void;

template<typename, typename, typename = void>
struct can_add : std::false_type {};

template<typename A, typename B>
struct can_add<A, B,
    void_t<decltype(std::declval<A>() + std::declval<B>())>
> : std::true_type {};

template<typename A, typename B>
auto add(A a, B b) {
    static_assert(can_add<A, B>::value, "Cannot add! You must send types that can add together.");
    return a + b;
}
```

You must think that the error is completly clean with that beautiful error message?

Not quite.

    main.cpp: In instantiation of 'auto add(A, B) [with A = main()::<anonymous struct>, B = int]':
    main.cpp:21:19:   required from here
    main.cpp:14:5: error: static assertion failed: Cannot add! You must send types that can add together.
         static_assert(can_add<A, B>::value, "Cannot add! You must send types that can add together.");
         ^~~~~~~~~~~~~
    main.cpp:15:14: error: no match for 'operator+' (operand types are 'main()::<anonymous struct>' and 'int')
         return a + b;
                ~~^~~
                
It's indeed not that great, we still see the same error as before, but with a nice message added. One could think a static_assert will stop the compilation, just as throwing will halt the function. But compilers usually don't stop at the first compilation error. Many, **many** other error can still appear after the static assert.

## SFINAE
This feature of C++ of probably one of the best one that came to existence for meta-programming. It's the building block of any library that want to introspect and validate types. As you noticed, I used it in the example above.

sfinae can also be used to prevent nasty compilation errors. Let's change out example again!

```c++
template<typename A, typename B>
auto add(A a, B b) -> decltype(a + b) {
    return a + b;
}
```
This code is even simpler than the one with static_assert, and can prevent enormous, incomprehensible error output.

But there's a catch. Look a this error message:

    main.cpp: In function 'int main()':
    main.cpp:9:19: error: no matching function for call to 'add(main()::<anonymous struct>, int)'
         add(someVar, 7);
                       ^
    main.cpp:2:6: note: candidate: template decltype ((a + b)) add(A, B)
     auto add(A a, B b) -> decltype(a + b) {
          ^~~
    main.cpp:2:6: note:   template argument deduction/substitution failed:
    main.cpp: In substitution of 'template decltype ((a + b)) add(A, B) [with A = main()::<anonymous struct>, B = int]':
    main.cpp:9:19:   required from here
    main.cpp:2:34: error: no match for 'operator+' (operand types are 'main()::<anonymous struct>' and 'int')
     auto add(A a, B b) -> decltype(a + b) {
                                 ~~^~~
    
Whoa! The error is a bit longer than expected! It shows us a lot of information information, but that information won't be there if we use `enable_if` with a type trait.

Fortunately, one can do *inverse-matching* to a deleted function instead. This will allow us to control the quantity of compiler output. It is as easy to add as:

```c++
void add(...) = delete;
```

If the substitution failed for our add function, it will use this one, which is deleted. This yield to a really clean error:
   
    main.cpp: In function 'int main()':
    main.cpp:11:19: error: use of deleted function 'void add(...)'
         add(someVar, 7);
                       ^
    main.cpp:6:6: note: declared here
     void add(...) = delete;
          ^~~

However, we loose a valuable information: the reason the error happened. We can easily add our error back with a static_assert inside the fallback function:

```c++
template<typename T = void>
void add(...) {
    static_assert(!std::is_same<T, T>::value,
        "Cannot add! You must send types that can add together."
    );
}
```

We got back our message! but there's another catch: we broke sfinae. Users cannot test if one can use `add` with given parameter. It will match to our poisoned function, and it's a valid function according to it's signature.

## Where are we going with this?
This is some examples of what programmers actually do to solve this problem. Most of the time, we will see code such as the very first example. At the very best, we may see something like the first sfinae example or with the fallback function asserting.

## Best of both worlds
Well, I have found another solution that may allows you to add custom compilation error messages without breaking sfinae.

Remember the first example with static_assert, how the compiler tried to find further errors? Let's exploit this to allow static_assert within a deleted function!

Let's make a simple class that will throw the error:
```c++
struct NoAddError {
    template<typename T>
    NoAddError(T&&) {
        static_assert(!std::is_same<T, T>::value,
            "Cannot add! You must send types that can add together."
        );
    }
};
```

Simply instantiating a constructor will lead to the error message.

Now, in the deleted function we must find a way to "force" the compiler to go further and instantiate the constructor of this class.

However, we must not alter too much the signature of the function, because causing the static_assert by trying to instantiate the signature of the function will lead to a hard error.

The instanciation must only occur when you actually directly invoke the deleted function, not while instanciating the function signature.

The solution lies in the constructor of the error class. When invoking directly the deleted function, the compiler will try to find further errors and will instanciate the body of the constructor, effectively triggering the static assert.

Let's change our deleted function to this:

```c++
void add(NoAddError, ...) = delete;
//       ^--- This class has a template constructor.
//            The compiler must check it's signature
//            to know if this function is a match.
```

What do we get in our compiler output?
```
main.cpp: In function 'int main()':
main.cpp:44:15: error: use of deleted function 'void add(NoAddError, ...)'
     add("", "");
               ^
main.cpp:28:6: note: declared here
 void add(NoAddError, ...) = delete;
      ^~~
main.cpp: In instantiation of 'NoAddError::NoAddError(T&&) [with T = const char (&)[1]]':
main.cpp:44:15:   required from here
main.cpp:16:9: error: static assertion failed: Cannot add! You must send types that can add together.
         static_assert(!std::is_same<T, T>::value,
         ^~~~~~~~~~~~~
```
    
**Now we're talking!** We have a deleted function, that when called directly also fire a static_assert!

Here's a another snippet that shows that sfinae is not gone:

```c++
template<typename A, typename B>
auto tryAdd(A a, B b) -> void_t<decltype(add(a, b))> {
    std::cout << add(a, b) << std::endl;
}

template<typename A, typename B>
auto tryAdd(A a, B b) -> std::enable_if_t<!can_add<A, B>::value> {
    std::cout << "Well, some runtime error." << std::endl;
}

int main() {
    tryAdd(1, 2); // calls first
    tryAdd("some", "test"); // calls second
}
```
[See how it runs live at coliru](http://coliru.stacked-crooked.com/a/f4480cb1226c775c)

If there were no sfinae involved, the second call would be ambiguous.

Edit the file and try to call `add("", "")` directly, and you'll get the static assert.

### Default values

Another trick to trigger our static assert would be to use default values. Replace the constructor of `NoAddError` to take no parameter and add the parameter to the deleted function:
```c++
template<typename A, typename B, std::enable_if_t<!can_add<A, B>::value>* = nullptr>
void add(A, B, NoAddError = {}) = delete;
//                        ^---- Notice the default argument that calls the default constructor!
```

The same process goes: when invoked directly, the compiler will instanciate the constructor body, triggerring the static assert.


## Multiple errors

It can happen a code misues function at many places, many times. Fortunatly, the error class trick handle those cases really well. Here's the GCC output for wrongly call the add function four times:
```
main.cpp: In function 'int main()':
main.cpp:44:15: error: use of deleted function 'void add(NoAddError, ...)'
     add("", "");
               ^
main.cpp:28:6: note: declared here
 void add(NoAddError, ...) = delete;
      ^~~
main.cpp:45:15: error: use of deleted function 'void add(NoAddError, ...)'
     add("", "");
               ^
main.cpp:28:6: note: declared here
 void add(NoAddError, ...) = delete;
      ^~~
main.cpp:46:15: error: use of deleted function 'void add(NoAddError, ...)'
     add("", "");
               ^
main.cpp:28:6: note: declared here
 void add(NoAddError, ...) = delete;
      ^~~
main.cpp:47:15: error: use of deleted function 'void add(NoAddError, ...)'
     add("", "");
               ^
main.cpp:28:6: note: declared here
 void add(NoAddError, ...) = delete;
      ^~~
main.cpp: In instantiation of 'NoAddError::NoAddError(T&&) [with T = const char (&)[1]]':
main.cpp:44:15:   required from here
main.cpp:16:9: error: static assertion failed: Cannot add! You must send types that can add together.
         static_assert(!std::is_same<T, T>::value,
         ^~~~~~~~~~~~~
```

This is it! There is only one static assert, nicely placed at the end of the error list. The first error you see in you console is the most relevant. Neat!

## Variadics
Variadic functions are quite different to deal with. We must introduce our error parameter somewhere without breaking argument deduction.

Here we cannot rely on default values to make the trick.

Let's use another example here. We want to implement a `callMe` function that will call a function with some parameters:

```c++
template<typename F, typename... Args>
auto callMe(F function, Args&&... args) -> decltype(function(std::declval<Args>()...)) {
    return function(std::forward<Args>(args)...);
}
```

To support errors like before, we will need to change the error class, and change the first parameter of the deleted function to be that error class. The constructor of the error class will match the `F` template parameter, and will use `Args...` without breaking type deduction.

In order to check if the deleted function is a match, the compiler will need to check if the first argument can be sent to our `NotCallableError` constructor. This makes the compiler instanciating the constructor signature. Since the `static_assert` is inside the constructor body, the error must occur only if you try to effectively call that constructor. It's signature is completely valid, and will trigger sfinae if called with wrong parameters. Let's see how to implement that:

```c++
template<typename...>
using void_t = void;

template<typename, typename = void>
struct is_callable : std::false_type {};

template<typename Sig>
struct is_callable<Sig, void_t<std::result_of_t<Sig>>> : std::true_type {};

template<typename... Args>
struct NotCallableError {
    //                   v--- Constructor exist only if not callable.
    template<typename F, std::enable_if_t<!is_callable<F(Args...)>::value>* = nullptr>
    NotCallableError(F) {
        static_assert(!std::is_same<F, F>::value,
            "The function cannot be called given parameters."
        );
    }
};
```

Now, just add the deleted function and some utility to control when deduction happen:

```c++
template<typename T>
struct identity_t {
    using type = T;
};

template<typename T>
using identity = typename identity_t<T>::type;

template<typename... Args>
void callMe(NotCallableError<identity_t<Args>...>, Args&&...) = delete;
```

Now, calling the function with the wrong arguments will yield this error:

    main.cpp: In function 'int main()':
    main.cpp:45:34: error: use of deleted function 'void callMe(NotCallableError<identity_t<Args>...>, Args&& ...) [with Args = {const char (&)[5], main()::<lambda(int, bool)>&}]'
         callMe(lambda, "test", lambda)
                                      ^
    main.cpp:39:6: note: declared here
     void callMe(NotCallableError<identity_t<Args>...>, Args&&...) = delete;
          ^~~~~~
    main.cpp: In instantiation of 'NotCallableError<Args>::NotCallableError(F) [with F = main()::<lambda(int, bool)>; std::enable_if_t<(! is_callable<F(Args ...)>::value)>* <anonymous> = 0; Args = {const char (&)[5], main()::<lambda(int, bool)>&}]':
    main.cpp:45:34:   required from here
    main.cpp:27:9: error: static assertion failed: The function cannot be called with given parameters.
             static_assert(!std::is_same<F, F>::value,
             ^~~~~~~~~~~~~

Here's the [code snippet](http://coliru.stacked-crooked.com/a/e544fb49a4bd9efa) that resulted in this error.

It's not as clean as I would like to be, but it's still output our error properly, while not breaking sfinae.

The great thing about having the static assert in a separate class is that you can add new errors simply by adding a new constructor:

```c++
// let's assume the trait `is_function` exists.

template<typename... Args>
struct NotCallableError {
    template<typename F, std::enable_if_t<
        is_function<F>::value &&
        !is_callable<F(Args...)>::value>* = nullptr>
    NotCallableError(F) {
        static_assert(!std::is_same<F, F>::value,
            "The function cannot be called with given parameters."
        );
    }
    
    template<typename F, std::enable_if_t<!is_function<F>::value>* = nullptr>
    NotCallableError(F) {
        static_assert(!std::is_same<F, F>::value,
            "The first parameter must be a function."
        );
    }
};
```

I have used this pattern extensively in the library [Kangaru](https://github.com/gracicot/kangaru/blob/master/include/kangaru/detail/error.hpp), where a lot of error cases have been written in the same error class.

## The catch

Well, the catch is, it only work great with GCC. Painful truth, but it's not so bad. Users of your code using GCC will have full messages with a static assert, and others will get only a plain deleted function call error.

Even with that, other compilers will still output the struct name `NoAddError` since it's in the signature of the function, which can still be useful if you don't use GCC.

I have seen this working in some cases with clang, but is not as reliable as GCC for executing the trick.

If you already marked invalid overloads as deleted, using this trick won't break source.

## Dealing with other compilers

Writing good error messages should not only benefit users of GCC. The thing is, since all static assert are in a separated class, you can reuse them. Here's an example of that:
```c++
template<typename F, typename... Args,
    std::enable_if_t<std::is_constructible<NotCallableError<Args...>, F>::value>* = nullptr>
void debug_callMe(F&& function, Args&&... args) {
    NotCallableError error{std::forward<F>(function)};
    (void)error;
}

template<typename F, typename... Args,
    std::enable_if_t<!std::is_constructible<NotCallableError<Args...>, F>::value>* = nullptr>
void debug_callMe(F function, Args&&... args) {
    static_assert(!std::is_same<F, F>::value, "No error detected.")
}
```

Using the function `debug_callMe` as if it was `callMe` will trigger directly the static assert inside `NotCallableError` constructor. If no errors are detected, this code will output `No error detected.` as a static assert.

## Simple cases

Here's another bonus trick that can be used for simple cases. If you only have one possible error, just like our add function, you can always put a string literal in decltype. The compiler will also output that string when you invoke the function:
```c++
template<typename A, typename B, std::enable_if_t<!can_add<A, B>::value>* = nullptr>
auto add(A, B) -> decltype("Cannot add! You must send types that can add together."
) = delete;
```

This will yield this compiler output:
```
main.cpp: In function 'int main()':
main.cpp:47:15: error: use of deleted function 'const char (& add(A, B))[55] [with A = const char*; B = const char*; std::enable_if_t<(! can_add<A, B>::value)>* <anonymous> = 0]'
     add("", "");
               ^
main.cpp:29:6: note: declared here
 auto add(A, B) -> decltype("Cannot add! You must send types that can add together."
      ^~~
```

The output is quite clear, cleaner than error classes, and less verbose. In fact, it's the less verbose solution of all. Unfortunately, it's not as extensible as the error class trick. When you have a lot of error cases, grouping them all in one place is really useful and easier to maintain.

Also, all errors will output the message in the string literal, and they will be scattered around in the compiler output. With the error class trick, only one static assert is fired and is placed at the end of the compiler output.

For clang, this same simple trick will result in the message being in the middle of the error:
```
main.cpp:47:5: error: call to deleted function 'add'
    add("", "");
    ^~~
main.cpp:29:6: note: candidate function [with A = const char *, B = const char *, $2 = nullptr] has been explicitly deleted
auto add(A, B) -> decltype("Cannot add! You must send types that can add together."
     ^
main.cpp:24:6: note: candidate template ignored: substitution failure [with A = const char *, B = const char *]: invalid operands to binary expression ('const char *' and 'const char *')
auto add(A a, B b) -> decltype(a + b) {
     ^                           ~
1 error generated.
```

At least this time, the message is there, as the error classes don't quite work under clang.

Personally, I prefer the error class trick even for small cases. The error being a static_assert makes it clear for the various IDEs out there that parses the compiler output. That effectively makes it an error, as this simple case the message in inside a note.

---

That's it for today! I hope you enjoyed it!

If you find something wrong or have suggestions, compliments, complains, or even insults, just leave a comment on the [reddit post](https://www.reddit.com/r/cpp/comments/6kttou/compiler_tricks_sfinae_and_nice_messages/) or send me a github issue, I'll greatly appreciate any feedback! 
