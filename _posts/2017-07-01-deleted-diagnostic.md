---
layout: default
title:  "Compiler Tricks: SFINAE and nice messages"
date:   2017-07-01 18:11:01 +0500
categories: tricks
---

C++ templates is often blamed of horrible errors. Diagnostics can be painfully large for users of heavily templated libraries. And indeed, there can be pretty horrible errors only using the STL.

## Do you even error?

Yeah... do you?

No, I didn't meant you, I meant the compilers. Do they. I felt like necessary to compare some solutions I have found to provide nicer messages from the compiler when having errors in templates. 

First, we will start with a non-sfinae enabled, simple template function that yield a compilation error. Let's see.

Look at this code:

```c++
template<typename A, typename B>
auto add(A a, B b) {
    return a + b;
}

int main() {
    struct {} someVar;

    add(someVar, 7);

    return 0;
}
```

Let's listen to the scream of pain of multiple compilers at the sight of this ill-formed program.

#### GCC

Ah! The good old GCC. The great pillar of GNU, that lighthouse of freedom in this proprietary sea, that everlasting source of... ok, let's cut the crap. Here's the output:

    add.cpp: In instantiation of ‘auto add(A, B) [with A = main()::<anonymous struct>; B = int]’:
    add.cpp:9:19:   required from here
    add.cpp:3:14: error: no match for ‘operator+’ (operand types are ‘main()::<anonymous struct>’ and ‘int’)
     return a + b
            ~~^~~

#### Clang

    add.cpp:3:14: error: invalid operands to binary expression ('(anonymous struct at add.cpp:7:5)' and 'int')
        return a + b;
               ~ ^ ~
    add.cpp:9:5: note: in instantiation of function template specialization 
        'add<(anonymous struct at add.cpp:7:5), int>' requested here
    add(someVar, 7);
    ^
    1 error generated.


#### MSVC
Oh, MSVC. That compiler. It always had some quirks and bugs, but it has gotten a lot better recently. Let's see how it goes.

    add.cpp(3): error C2676: binary '+': 'main::<unnamed-type-someVar>' does not define this operator or a conversion to a type acceptable to the predefined operator
    add.cpp(9): note: see reference to function template instantiation 'auto add<main::,int>(A,B)' being compiled
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
struct can_add<A, B, void_t<decltype(std::declval<A>() + std::declval<B>())>> : std::true_type {};

template<typename A, typename B>
auto add(A a, B b) {
    static_assert(can_add<A, B>::value, "Cannot add! You must send types that can add together.");
    return a + b;
}
```

You must think that the error is completly clean with that beautiful error message?

Wrong!

    main.cpp: In instantiation of 'auto add(A, B) [with A = main()::; B = int]':
    main.cpp:21:19:   required from here
    main.cpp:14:5: error: static assertion failed: Cannot add! You must send types that can add together.
         static_assert(can_add<A, B>::value, "Cannot add! You must send types that can add together.");
         ^~~~~~~~~~~~~
    main.cpp:15:14: error: no match for 'operator+' (operand types are 'main()::' and 'int')
         return a + b;
                ~~^~~
                
It's indeed not that great, we still see the same error as before, but with a nice message. One could think a static_assert will stop the compilation, just as a `throw` will halt the function. But compilers usually don't stop at the first compilation error, so many other error can appear.

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
    main.cpp:9:19: error: no matching function for call to 'add(main()::&, int)'
         add(someVar, 7);
                       ^
    main.cpp:2:6: note: candidate: template decltype ((a + b)) add(A, B)
     auto add(A a, B b) -> decltype(a + b) {
          ^~~
    main.cpp:2:6: note:   template argument deduction/substitution failed:
    main.cpp: In substitution of 'template decltype ((a + b)) add(A, B) [with A = main()::; B = int]':
    main.cpp:9:19:   required from here
    main.cpp:2:34: error: no match for 'operator+' (operand types are 'main()::' and 'int')
     auto add(A a, B b) -> decltype(a + b) {
                                 ~~^~~
    
Whoa!? It is by itself a huge compilation error! How to fix this?

Fortunately, one can do *inverse-matching* to a deleted function instead. It is as easy to add as:

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
    static_assert(!std::is_same<T, T>::value, "Cannot add! You must send types that can add together.");
}
```

We got back our message! but there's another catch: we broke sfinae. Users cannot test if one can use `add` with given parameter. It will match to our poisoned function, and it's a valid function according to it's signature.

## Where are we going with this?
This is some examples of what programmers actually do to solve this problem. Most of the time, we will see code such as the very first example. If programmers are constrained in a C++11 only environment, they may do as the first sfinae example did.

