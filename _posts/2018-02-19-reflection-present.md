---
layout: default
title:  "Reflection in C++: The Present"
date:   2018-02-19 15:32:00 +0500
categories: reflection
excerpt_separator: <!--more-->
---

# Reflection in C++ Part 1: The Present

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

It's not an article about sfinae, since I consider this topic quite well covered, maybe another article. Anyways, in this case,  the compiler will try to find the more specialized version of `has_perimeter` for a given set of template arguments. If the expression `(<value of T>).perimeter` is invalid, the compiler cannot pick that specialization and will fallback to the default, which is equal to false. On the other hand, if the expression is valid,
the specialization can be picked, then yeild true.

This is a simple case of very basic reflection capability, but just this feature alone can yeild impressive results, such as emulating concepts.

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
using function_arguments_t = typename function_traits<F>::parameters;
```

Since we cannot make an alias to a argument pack, we make an alias to a tuple type.

## Lambda

Of course, we also want to support lambda types.

## Using function reflection

Using the facilities we make if fairly straightforward. Simply send a function type to an alias and use the types:

```c++
int some_function(std::string, double);

int main() {
    using F = decltype(&some_function);
    
    // The type of the first argument
    auto arg1 = std::tuple_element_t<0, function_arguments_t<F>>{};
    
    // Equivalent to decltype(auto) in that case
    function_result_t<F> result = some_function(arg1, 4.3);
}
```

The cool thing here is it enable `decltype(auto)` like deduction without using return type deduction. Useful if you don't have C++14 enabled on your project.

## Reification

At this point, we are able to extract information about a function type and use it in a meaningful way. Yet our facility supports a powerful feature of reflection: reification.

What is that? Reification is to make something real, concrete. Here's a quote from wikipedia:

> Reification is making something real, bringing something into being, or making something concrete.

The idea is this: since we have meta information about an object, we can make another object made from this meta information. We will re-use the return type and parameter type to create a new, different object type.

First, let's make some additional utilities about function related traits:

```c++
template<std::size_t N, typename F>
using nth_argument_t = std::tuple_element_t<N, function_arguments_t<F>>;

template<typename F>
constexpr auto arguments_count = std::tuple_size<function_arguments_t<F>>::value;
```

Then, here's a simple example of reification, which we recreate the call operator of a lambda:

```c++
template<typename L, std::size_t... S>
auto wrap_lambda(L lambda, std::index_sequence<S...>) {

    // A wrapper, local struct
    struct Wrapper : private L {
        using L::L;
        
        // Note: We could use `using L::operator()`, but we reify instead
        auto operator() (nth_argument_t<S, L>... args) const -> function_result_t<L> {
            return L::operator()(std::forward<nth_argument_t<S, L>>(args)...);
        }
    };
    
    retrurn Wrapper{std::move(lambda)};
}

// Provide an overload without the sequence:
template<typename L>
auto wrap_lambda(L lambda) {
    return wrap_lambda(lambda, std::make_index_sequence<arguments_count<F>>());
}
```

Here in this example we are creating a new callable type that extend privately the lambda. Yet, for the sake of this example, we expose a function that has the same parameters as the lambda function we recieve. We used reification to recreate the call operator, but we can go further and implement something we could not see without reflection and reification. 

```c++
template<typename L, std::size_t... S>
auto make_deferred(L lambda, std::index_sequence<S...>) {

    // We define our wrapper struct
    struct Wrapper : private L {
        // We create a tuple type that can store the list of arguments of the lambda 
        using parameter_pack = std::tuple<nth_argument_t<S, L>...>;
        using L::L;
        
        // We make a bind function that take the same arguments as our lambda
        void bind(nth_argument_t<S, L>... args) {
            bound.reset();
            bound = parameter_pack{std::forward<nth_argument_t<S, L>>(args)...};
        }
        
        explicit operator bool () const {
            return bound;
        }
        
        // We make a call operator that has the same return type as our lambda
        auto operator()() -> function_result_t<L> {
            return L::operator()(std::forward<nth_argument_t<S, L>>(std::get<S>(*bound))...);
        }
        
    private:
        std::optional<parameter_pack> bound;
    };
    
    retrurn Wrapper{std::move(lambda)};
}

