---
layout: default
title:  "Concept-Model Idiom Part One: A new look at polymorphism"
date:   2017-07-11 18:11:01 +0500
categories: conceptmodel
excerpt_separator: <!--more-->
---

# Concept-Model Idiom Part One: A new look at polymorphism

Ah, good old Oriented Object Programming. Have a problem? Just make an interface, and implement it! Simple as that heh? Well no. Polymorphism as done today with classic OOP is intrusive, forces polymorphic behavior even in place where it's not really need it, almost always imply heap allocation, needs to carry the vtable pointer and the list goes on. Fortunatly, other pattern and idiom exixt. Here's an approach that might change how you see polymophism, and the problem it tries to solve. Let me introduce the Concept-Model idiom, also called runtime concept, or virtual concept.

I'm not the inventor of this idiom. I'm sure everyone doing OOP ended up doing something that look like it in some way (hint: Adapter pattern). The first time I saw the idea in a concrete way was in the Sean Parent's talk [Inheritance Is The Base Class of Evil](https://channel9.msdn.com/Events/GoingNative/2013/Inheritance-Is-The-Base-Class-of-Evil). Thanks to him, I manage to use it and practice it a lot. I want to share what I've learned by playing around with this pattern.

<!--more-->

Let's start with a simple example:

```c++
// Our interface.
struct abstract_task {
    virtual void execute() = 0;
    virtual ~abstract_task() = default;
};

// An implementation of our interface.
struct print_task : abstract_task {
    void execute() override {
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
        task->execute();
    }
    
    tasks.clear();
}
```

I'd say taht the code above is quite common. You have an interface, one or more implementation, a type erased list and we execute the `execute` function polymorphically.

There are few problem with that. First, dynamic allocation is required here. There are no way around it. The intent of this code is not "we want dynamic allocation 100% of the time!" No, the intent is "We want a type erased list of somthing that is a Task." Yet, always using dynamic allocation and polymorphism via vtable happen to be the most common way to do it, and it happen also that it's the only way the language automate the implementation of polymorphism.

Secondly, it doesn't work with all classes. I heard a lot people responding

 > Yeah, just implement that interface and voilà! All types works!

The thing is, not all types can implement your interface. For example, you have some library that has a class like that:

```c++
struct some_library_task : library_task {
    void execute() { /* ... */ }
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

First, the function `push` takes a `std::unique_ptr`. Think about it. Imageine you have dozens of functions that takes a task that way. What if one day you need all those functions that take a `std::unique_ptr<abstract_task>` to take a raw pointer or a reference instead? Well, you have to change all of those.

We will do what you should have done from the start: taking a struct that contains the pointer instead:
```c++
struct task {
    task(std::unique_ptr<abstract_task> t) noexcept : wrapped{std::move(t)} {}

    std::unique_ptr<abstract_task> wrapped;
};

void push(Task t);
```
But now there is still some problem. Using a `task` is quite dull, imagine writing `some_task.wrapped->process()`! Let's change that:
```c++
struct task {
    task(std::unique_ptr<abstract_task> t) noexcept : wrapped{std::move(t)} {}
    
    void process() {
        wrapped->process();
    }
    
private:
    std::unqiue_ptr<abstract_task> wrapped;
};
```

Now that is already nice! Everywhere you had a `std::unique_ptr<abstract_task>`, you can secretly use a `Task` transparently, and using that class is more natural: dot instead of arrow for members, move semantics, you receive by value... all that good stuff!

And yet, the semantics didn't change for our caller:
```c++
push(std::make_unique<print_task>());
```
> But wait... we haven't fixed our problem yet! We want to support lambdas, change the way objects are sent, avoir heap allocation, is this really ganno help?

Of course! There is one thing in that list we can now do: change the way objects are sent. Instead of changing 200 function signature, we only have to change the constructor of `task`, and this is what we're gonna do.

Now, want the `push` function to albe be able to receive `some_library_task`. For that, we need adapters to adapt those types to the `AbstractTask` interface, and change the constructor of `Task`:
```c++
struct some_library_task_adapter : abstract_task {
    some_library_task_adapter(some_library_task t) : task{std::move(t)} {}

    void process() override {
        task.process();
    }
    
