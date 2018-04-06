---
layout: default
title:  "Reflection in C++: The Present"
date:   2018-04-03 22:00:00 +0500
categories: reflection
excerpt_separator: <!--more-->
---

# Reflection in C++ Part 1: The Present

New popular languages often come with reflection baked in the language. All python, Java, Ruby and Javascript folk all shows up
with some fancy way to reflect on data structures.
What about C++? It's often criticized for its lack of reflection, but it doesn't mean it doesn't have any reflection capabilities.

In this post, we're gonna explore what reflection facilities are available to us today and what is possible to achieve given its
limitations. 

<!--more-->

## Reflection Broken Down

There is two main part that people refer when talking about reflection: reflection and reification. Here I want to bring to light multiple facets of the reflection part. For the lack of official terms, I'll refer to what I call type introspection and meta querying. Both are really useful in many applications. C++ has a great support for type introspection and has a basic support of meta querying. We'll see how the two compares.

 - **Type introspection** is the feature of reflection to ask the object something about something in particular. For example, you could ask an object if it has a `get_area()` member function in order to call it, or you could query the object to know if it has a `perimeter` data member convertible to int. What we're doing here is basically inspect the object to check if it fulfills a set of criteria. You can check for the validity of almost any expression in C++.

 - **Meta querying** is what I refer when I have an object and ask it to expose a set of its attributes, like querying the set of data members of a class. It's the kind of reflection we can get a list of all member functions, or a set off all the data members and thier declared names. This is what most people think about when talking about reflection in programming languages. That information is usually obtained through meta objects.

## Type Introspection in C++