## Best of both worlds
Well, I have found another solution that may allows you to add custom compilation error messages without breaking sfinae.

Remember the first example with static_assert, how the compiler tried to find further errors? Let's exploit this to allow static_assert within a deleted function!

Let's make a simple class that will throw the error:
```c++
struct NoAddError {
    template<typename T = void>
    NoAddError() {
        static_assert(!std::is_same<T, T>::value, "Cannot add! You must send types that can add together.");
    }
};
```

Simply instantiating a constructor will lead to the error message.

Now, in the deleted function we must find a way to "force" the compiler to go further and instantiate the constructor of this class.

However, we must not alter too much the signature of the function, because causing the static_assert by trying to instantiate the signature will lead to a hard error.

The solution lies in default values. Some compiler will try to find further error instantiating the deleted function by instantiating, if  needed, the templates needed by the default value expression.

Let's change our deleted function to this:

```c++
template<typename A, typename B, std::enable_if_t<!can_add<A, B>::value, int> = 0>
void add(A, B, NoAddError = {}) = delete;
//                        ^---- Notice the default argument that calls the default constructor!
```

What do we get in our compiler output?

    add.cpp:16:6: note: declared here
     void add(A, B, NoAddError = {}) = delete;
          ^~~
    add.cpp: In instantiation of 'NoAddError::NoAddError() [with T = void]':
    add.cpp:21:19:   required from here
    add.cpp:6:9: error: static assertion failed: Cannot add! You must send types that can add together.
             static_assert(!std::is_same<T, T>::value, "Cannot add! You must send types that can add together.");
             ^~~~~~~~~~~~~
    
Now we're talking! We have a deleted function, that when called directly (in evaluated context) fire a static_assert!

Here's a another snippet that shows that sfinae is not gone:

```c++
template<typename A, typename B>
auto tryAdd(A a, B b) -> void_t<decltype(add(a, b))> {
    add(a, b);
}

template<typename A, typename B>
auto tryAdd(A a, B b) -> std::enable_if_t<!can_add<A, B>::value> {
    std::cout << "Well, some runtime error." << std::endl;
}
```

### The catch

Well, the catch is, it only work with GCC. Painful truth, but it's not so bad: Other compilers still output the "error name". In other words, the compiler will still output the struct name `NoAddError`, which can still be useful if you don't use GCC.

### Variadics
Variadic functions are quite different to deal with. We must introduce our error parameter somewhere without breaking argument deduction.

Let's use another example here. We want to implement a `callMe` function that will call a function with some parameters:

```c++
template<typename F, typename... Args>
auto callMe(F function, Args&&... args) -> decltype(function(std::declval<Args>()...)) {
    return function(std::forward<Args>(args)...);
}
```

To support errors like before, we will need to change the error class, and change the first parameter of the deleted function to be that error class. The constructor of the error class will match the `F` template parameter, and will use `Args...` without breaking type deduction.

```c++
template<typename...>
using void_t = void;

template<typename, typename = void>
struct is_callable : std::false_type {};

template<typename Sig>
struct is_callable<Sig, void_t<std::result_of_t<Sig>>> : std::true_type {};

template<typename... Args>
struct NotCallableError {
    //                                     v---- Constructor exist only of not callable.
    template<typename F, std::enable_if_t<!is_callable<F(Args...)>::value>* = nullptr>
    NotCallableError(F) {
        static_assert(!std::is_same<F, F>::value, "The function cannot be called given parameters.");
    }
};
```

Now, just add the deleted function:

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
    main.cpp:27:9: error: static assertion failed: The function cannot be called given parameters.
             static_assert(!std::is_same<F, F>::value, "The function cannot be called given parameters.");
             ^~~~~~~~~~~~~

Here's the [code snippet](http://coliru.stacked-crooked.com/a/56e296157cfbe5f5) that resulted in this error.

The great thing about having the static assert in a separate class is that you can add new errors simply by adding a new constructor:

```c++
// let's assume the trait `is_function` exists.

template<typename... Args>
struct NotCallableError {
    template<typename F, std::enable_if_t<is_function<F>::value && !is_callable<F(Args...)>::value>* = nullptr>
    NotCallableError(F) {
        static_assert(!std::is_same<F, F>::value, "The function cannot be called given parameters.");
    }
    
    template<typename F, std::enable_if_t<!is_function<F>::value>* = nullptr>
    NotCallableError(F) {
        static_assert(!std::is_same<F, F>::value, "The first parameter must be a function.");
    }
};
```
