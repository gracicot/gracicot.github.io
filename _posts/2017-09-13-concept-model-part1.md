---
layout: default
title:  "Concept-Model Idiom Part One: A new look at polymorphism"
date:   2017-09-13 11:30:00 +0500
categories: conceptmodel
excerpt_separator: <!--more-->
---

# Concept-Model Idiom Part One: A new look at polymorphism

Ah, good old Oriented Object Programming. Have a problem? Just make an interface, and implement it! Simple as that heh? Well no. Polymorphism as done today with classic OOP is intrusive, forces polymorphic behavior even in place where it's not really needed, almost always imply heap allocation, needs to carry the vtable pointer and the list goes on. Fortunatly, other pattern and idiom exists. Here's an approach that might change how you see polymophism, and the problem it tries to solve. Let me introduce the Concept-Model idiom, also called runtime concept, or virtual concept.

I'm not the inventor of this idiom. I'm sure everyone doing OOP ended up doing something that look like it in some way (hint: Adapter pattern). The first time I saw the idea in a concrete way was in the Sean Parent's talk [Inheritance Is The Base Class of Evil](https://channel9.msdn.com/Events/GoingNative/2013/Inheritance-Is-The-Base-Class-of-Evil). Thanks to him, I manage to use it and practice it a lot. I want to share what I've learned by playing around with this pattern.

<!--more-->

Let's start with a simple example:

```c++
// Our interface.
struct abstract_task {
    virtual void process() = 0;
    virtual ~abstract_task() = default;
};

// An implementation of our interface.
struct print_task : abstract_task {
    void process() override {
        std::cout << "Print Task" << std::endl;
    }
};

// A type erased list of tasks.
std::vector<std::unique_ptr<abstract_task>> tasks;

// A function that push a new task in the list.
void push(std::unique_ptr<abstract_task> task) {
    tasks.emplace_back(std::move(task));
}

// execute all tasks and clear the list.
void run() {
    for(auto&& task : tasks) {
        task->process();
    }
    
    tasks.clear();
}
```

I'd say that the code above is quite common. You have an interface, one or more implementation, a type erased list and we execute the `process` function polymorphically.

There are few problem with that. First, dynamic allocation is required here. There are no way around it. The intent of this code is not *"we want dynamic allocation 100% of the time!"* No, the intent is *"We want a type erased list of somthing that is a task."* Yet, always using dynamic allocation and polymorphism via vtable happen to be the most common way to do it, and it happen also that it's the only way the language automate the implementation of polymorphism.

Secondly, it doesn't work with all classes. I heard a lot people saying:

 > Yeah, just implement that interface and voilà! All types works!

The thing is, not all types can implement your interface. For example, you have some library that has a class like that:

```c++
struct some_library_task : library_task {
    void process() { /* ... */ }
};
```

You cannot change that class. It won't ever extend from your interface. You have to make an adapter to make it work in your code.

Also, there another kind of classes that cannot possibly extend the interface: lambdas.

Yeah, lambdas! Your code is not compatible with them! Imagine writing something like that:
```c++
push([] { std::cout << "print something!"; });
```

Sadly, that won't work, because the lambda is not dynamically allocated, and it don't extends the `abstract_task` class.

The Concept-Model idiom aim to fix these problem, and to give us even more control over what's happening under the hood.

## Concept-Model: The adapter pattern on steroids

In this section, I'll explain the process of going from classical OOP to the Concept-Model idiom. I'll try to break this into many small steps to make understanding easier.

First, the function `push` takes a `std::unique_ptr`. Think about it. Imagine you have dozens of functions that takes a task that way. What if one day you need all those functions that take a `std::unique_ptr<abstract_task>` to take a raw pointer or a reference instead? Well, you have to change all of those.