// Make an overload without the index sequence
template<typename L>
auto make_deferred(L lambda) {
    return wrap_lambda(lambda, std::make_index_sequence<arguments_count<F>>());
}
```
Now that's something! We have now an object that supports deferring a call and bind parameter separately from the creation site of the callable. And that without heap allocation!

We simply reify a bind function that takes the exact parameters as the lambda, and then store them in a reified tuple.

Although note that this implementation is minimal and does not handle some ceveats, such as lifetime extension.

Let's look at some useage of our `make_deferred` function:

```c++
int main() {
    // We make a deffered lambda
    auto func = make_deffered([](int a, std::vector<int>& b, double c) {
        std::cout << a << '\n';
        
        for (auto&& i : b) {
            std::cout << ' ' << a;
        }
        
        std::cout << '\n' << c;
    });
    
    auto vec = std::vector{1, 2, 3, 4, 5, 42};
    
    // Bind parameters to our function
    func.bind(12, vec, 5.4);
    
    // call the function with bound parameters:
    func();
}
```

That's the power of reflection + reification.

## Generic Lambdas

At this point, we simply reflect on normal function. Indeed, they are the easiest to reflect, but why stop there? There may be useful use case where you'd want to reflect on generic lambda. Imagine you're in a situation where the user gives you a lambda, and a partial set of arguments. Let's say you have a function that gives you a value of any type called `get_val<T>()`, and you have to call the lambda function. To do that, you'll have to inspect the parameters of the lambda to know the type of the missing parameter from the user provided set.

Here's an example of usage:

```c++
// This is our goal:
magic_call(
// get_val<T> called for those two
//     v----------v-----/
    [](SomeType1, SomeType2, int, double, auto&&, auto&&  ) {},
       /*magic*/  /*magic*/  4,   5.4,    "str",  "strv"sv
);
```

As we know, we cannot just take the address of a template function, we have to send it the template parameter first:

```c++
auto lambda = [](auto) {};

auto fctptr1 = &decltype(lambda)::operator(); // Error!
auto fctptr1 = &decltype(lambda)::operator()<int>; // works.
```

For the example of `magic_call` to work, we must deduce template parameters from a potentially different set. In the example of usage above, the user send `int, double, char const(&)[4], std::string_view`, but the template argument to deduce are `const const(&)[4]` and `std::string_view` only, so we must drop the `int` and the `double`.

So our utility `function_traits` won't work directly, since it needs the type to extract the call operator directly. To support generic lambdas, we will introduce a new utility.

To deduce template arguments, we will use a simple algorithm. We will try to instanciate the template with all every parameter type the use send to us. If it result in a subtitution failure (since we sent too many template arguments) then we will drop the first parameter and try again. In pseudocode, it will look like that:

    function deduced_function_traits(tfunc, ...arg_types)
        if tfunc instantiable with arg_types... then
            return function_traits(tfunc<arg_types...>)
        else if size of arg_types larger than 0
            return deduced_function_traits(tfunc, drop first arg_types...)
        else
            return nothing

In C++ template syntax, a functioning algorithm would look like that: 

```c++
// Else return nothing, end of algorithm
template<typename, typename, typename = void>
struct deduced_function_traits_helper {};

template<typename F, typename... Args>
struct deduced_function_traits_helper<
    F, std::tuple<Args...>, // arguments tfunc and arg_types
    std::void_t<decltype(&F::template operator()<Args...>)> // if is instantiable
> // return function_traits with function pointer
    : function_traits<decltype(&F::template operator()<Args...>)> {};

template<typename F, typename First, typename... Rest>
struct deduced_function_traits_helper<
    F, std::tuple<Args...>, // arguments tfunc and arg_types
    void // if not instantiable
> // return deduced_function_traits dropping the first argument
    :  deduced_function_traits<F, std::tuple<Rest...>> {};

// Define some alias to ease it's usage:
template<typename F, typename... Args>
using deduced_function_traits = deduced_function_traits_helper<F, std::tuple<Args...>>;

template<typename F, typename... Args>
using deduced_function_result_t = typename deduced_function_traits<F>::result;

template<typename F, typename... Args>
using deduced_function_arguments_t = typename deduced_function_traits<F>::parameters;
```

