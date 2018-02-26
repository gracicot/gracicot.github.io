---
layout: default
title:  "Reflection in C++: The Present"
date:   2018-02-19 15:32:00 +0500
categories: reflection
excerpt_separator: <!--more-->
---

# Reflection in C++: The Present

New popular languages often comes with reflection baked in the language. All python, Java, Ruby and Javascript folk all shows up
with fancy way to reflect on data structures.
What about C++? It's often criticized for it's lack of reflection, but it doesn't mean it don't have any reflection capabilities.

In this post we're gonna explore what reflection facilities are available to us today and what is possible to achieve given it's
limitations. 

<!--more-->

## Reflection Broken Down

There are two main part that people refer when talking about reflection: reflection and reification. Here I want to bring to light multiple facets of the reflection part: introspection and querying.

### Introspection

Introspection is the feature of reflection to ask the object something about something in particular.
For example, you could ask an object if it has a `get_area()` member function in order to call it,
or you could query the object to know if it has a `perimeter` data member convertible to int.

What we're doing here is basically inspect the object to check if it fulfill a set of crieria.

### Querying

This is what I refer when I have an object and ask it to expose a set of it's attributes. For example, having a set of data members
to iterate on and process. This is what most people think about when talking about reflection in programming languages.

## Introspection in C++

C++ offers a quite powerful way to test any expression validity and let you inspect whether an object has a specific member or not.
This is done with sfinae today. Here's a basic sfinae example:

```c++
template<typename T> // foo version
auto foo_or_bar(T const& t) -> decltype(t.foo()) {
    return t.foo();
}

template<typename T> // bar version
auto foo_or_bar(T const& t) -> decltype(t.bar()) {
    return t.bar();
}

int main() {
    struct has_foo { void foo() const {} } f;
    struct has_bar { void bar() const {} } b;
    struct has_both : has_foo, has_bar {} fb;
    
    foo_or_bar(f);  // calls foo version
    foo_or_bar(b);  // calls bar version
    foo_or_bar(fb); // error, ambiguous call
}
```

In simply a few lines you can create a predicate that let you check an expression, therefore the presence of a member:

```c++
// Our predicate, false by default
template<typename, typename = void>
constexpr bool has_perimeter = false;

// If the expression `(<value of T>).perimeter` is valid, then true
template<typename T>
constexpr bool has_perimeter<T, std::void_t<decltype(std::declval<T>().perimeter)>> = true;

struct Foo {
    int perimeter;
};

struct Bar {};

static_assert(has_perimeter<Foo>);
static_assert(!has_perimeter<Bar>);
```
> How does it work?

It's not an article about sfinae, maybe another time, but basically the compiler will try to find the more specialized version of
`has_perimeter` for a given set of template arguments. If the expression `(<value of T>).perimeter` is invalid, the compiler cannot
pick that specialization and will fallback to the default, which is equal to false. On the other hand, if the expression is valid,
the specialization can be picked, then yeild true.

This is a simple case of very basic reflection capability, but just this feature alone can yeild impressive results.

If you're interested in introspection capabilities of C++, please go check
Jean Guegant's blog post [An introduction to C++'s SFINAE concept: compile-time introspection of a class member](https://jguegant.github.io/blogs/tech/sfinae-introduction.html)

## Querying Objects in C++

Who dream of writing `$Foo.members()`? yeah, me too. But this isn't about the future but what we can do now. Does C++ let you
list the set of members or the set of other things? For listing members, unfortunately, no. I don't have a magical solution for you today.
You can't ask the compiler to instanciate some code for each data members, or for each member functions.
However, there is one construct in the language that exposes how it's composed
and let you do all sorts of metaprogramming on it.

Functions. Yes, normal functions, member function, function objects and even function templates. As long as it's not overloaded,
you can inspect it's return type, it's parameter types, it's member of which type in case of member functions, and even let you
inspect how many template parameter it need to take in some cases.

Why this is big you ask? Why do I seem to act like this is a revolutionnary thing? Simply because it's rarely seen
as a reflection capability. But it is, give me a function object or function pointer, I can meta-tell you how can call it!
This can happen because function and therefore function objects are the only entities (besides tuples) that exposes valuable information in their type, such as taken parameters and return type. We'll see how we can use this property to make a small reflection library.

## Reflecting Free Function

Querying function to get a list of parameters can be done because all that data is encoded in it's type. You probably did once reflected a function type by accident, but you know, many good things in C++ happened by accident. Here's an example of such code:

```c++
template<typename R>
auto print_default_result(R(*)()) -> void {
    std::cout << R{} << '\n';
}

auto ret_int() -> int;

auto main() -> int {
    print_default_result(ret_int);
}
```

In this example, we are using some reflection capability of the compiler to get the return type of a free function, and default initialize the return value. Is it really reflection? In some way, yes. We can send a function to the `print_default_result` function, and it can infer the return type from the function we sent it. This can work because of template argument deduction. We will use this feature to make our small reflection library.

In the example abouve, we only extracted the return type of the function. In fact, our `print_default_result` function only work with function that takes no parameter. Let's fix that by adding a version that accept a function with a parameter:

```c++
template<typename R, typename Arg1>
auto print_default_result(R(*)(Arg1)) -> void {
    std::cout << R{} << '\n';
    // Do other stuff with Arg1, maybe.
}
```

This can be easily generalized using a variadic template:

```c++
template<typename R, typename... Args>
auto print_default_result(R(*)(Args...)) -> void { // Accept a function with any number of parameter
    std::cout << R{} << '\n';
    // Do other stuff with Args, maybe...
}
```

As we said, functions are almost the only basic entities that exposes valuable properties in their type. Since what we want to reflect is part of their type, we can use the pattern matching habilities of template argument deduction to extract and expose the return and argument types.

Can we do something more useful than print a defaut constructed object when reflecting the return type of a function? Yes of course! Here's the first building block of our reflection library:

```c++
template<typename NotAFunction>
struct function_traits {}; /// #1

template<typename R>
struct function_traits<R(*)()> { /// #2
    using result = R;
};

template<typename F>
using function_result_t = typename function_traits<F>::result; /// #3
```

So what is happening here? Here instead of using template argument deduction for our pattern matching, we used partial specialization. At the line marked `#1`, we are defining the base case, where the type sent to us is not a function. We expose nothing in this case. That way we stay sfinae friendly.

Then, at line marked `#2` we expose an alias equal to the return type of the function type sent as template parameter.

Finally, at line `#3`, we are making an alias to the member type alias to make it easier to use.

Note that we are not handling parameter types yet. As with our function `print_default_result`, we simply add an argument pack so it will also deduce the argument types:

```c++
template<typename R, typename... Args>
struct function_traits<R(*)(Args...)> {
    using result = R;
    using parameters = std::tuple<Args...>;
};

template<typename F>
using function_arguments = function_traits<F>::parameters;
```

Since we cannot make an alias to a argument pack, we make an alias to a tuple type.

## Using function reflection