We will do what you should have done from the start: taking a struct that contains the pointer instead:
```c++
struct task {
    task(std::unique_ptr<abstract_task> t) noexcept : wrapped{std::move(t)} {}

    std::unique_ptr<abstract_task> wrapped;
};

// A vector of task, which wrap a unique pointer.
std::vector<task> tasks;

// We take a task by value, since it's constructible from a unique pointer.
void push(task t) {
    tasks.emplace_back(std::move(t));
}
```
But now there is still some problem. Using a `task` is quite dull, imagine writing `some_task.wrapped->process()`! Let's change that:
```c++
struct task {
    task(std::unique_ptr<abstract_task> t) noexcept : wrapped{std::move(t)} {}
    
    void process() {
        wrapped->process();
    }
    
private:
    std::unique_ptr<abstract_task> wrapped;
};

void run() {
    for(auto&& task : tasks) {
        task.process();
    }
    
    tasks.clear();
}
```

Now that is already nice! Everywhere you had a `std::unique_ptr<abstract_task>`, you can secretly use a `task` transparently, and using that class is more natural. Dot instead of arrow for members, receive by value... all that good stuff!

And yet, the semantics didn't change for our caller:
```c++
push(std::make_unique<print_task>());
```
> But wait... we haven't fixed our problem yet! We want to support lambdas, change the way objects are sent, avoid heap allocation, is this really gonna help?

Of course! There is one thing in that list we can now do: change the way objects are sent. Instead of changing 200 function signature, we only have to change the constructor of `task`, and this is what we're gonna do.

Now, want the `push` function to be able to receive `some_library_task`. For that, we need adapters to adapt those library types to the `abstract_task` interface, and change the constructor of `task`:
```c++
// Our adapter. We contain a library task and implementing the abstract_task interface
struct some_library_task_adapter : abstract_task {
    some_library_task_adapter(some_library_task t) : task{std::move(t)} {}

    void process() override {
        task.process();
    }
    
    some_library_task task;
};

struct task {
    task(std::unique_ptr<abstract_task> t) noexcept : wrapped{std::move(t)} {}
    
    // We can now receive a library task by value.
    // We move it into a new instance of adapter.
    task(some_library_task t) noexcept :
        wrapped{std::make_unique<some_library_task_adapter>(std::move(t))} {}
    
    void process() {
        wrapped->process();
    }
    
private:
    std::unique_ptr<abstract_task> wrapped;
};

int main() {
    // push a new task to the vector
    push(some_library_task{});
}
```

Okay, now we're getting somewhere: We can use the push function by sending a library task by value!

However, our own task cannot be sent by value yet, we must send the pointer to it. So **let's treat our own code as library code**. All of our task class won't extend the abstract class anymore, just like the library code, and we will create an adapter for each of our classes. Also, we don't want any external classes to extends `abstract_task`, so it will be a private member type:
```c++
struct task {
    task(some_library_task task) noexcept :
        self{std::make_unique<library_model_t>(std::move(t))} {}
    task(print_task task) noexcept :
        self{std::make_unique<print_model_t>(std::move(t))} {}
    task(some_other_task task) noexcept :
        self{std::make_unique<some_other_model_t>(std::move(t))} {}
    
    void process() {
        self->process();
    }
    
private:
    // This is our interface, now named concept_t instead of abstract_task
    struct concept_t {
        virtual ~concept_t() = default;
        virtual void process() = 0;
    };
    
    // We name our struct `model` instead of `adapter`
    struct library_model_t : concept_t {
        library_model_t(some_library_task s) noexcept : self{std::move(s)} {}
        void process() override { self.process(); }
        some_library_task self;
    };
    
    struct print_model_t : concept_t {
        library_model_t(print_task s) noexcept : self{std::move(s)} {}
        void process() override { self.process(); }
        print_task self;
    };
    
    struct some_other_model_t : concept_t {
        library_model_t(some_other_task s) noexcept : self{std::move(s)} {}
        void process() override { self.process(); }
        some_other_task self;
    };
 
    // We quite know it's wrapped. Let's name it self
    std::unique_ptr<concept_t> self;
};
```
> That's preposterous! You can't copy paste the same code for all of your (previously) subclass of `abstract_task`! There must be a way arount it!