C++ offers a quite powerful way to test any expression validity and let you inspect whether an object has a specific member or not. This is done with [SFINAE](http://en.cppreference.com/w/cpp/language/sfinae) today. Here's a basic example of this technique:

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

The expression `t.foo()` and `t.bar()` are part of the signature of the function, more precisely it's return type. While performing overload resolution, the compiler will ignore any function that their instantiation would cause an ill-formed expression.

So in the example above, we were able to check for the presence of `T::bar()` and `T::foo()` using SFINAE.

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

It's not an article about SFINAE, but I'll make a quick summary.

In this particular case, the compiler will try to find the more specialized version of `has_perimeter` for a given set of template arguments. If the expression `(<value of T>).perimeter` is invalid, the compiler cannot pick that specialization and will fallback to the default, which is equal to false. On the other hand, if the expression is valid, the specialization can be picked, so the value true is obtained.

This is a simple case of very basic reflection capability, but just this feature alone can yield impressive results, such as emulating concepts.

If you're interested in introspection capabilities of C++, please go check
Jean Guegant's blog post [An introduction to C++'s SFINAE concept: compile-time introspection of a class member](https://jguegant.github.io/blogs/tech/sfinae-introduction.html)

### Type Traits

We also cannot dismiss the type traits library provided by the STL. To some extents, it allows reflecting on types by implementing predefine capability or property to check. Some of these traits such as `std::is_final` or `std::is_polymorphic` cannot be implemented in pure C++, and needs compiler support.

## Meta Querying Objects in C++

Right now, C++ has very limited support for meta querying. For example, we cannot *yet* write `$Foo.members()`. There are some tricks that exists that we can use today, but not without serious limitations.

Using dark wizardry, some brilliant people managed to reflect members of class types to some extent using only C++14. The library implementing this is called [`magic_get`](https://github.com/apolukhin/magic_get). The limitation is that the reflected class must be an aggregate type. Sadly, many of reflection use case for class member also need member names, such as serialization. `magic_get` cannot fetch member names, only the count of members in the reflected class and their types.

The mechanism implementing this is too complicated for a single blog post. If you're interested in the implementation details of `magic_get`, I suggest you to watch the cppcon 2016 talk [C++14 Reflections Without Macros, Markup nor External Tooling..](https://www.youtube.com/watch?v=abdeAew3gmQ).

## Another Kind Of Reflection?

There is another particular construct in the language I want to bring our attention into. It exposes its interface in such a way that let you do all sorts of metaprogramming on it, and I want to focus on it today.

*Functions*. Yes, normal functions, member function, function objects and even function templates (to some extent). As long as it's not overloaded,
you can inspect its return type, it's parameter types, member of which class in case of member functions, and even let you
inspect how many template parameter it needs to take in some cases.

Why do I seem to act like this is a revolutionary thing? Well first, we can use functions to transfer behaviour and data around quite easily. Also, the whole interface of a function is the way you call it. Give me a function object or function pointer, I can tell you how can call it using meta data!

This can happen because function and function objects expose their whole interface in their type or `operator()` type, such as parameter types and return type. We'll see how we can use this property to make a small reflection library.

## Reflecting Function Types

Querying function to get a list of parameters can be done because all that data is encoded in its type. You probably did once reflected a function type by accident, but you know, many good things in C++ happened by accident. Here's an example of such code:

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

In this example, we are using some reflection capability of the compiler to get the return type of a free function, and default initializes the return value. Is it really reflection? In some way, yes. We can send a function to the `print_default_result` function, and it can infer the return type from the function we sent it. This can work because of template argument deduction. We will use this feature to make our small reflection library.

In the example above, we only extracted the return type of the function. In fact, our `print_default_result` function only work with function that takes no parameter. Let's fix that by adding a version that accept a function with a parameter:

```c++
template<typename R, typename Arg1>
auto print_default_result(R(*)(Arg1)) -> void {
    std::cout << R{} << '\n';
    // Do other stuff with Arg1, maybe.
}
```

This can be easily generalized using a variadic template:

```c++
template<typename R, typename... Args> // Accept a function with any number of parameter
auto print_default_result(R(*)(Args...)) -> void {
    std::cout << R{} << '\n';
    // Do other stuff with Args, maybe...
}
```

As we said, functions are almost the only basic entities that expose valuable properties in their type. Since what we want to reflect is part of their type, we can use the pattern matching abilities of template argument deduction to extract and expose the return and argument types.

Can we do something more useful than print a default constructed object when reflecting the return type of a function? Yes of course! Here's the first building block of our reflection library:

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

So what is happening here? Here instead of using template argument deduction for our pattern matching, we used partial specialization. At the line marked `#1`, we are defining the base case, where the type sent to us is not a function. We expose nothing in this case. That way we stay SFINAE friendly.

Then, at the line marked `#2` we expose an alias equal to the return type of the function type sent as template parameter.

Finally, at line `#3`, we are making an alias to the member type to make it easier to use.

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

Since we cannot make an alias to a parameter pack, we make an alias to a tuple type.

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

The cool thing here is it enables `decltype(auto)` like deduction without using return type deduction. Useful if you don't have C++14 enabled on your project.

## Lambdas

Of course, we also want to support lambda types. Inspecting a lambda type isn't much harder than inspecting a function pointer directly. A lambda is a compiler-generated type that has a member `operator()`. If we take a pointer to that member, we can send it to our `function_traits` struct so we can see its gut!

```c++
template<typename Lambda>
struct function_traits : function_traits<decltype(&Lambda::operator())> {}; /// #1

template<typename R, typename C, typename... Args>
struct function_traits<R(C::*)(Args...) const> { // #2
    using result = R;
    using parameters = std::tuple<Args...>;
};
```

We changed two things here. First, at the line marked `#1`, we changed our default case. We will assume (for the sake of simplicity) that when a type that is not a function pointer it's a lambda. It's a bit less SFINAE friendly but simpler to show here.

At line `#2`, we specialize our `function_traits` struct for a member function type. This will let us inspect the call operator of lambdas.

We can now use our updated utility for reflecting lambdas:

```c++
auto lambda = [](int) {};

using T = std::tuple_element_t<0, function_arguments_t<decltype(lambda)>>;

// T is int
```

## Reification

At this point, we are able to extract information about a function type and use it in a meaningful way. Yet our facility supports a powerful feature of reflection: reification.

What is that? Here's a quote from Wikipedia:

> Reification is making something real, bringing something into being, or making something concrete.

Like this quote is decribing, we will use reification to create something concrete out of high level meta data.

The idea is this: since we have meta-information about an object, we can make another object made from this meta information. We will re-use the return type and parameter type to create a new, different object type.

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
constexpr auto wrap_lambda(std::index_sequence<S...>, L lambda) {

    // A wrapper, local struct that recreate the lambda passed as parameter.
    struct Wrapper {
        constexpr Wrapper(L l) noexcept : lambda{std::move(l)} {}
        
        // Reify the call operator with the exact same parameter type and return type.
        constexpr auto operator() (nth_argument_t<S, L>... args) const -> function_result_t<L> {
            return lambda(std::forward<nth_argument_t<S, L>>(args)...);
        }
        
    private:
        L lambda;
    };
    
    return Wrapper{std::move(lambda)};
}

// Provide an overload without the sequence:
template<typename L>
constexpr auto wrap_lambda(L lambda) {
    return wrap_lambda(std::make_index_sequence<arguments_count<L>>(), std::move(lambda));
}
```

Here's a quick demo of it's usage:
```c++
int main() {
    constexpr auto wrapped = wrap_lambda([](int i) { return i * 2; });

    // Brace initialization works too, we are not using forwarding references and argument deduction.
    static_assert(wrapped({4}) == 8);
}
```
[Compile this code on godbolt](https://godbolt.org/g/iaZfMX)

Here in this example, we are creating a new callable type that privately contain the lambda. Yet, we expose a function that has the same parameters as the lambda function we receive. Since we don't use a variadic template to forward them into the lambda, we can still use brace initialization and we can still take a pointer to `operator()` and reflect on it again. We used reification to recreate the call operator and expose the *exact* same interface as our lambda.

Ne can go even further than that and implement something we could not see without reflection and reification. A function wrapper that contains the function to call and the parameters. The parameter will first be bound, then the call operator will be called without parameters. In other words, we will create a function object that allows binding parameter before calling the function.

Here's the implementation of that using reflection and reification:
```c++
template<typename L, std::size_t... S>
constexpr auto make_deferred(std::index_sequence<S...>, L lambda) {
    // We create a tuple type that can store the list of arguments of the lambda 
    // We are going to store that in our callable type to cache parameters before call
    using parameter_pack = std::tuple<nth_argument_t<S, L>...>;
        
    // We define our wrapper struct
    // We inherit privately to leverage empty base class optimization, but we could easily
    // have choose to contain a private member instead.
    struct Wrapper : private L {
        constexpr Wrapper(L l) : L{std::move(l)} {}
        
        // We make a bind function that take the same arguments as our lambda
        // we are going to store each argument in ...args for a later call
        void bind(nth_argument_t<S, L>... args) {
            bound.reset();
            bound = parameter_pack{std::forward<nth_argument_t<S, L>>(args)...};
        }
        
        explicit operator bool () const {
            return bound.has_value();
        }
        
        // We make a call operator that has the same return type as our lambda
        auto operator() () -> function_result_t<L> {
            // Here we are using the stored parameters set in the `bind` function
            return L::operator()(std::forward<nth_argument_t<S, L>>(std::get<S>(*bound))...);
        }
        
    private:
        std::optional<parameter_pack> bound;
    };
    
    return Wrapper{std::move(lambda)};
}

// Make an overload without the index sequence
template<typename L>
constexpr auto make_deferred(L lambda) {
    return make_deferred(std::make_index_sequence<arguments_count<L>>(), lambda);
}
```
Now that's something! We have now an object that supports calling and binding parameters in separate steps. Without reflection, dynamic allocation would have been required, since the list of types to forward to the lambda would only have been known at the moment we called `bind(...)`.

Instead, we simply reify a bind function that takes the exact same parameters as the lambda, and then store them in a reified tuple tailored to contain them.

Let's look at some usage of our `make_deferred` function:

```c++
int main() {
    // We make a deffered lambda
    auto func = make_deffered(
        [](int a, std::vector<int>& b, double c) {
            std::cout << a << '\n';

            for (auto&& i : b) {
                std::cout << ' ' << i;
            }

            std::cout << '\n' << c;
        }
    );
    
    auto vec = std::vector{1, 2, 3, 4, 5, 42};
    
    // Bind the parameters to our function
    func.bind(12, vec, 5.4);
    
    // Call the function with bound parameters:
    func();
}
```
[Run this example on Coliru](http://coliru.stacked-crooked.com/a/921de6cee166ed2a) (sorry godbolt, you can't run code yet)

That's the power of reflection + reification. You can generate new classes that changes their implementation according to our lambda.

Note that this implementation is minimal for the sake of simplicity and does not handle some caveats, such as the lifetime of references sent to `bind(...)`.

### Reflecting Other Functions

In all the example above we reflected the call  operator of lambda and function pointers. However, everything there is also applicable to other member function than `operator()`. Just replace `&T::opreator()` by `&T::funcName` and you can reflect specific member functions.

### Don't Reinvent The Wheel!

This looks all beautiful, but it can be quite hard to create and maintain it by yourself. This is why boost ship a complete implementation of function traits. I found it actually amazing how much information can be reflected in function types. If you're interested, check out [Boost.CallableTraits](https://github.com/boostorg/callable_traits)!

## Generic Lambdas

At this point, we simply reflect on normal function. Indeed, they are the easiest to reflect, and there are even libraries to do this. But... why stop there? There may be a useful use case where you'd want to reflect on generic lambda. Imagine you're in a situation where the user gives you a lambda, and a partial set of arguments. Let's say you have a function that gives you a value of any type called `magic_val<T>()`, and you have to call the lambda function using the partial set of parameter, and use `magic_val<T>` for all other non-provided parameters. To do that, you'll have to inspect the parameters of the lambda to know the type of the missing parameter from the user provided set.

Here's an example of usage:

```c++
// This is our goal:
magic_call(
// magic_val<T>() called for those two
//     v----  ----v
    [](SomeType1, SomeType2, int, double, auto&&, auto&&  ) {},
       /*magic*/  /*magic*/  4,   5.4,    "str1", "str2"sv
);
```

As we know, we cannot just take the address of a template function, we have to send it the template parameter first:

```c++
auto lambda = [](auto) {};

auto fctptr1 = &decltype(lambda)::operator(); // Error!
auto fctptr2 = &decltype(lambda)::operator()<int>; // works.
```

For the example of `magic_call` to work, we must deduce template parameters from a potentially different set. In the example of usage above, the user sends `int`, `double`, `char const(&)[5]`, `std::string_view`, but the template argument to deduce are `char const(&)[5]` and `std::string_view` only, so we must drop the `int` and the `double`.

So our utility `function_traits` won't work directly since it needs the type to extract the call operator directly. To support generic lambdas, we will introduce a new utility.

To deduce template arguments, we will use a simple type deduction algorithm. We will try to instantiate the template with all parameter type from the user provided set. If it results in a substitution failure (since we sent too many template arguments)  we will drop the first parameter and try again. In pseudocode, it will look like that:

    function deduced_function_traits(TFunc, ...ArgTypes)
        if TFunc instantiable with ArgTypes... then
            return function_traits(TFunc<ArgTypes...>)
        else if size of ArgTypes larger than 0
            return deduced_function_traits(TFunc, drop first ArgTypes...)
        else
            return nothing
            
To implement this in C++, we will need to know if a lambda is instantiable using a given set of template arguments.

We will use SFINAE to do this:

```c++
// Default case, the lambda cannot be instantiated, yields false
template<typename, typename, typename = void>
constexpr bool is_call_operator_instantiable = false;

// Specialization if the expression `&L::template operator()<Args...>` is valid, yields true
template<typename L, typename... Args>
constexpr bool is_call_operator_instantiable<
    L, std::tuple<Args...>,
    std::void_t<decltype(&L::template operator()<Args...>)> > = true;
```

Using this predicate we can then implement the algorithm just like in the pseudocode above. Each branch of the if in the pseudocode will become a partial specialization. Since there are three possible branch, we will need three specializations.
```c++
// Declaration of our function. Must be declared to provide specializations.
template<typename, typename, typename = void>
struct deduced_function_traits_helper;

// This specialization matches the first path of the pseudocode,
// more presicely the condition `if TFunc instantiable with ArgTypes... then`
template<typename TFunc, typename... ArgTypes>
struct deduced_function_traits_helper<TFunc, std::tuple<ArgTypes...>, // arguments TFunc and ArgTypes
    // if TFunc is instantiable with ArgTypes
    std::enable_if_t<is_call_operator_instantiable<TFunc, std::tuple<ArgTypes...>>>
> // return function_traits with function pointer
  // Returning is modelized as inheritance. We inherit (returning) the function trait with the deduced pointer to member.
     : function_traits<decltype(&TFunc::template operator()<ArgTypes...>)> {};

// This specialisation matches the second path of the pseudocode,
// the `else if size of ArgTypes larger than 0`
template<typename TFunc, typename First, typename... ArgTypes>
struct deduced_function_traits_helper<TFunc, std::tuple<First, ArgTypes...>, // arguments TFunc and First, ArgTypes...
    // if not instantiable
    std::enable_if_t<!is_call_operator_instantiable<TFunc, std::tuple<First, ArgTypes...>>>
> // return deduced_function_traits(TFunc, drop first ArgTypes...)
  // again, returning is modelized as inheritance. We are returning the next step of the algorithm (recursion)
     :  deduced_function_traits_helper<TFunc, std::tuple<ArgTypes...>> {};

// Third path of the algorithm.
// Else return nothing, end of algorithm
template<typename, typename, typename>
struct deduced_function_traits_helper {};
```

We can also define some alias to ease it's usage:
```c++
template<typename F, typename... Args>
using deduced_function_traits = deduced_function_traits_helper<F, std::tuple<Args...>>;

template<typename F, typename... Args>
using deduced_function_result_t = typename deduced_function_traits<F, Args...>::result;

template<typename F, typename... Args>
using deduced_function_arguments_t = typename deduced_function_traits<F, Args...>::parameters;

template<std::size_t N, typename F, typename... Args>
using deduced_nth_argument_t = std::tuple_element_t<N, deduced_function_arguments_t<F, Args...>>;

template<typename F, typename... Args>
constexpr auto deduced_arguments_count = std::tuple_size<deduced_function_arguments_t<F, Args...>>::value;
```

This implements the tool we need in order to reflect on generic lambdas. Instead of using `function_arguments_t` to reflect parameters off them, we will use `deduced_function_arguments_t`.

To implement magic call, we will call `magic_val<P>` for all the first parameters. The number of parameters to get through `magic_val` is the total number of parameter the function takes minus the number of provided arguments. To do this, we will use an index sequence:
```c++
//                                      the lambda type `L`  -----v
auto sequence = std::make_index_sequence< deduced_arguments_count<L, Args...> - sizeof...(Args) >();
//                                       The provided arguments  ----^
```

The Lambda of type `L` can be called with the set of provided arguments and the remaining parameters get through `magic_val`. Let `S` the the sequence generated above:

```c++
lambda(magic_val<deduced_nth_argument_t<S, L, Args...>>()..., std::forward<Args>(args)...);
```

Now to implement the `magic_call` function, we will simply need to call the expression above with the generated sequence. Here's how it would look like:

```c++
template<typename T>
constexpr T magic_val() {
    return T{}; // Simple implementation that default construct the type
}

//   We could have used decltype(auto) instead of reflection here ------v
template<typename L, typename... Args, std::size_t... S> //             v
constexpr auto magic_call(std::index_sequence<S...>, L lambda, Args&&... args) -> deduced_function_result_t<L, Args...> {
    // Call the lambda with both magic_val and provided parameter in ...args 
    return lambda(magic_val<deduced_nth_argument_t<S, L, Args...>>()..., std::forward<Args>(args)...);
}

template<typename L, typename... Args>
constexpr auto magic_call(L lambda, Args&&... args) -> deduced_function_result_t<L, Args...> {
    // We generate a sequence from 0 up to the number of parameter we need to get through `magic_val`
    auto sequence = std::make_index_sequence<deduced_arguments_count<L, Args...> - sizeof...(Args)>();
    return magic_call(sequence, std::move(lambda), std::forward<Args>(args)...);
}
```

And that will do the trick! We have the first call that generate an index sequence and call the function that invokes the lambda with both parameter sets. Now, for a use case like this one:
```c++
magic_call(
    [](SomeType1, SomeType2, int, double, auto&&, auto&&  ) {},
       /*magic*/  /*magic*/  4,   5.4,    "str1", "str2"sv
);
```
[Compile this code on godbolt](https://gcc.godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAKxAEZSAbAQwDtRkBSAJgCFufSAZ1QBXYskwgA5NwDMeFsgYisAag6yAwgREAHBpg3YOABgAipgIJyFSlZnVbUugnlQsmDI6YsnrXeUVlNQ1NADdMZCJib3MrHysAekSEv2TVAEkAW31MLMwWAiZXd1VUADNVcpFFEpZVAmImQkFUy2S2gjz9YsMtAgBPXQKmfNUAGVGAI3QmWN9LQUaRKKqaqLcWAH1G5oJBVRA12s2dppbQrCVB4YhuADZJrJmmEBBnTCboiABKH6N1AB2PiAiyyPhxPydbrMLqhG4jMYAJVIDSGiMwADpsapLMRgK1ZMZIYtlqtqid3Gc9oTNEiIAAqH4QPEE7GY/5EoEQhaqPmqESCBTAVTETCCEQMAiOMyqJEaHlWfkCoVsVS6JhNfJdYgHDSypboN46XKhVmCdnecFtDighU2kldHKwvradEeZGohEehyaL3u0ZYnHm%2BZWJbEFbSikbKm7C5aemaN5Mln4i3Yn6qNAsJYA22KvzKwXC0XiyXS/Vy%2B0kouqkUarWYHV62QGghGkAmgxmtOWonVha2sEF6yOmG9eEBsYAMVDfmLaujdS2YolUp2MrRwx9xxj2zj%2B1Cs6Jb1X5YH8THzon/SnDmPxIWC5FS9OmuAInyhUEG8r3sDu7LgetIPm8DaBs2F5QlePRwlohpvEKABemAbgAcv624AQ%2BbTPqohQIFs76fgUBC/q2qgIZ2egGFsmAGF%2BZEEKEGGAW%2B%2BIkd%2BOxHlaxjWiS0LXnBbpYTOc6WNmSyYAAHroxCqEwIhEApHGMT%2BaA1BWFFUV2qHIa6r5UsRancVoOEniAYSeCIfQjg6CwdCS6RmJg6ArK5bGlAeqh4M6eSkcUmxtI5g5OYkfIACoIA4ShMIIBxYG5EjoFshn7uc%2BxbNFDDDPJvm5GpDTRVUqAMAwqAAO4lp4wAkIQCBZMFKRhcqaWqIl7kpWl1ItBAEXTusqLsqyEXooI/wtcq/J4JU/XrD5OZFIUeBMFMBiqFVBAILi%2BKjcM6aYkVBRNVNp2ljoxD1N1wF9QNig9sAe3in22ATQ54VnfRggODNlF4ChZSVCNY2qMw%2BKfEVrCqCYJ1ncqYoXfUHXJal6xARlgi3YN7XEM4VR4Lq0rA/t7JvUkH2nV9hiTXDfII2I9QsKgW3CvZVhOrBrr/vkmEYrzO6VmEqB4Og4mSV0snyVMqClT56meAwWwfF8JBbAoSysK4q3rZW5SeN9UGjgsHMupOokOOM/OBuyO0EmL7hSZLqjS7LeDy2VSu5cUqvq0tWtra6luUe2xo0a65ovaiVFCyLpmaFcUroncXCPMa45dGUXvfJymgR9iRicnx2Cbsstls2k4UubFXybIDZRiJ5LCHQAsoK0pTA4Ceah5ylyagYQiw4gjDMgK0MP9gUO5igmc2bfNbvP3MOILwui/2JLhpG7WuZ1qOUulNJZfRuWGydUVu5RI9jxPdSqFkxTINFBxbQ45QE0s6rFNtFRHeq30qKgNAWBSAnSyCQBwclxR4AkAwAYv9sz4Fvhwe4JhfpzUUAtDWy1tYOE2ttYmz0cQvxYMgmGMFTa3nNqodByArb5BtgQwkj4wxkmlMjVye89w9UytlE%2BWgaFRxDtRU0WhGGR1UOkYypEDg0IUiwdAtsnqtBrPydIaC7rIDlpgv2K0A4bXqoosabRlRURGAHNW5Q45uy2MgBWntPje2IGrRamtdHdn4RowRHZdIPSUS9XiNouTpHppdRu3CDh4MbuqYWhRPjGIkeFJETYGYlgvmArA48UIKLigtaKxBCCsAkIdAA6j9FgeTCCqAgCElgwpMwvyid5SJDT2EKN0DEnUDRUB3zyB3Yg08VH8iONdDGlxIiJ1uA8GhachIOGVo434vixovULtyIcp8WrnwOMPSI19BCT3qPfAgj9xS/2%2Bggz%2BW064NN0P/dAgCMCYBAS1BppCqY%2BUqPpOujDQbvghltKGMMUEzwoSJDE1DPELx3NOd%2BBA6FBkOow8Sm9VitM4ejQ%2BvC4keOxjpMOR5YWojEfnIkqJJGqWkRC%2BarAFEwsJkS3ayz86DISR8/CzNtGuJwfEvkpiPDmJmnHMAYBrG2I9vM6IzisH%2B3cZoARwdvH4rMoSwxJMSVF3EgCYJyTQlopGTSLGihUToFxrofGhNVWEI5PEyRwBmgsFRDU1JBx0n0X%2Bh5HJCgKlLSKaoUpKkHBOrVA0lgMlpRSTNT/BpNU6pbSyFUsUyAxBCncGTQsyojjbyShw/VLQj45WxXKyFeKRG50ZWq6e/YuT5nWfxUK71qEIAJm0r%2B1zioxvyXGgZDbsAMG%2BudBm7KWZsFRAUBRP8O31UauQm8YKfTwvhci1hWbd65p4cfCGNa7R1svPW8mqgADKqAxieBWs/bpvCVQqh%2BsxASM7hJL1UNOeFDC0ziTwnqtGpxgKbk/fvcJ%2Ba%2BGaGfQq0Opa86Vo1Tu6Cxt05czvE%2Bl9wY33ryfHWFdKNupnnXFpWUj6/1cOAkeBlbJ1WnjLFKDZsHZlz2hUhxFKHmHznQwR5cUiuK4ahQBVj37RlmRIwdIwYFNQQU%2BISOy97XRUX0uheFiGuP0OQ3bVDVgP07xRgRIiFLCjkTbIq3IdEGKkTjqxHjRltOZWYvx22gmq18Qk9R2elDwUgaXq%2B5TTGJIOwlnJBSSlulovY5lDShRNwltovpMZ2aupfvMx%2BEyVngMCf8RZKyygy53r3RXE6zcmDAGgVmBWPk/KMQOU1culgTazsfRFe2i0ZK%2BYinfPL0Cthpd%2BGsllNTqFbuHKyg9%2BV1qDf8oUA5kM2GYD1uWLM3mIyrAaQiHltNlTpAAJqiEK/UHZo9T0AxZgcNqylE3JrwBEWBhWyrNfy5osVXgWXLb5OkAA8vJTAER6hBpFEwP6bADBEHqL7Qp1NBx2ky/uvk/qNIMAUQgJgERr0KITgiCAikiCZl9pgJg47KhinKAYPcqg8kOAALSk7J2EEF1WENBzc0pwQXjELup2DbA9WqKbLYpyScWDX5Ko%2B6ffa7NiFYQCogoLA0ktjfQAI42UUK6A94jxi/OeLMATDwHg23fONVQxOARme2NhpioQg4QbzMCJb6RNBFYacwFX33InSyuQL1raW5FtNxgPRKn9GydIUKodkWv1BdZ1fUW3LwIDO%2BQG1zwUXd6aaC3HA9qITe9nVUYX47IGcgHKCQCqmo15lo8yjtMPxSan1B1lyrcHaMARp3edzTC2jc6dnzq7rXbsQCV2H1XNn1cp01yXnXev1M5tiwbijRutAp9I5BzrvJVHhX9cAAoDiM7fel7LiQVRcbxpMAKM1ykQ2fj6XXcC2oIYVQcKGnu3Tl/Si2rjEQwBtqkMj9HrwwKWWt43wULflYqL3wADWqEYuMkkumAMuv%2BrogWFm6koghQxuyW6qOuf0KEFQ7IqYBIhciy0Glg8MIebeUeHeP%2BcuWeYCEQEA3eTAPwWeOexAeexABeIYRIxeWBZeuB6yYOfgCg0o98CgHWNaLKKK0oR6%2BQT0tAay26I4Jiy6ohmAT0XAkhw4S2S2eEPow8TAW%2BVE48Oo%2BsVGSo/Ib%2BHeS2yoHAAArHwGYWYBAHIeIaiLYeiFwKiDwcaqIAHKiHzn3vcB4f5l4XyJmL1s8vPrTIkAyJHgyB9KEeER9AACyoh8hmGYhxGmFcBcDhi0DcBOHqCpHhhcCZGCCc7BGqCch2QV5WBSA0EMDSBmFSCkAsDSAmC1GoDSCaD8D8CUSiDiDLwBC0C1EEANEVE0GAEgBmGyCYgACcsggIZhMRqR4x9wKC9wMRcRVRUgMRtR9RUgjRpAzRUgtRggIAJgpA/RWxFRpAcAsAMAiAKAx6ugeABgZAFAEAaAOQ9xnwKAzAbAKCRxb8UoYmlAUwAxpAUwCgmoAw0gvRpALxjET2LAsCQJWA98v2kgpxpA%2BAiargEQBxqJMkkQSkKJkJPB9EQJjQvkEJZxnx7AbRvAjAeAUwBxkANBzgdQ2JxOT2sgOuho%2BonAvA/AtAgIOu5QTMxOMkEgLgmwgg%2BxnREgdAlR1RGxQJux0kAAHPcMTksVmJSaoPcJiCYLqVUrgIQCQNkbIPQKoJoLcW8fJHILQJmK0bybwH0QMX8KQMMTETqSYPcFwDEWYYCICCYLILIJMWYd4asesaQFkCALIDqWYSYLQOMXGbQDEZ6cqcsfQJsdsbsfsYcccc6ecVcRAEgKIAQLoEpOQJQC8XcQ8XQKQK5EacQDWRVE0LoOSXKVIDUXUYqdIDafolciqWqRqbFGqB6bqU6acS6dFFju8b8IwNIOGZGfcGMTEeMcqSYCucqWYZMbQGYZuZ2aiVmUIDmScY0S6W6R6V6T6X6QGUGbICGbOVILIAqfudIGOSeW2VwE%2BZmS%2BbmeOTQRELqJsCADEUAA%3D%3D%3D)

The implementation will expand to this code (pseudo template expansion):
```c++
template<>
constexpr auto magic_call(
    std::index_sequence<0, 1>,
    L lambda,
    int&& arg0,
    double&& arg1,
    char const(&arg2)[5],
    std::string_view&& arg3
) -> void
{
    return lambda(
        magic_val<SomeType1>(),  // deduced_nth_argument_t<0, L, int, double, char const(&)[5], std::string_view>
        magic_val<SomeType2>(),  // deduced_nth_argument_t<1, L, int, double, char const(&)[5], std::string_view>
        std::forward<int>(arg0), // std::forward<Args>(args)...
        std::forward<double>(arg1),
        std::forward<char const(&)[5]>(arg2),
        std::forward<std::string_view>(arg3)
    );
}
```

Implement `magic_val` for a set of predefined types and we basically have implemented automatic dependency injection of function parameter, just like those fancy Javascript framework are so proud of! 

## Caveats of reflecting Generic Lambdas

As showed above, reflecting generic lambdas require us to deduce template arguments manually. The showned algorithm is far from perfect. It may produce incorrect result with variadic generic lambda, because any number of template parameter can be sent. Here's an example of that:

```c++
magic_call(
    [](int, auto&&... args) {},
       1,   2, 3
);
```

What should be part of `...args`? Our current algorithm will find that `operator()<int, int, int>` can be instantiated, so the first int will be obtained through `magic_val<int>()` instead of the first argument sent. This could in theory be fixed by trying to find the *minimum* amount of template parameter to send, but I haven't tried yet.

Other than that, the deduction will only produce the exact result if the template arguments are forwarding references. Consider this code:

```c++
auto vec = std::vector{1, 2, 3};

magic_call(
    [](auto) {},
    vec
);
```

What's happening here? The `operator()` is instantiated two times. Since we are using perfect forwarding and we send template parameter manually, we first instantiate the call operator like that in our algorithm: `operator()<std::vector<int>&>`. So even though `auto` parameter should not deduce references, we are instantiating it with a reference type.

Then, we call the function normally, instantiating the call operator with the correctly deduced arguments. This may produce an incorrect result or unwanted compilation slowdown. Using `auto&&` or sending prvalues + `auto`  will prevent the double instantiation.

We are also requiring all deduced parameters are at the end of the argument list. This may be a limitation in some cases.

# Conclusion

Reflection in C++ is yet to be added to the language, but that doesn't mean C++ don't have any reflection capabilities today. We just demonstrated that to some extent, C++ is capable of providing some reflection features that can solve some of today's problems.

What will be possible with tomorrow's reflection? What are the competing proposals and what are the pros and cons of each of those? We will see that in the next part of this blog post series about reflection in C++.

Let me know your opinion and comments in the [Reddit post](https://www.reddit.com/r/cpp/comments/8aalir/reflection_in_c_part_1_the_present/), take part of the discussion with me and share your thoughts on the subject!