    some_library_task task;
};

struct task {
    Task(std::unique_ptr<abstract_task> t) noexcept : wrapped{std::move(t)} {}
    Task(some_library_task t) noexcept : task{std::make_unique<SomeLibraryTaskAdapter>(std::move(t))} {}
    
    void process() {
        wrapped->process();
    }
    
private:
    std::unqiue_ptr<abstract_task> wrapped;
};

int main() {
    push(some_library_task{});
}
```

Okay, now we're getting somewhere: We can use the push function by sending a library task by value!

However, our own task cannot be sent by value yet, we must send the pointer to it. So let's treat our own code as library code. All of our task class won't extend the abstract class anymore, just like the library code, and we will create an adapter for each of our classes. Also, we don't want any classes to extends `abstract_task`, so it will be a private member type:
```c++
struct task {
    task(some_library_task task) noexcept : task{std::make_unique<library_model_t>(std::move(t))} {}
    task(print_task task) noexcept : task{std::make_unique<print_model_t>(std::move(t))} {}
    task(some_other_task task) noexcept : task{std::make_unique<some_other_model_t>(std::move(t))} {}
    
    void process() {
        self->process();
    }
    
private:
    // This is our interface, now named concept_t instead of abstract_task
    struct concept_t {
        virtual ~concept_t() = default;
        void process() = 0;
    };
    
    // model instead of adapter here
    struct library_model_t {
        library_model_t(some_library_task s) noexcept : self{std::move(s)} {}
        void process() override { self.process(); }
        some_library_task self;
    };
    
    struct print_model_t {
        library_model_t(print_task s) noexcept : self{std::move(s)} {}
        void process() override { self.process(); }
        print_task self;
    };
    
    struct some_other_model_t {
        library_model_t(some_other_task s) noexcept : self{std::move(s)} {}
        void process() override { self.process(); }
        some_other_task self;
    };

    std::unique_ptr<concept_t> self;
};
```
> That's preposterous! You can't copy paste the same code for all of your (previously) subclass of `abstract_task`! There must be a way arount it!

Yes indeed, there's a great tool in C++ that was carefully made to avoid mindless copy paste like that: templates!
```c++
struct task {
    template<typename T>
    task(T task) noexcept : task{std::make_unique<model_t<T>>(std::move(t))} {}
    
    void process() {
        self->process();
    }
    
private:
    struct concept_t {
        virtual ~concept_t() = default;
        void process() = 0;
    };
    
    template<typename T>
    struct model_t {
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

Our class is a *universal adapter* for any classes that fits our need: having a `process` member function, in about **20 lines of code**.

## Perks

How does this idiom fixes all the problem problem I listed at the beginning? Let me show you with examples.

First, it enable polymorphism with a natural syntax. It looks uniform, and also has a lighter syntax.
```c++
void do_stuff() {
    std::string s = "value";
    task t = print_task{};
    
    // use string like this
    auto size = s.size();
    
    // use task like that
    t.process();
}
```
No arrow, no `new`, no `std::make_*`. Polymorphism is done transparently, without any additional overhead.

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
In this example, `t` is pushed into the list conditionally. If we don't need heap allocation and polymorphism, we can decide at runtime to not use it.

Third, our implementation of tasks can implement the `process` function the way it wants. so for example:
```c++
struct special_task {
    int process(bool more_stuff = false) const {
        // ...
    }
};
```
This still satifies the concept. We can still call `t.process()` even if the function is const, takes an optional parameter or has a different return type.

Another nice property of this idiom, is that **we treat our own code the same as library code**. This makes `task` truely generic. No matter where the object code from, it just work. That class doesn't need to know about the interface, or heap allocation, or even polymorphism. It just need to fit into the task concept.

## In closing

As we can see, the Concept-Model idiom, also called runtime-concept or virtual-concept is really powerful, and enables us a control over how polymorphism is done in our progerams.

> Hey, you forgot about lambdas!

We will see that in part 2, along other techniques for mapping our concept, and variations of the idiom.

I welcome any comments and criticism. If I can make this post better or less confusing, give me some feedback and I'll do my best to incorporate it in this post, or the next parts. If you have any questions, simply post in the reddit discussion, and I'll gladly answer!