Yes indeed, there's a great tool in C++ that was carefully made to avoid mindless copy paste: templates!
```c++
struct task {
    template<typename T>
    task(T t) noexcept : self{std::make_unique<model_t<T>>(std::move(t))} {}
    
    void process() {
        self->process();
    }
    
private:
    struct concept_t {
        virtual ~concept_t() = default;
        virtual void process() = 0;
    };
    
    template<typename T>
    struct model_t : concept_t {
        model_t(T s) noexcept : self{std::move(s)} {}
        void process() override { self.process(); }
        T self;
    };

    std::unique_ptr<concept_t> self;
};

int main() {
    // natural syntax for object construction! Yay!
    push(some_library_task{});
    push(my_task{});
    push(print_task{});
}
```
Problem solved! No more copy paste, no more inheritance, no more pointers in the API of our code!

Can you see the pattern now? We now have a class `task` that that is constructible with *any* object that happen to possess a member function named `process` callable with no parameter.

Our class is a *universal adapter* for any classes that satifies our concept. All that in about **20 lines of code**.

## Perks

How does this idiom fixes all the problem problem I listed at the beginning? Let me show you with examples.

First, it enable polymorphism with a natural syntax. It looks uniform, and also has a lighter syntax.
```c++
void do_stuff() {
    // Initialize a std::string using a value in direct initialization 
    std::string s{"value"};
    
    // Pretty similar syntax eh?
    task t{print_task{}};
    
    // Or if you like AAA style
    auto s2 = std::string{"potato"};
    auto t2 = task{print_task{}};
    
    // use string like this
    auto size = s.size();
    
    // use task like that. Again, pretty similar 
    t.process();
}
```
No arrow, no `new`, no `std::make_*`. Simple values. All polymorphism is done without it creeping into our classes, and hidden in implementation detail.

Second, it *avoid heap allocation*. Yes indeed, even if we pass around our object inside a unique pointer internally.
```c++
void do_stuff() {
    some_task t;
    
    // do some stuff with task
    t.stuff();
    
    // maybe push the task
    if (condition()) {
        push(std::move(t));
    }
}
```
In this example, `t` is pushed into the list conditionally. If we don't need heap allocation and polymorphism, we can decide at runtime to not use it. There are also other strategies, like using SBO for avoiding dynamic allocation that I'll cover in other parts.

Third, our implementation of tasks can implement the `process` function the way it wants. so for example:
```c++
struct special_task {
    int process(bool more_stuff = false) const {
        // ...
    }
};
```
This still satifies the concept. We can still call `t.process()` even if the function is const, takes an optional parameter or has a different return type.

Another nice property of this idiom, is that **we treat our own code the same as library code**. This makes `task` truely generic. No matter where the object code from, it *just work*. That class doesn't need to know about the interface, or heap allocation, or even polymorphism. It just need to satify the task concept.

Also, there is now a single place where the interface is inherited. One. This in itself dramatically reduces the complexity of our code, because that single implementation is the only place where the polymorphism is done. Previously, polymorphism was scattered all over our codebase.

## Conclusion

I want to conclude part one there. Indeed, there is much more to cover about this idiom, but this article has aready a satifying size. 

> Hey, you forgot about lambdas! Wasn't that the whole point of this?

We will see that in part two, along other techniques for mapping our concept, and allow more types like lambdas. In later parts, I'll also cover variations of the idiom, and how you can use this idiom to refactor your code incrementally, and probably much more!

Also, our example still forced any `task` to have dynamic allocations for the type erased model. I'll also cover a strategy to overcome this and do polymorphism on the stack using Concept-Model.

If you want to play around with this code, I uploaded a live example on [compiler explorer](https://godbolt.org/g/rLn9gu) and [Coliru](http://coliru.stacked-crooked.com/a/371c7e9589d79872).

I welcome any comments and criticism. If I can make this post better or less confusing, give me some feedback and I'll do my best to incorporate it in this post, or the next parts. If you're interested in more content, or you have any questions, found an error, simply post in the [reddit discussion](https://www.reddit.com/r/cpp/comments/709ttn/conceptmodel_idiom_part_one_a_new_look_at/), and I'll gladly answer!
