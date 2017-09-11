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
// Our interface
struct Task {
    virtual void execute() = 0;
    virtual ~Task() = default;
};

// A implementation
struct PrintTask : Task {
    void execute() override {
        std::cout << "PrintTask" << std::endl;
    }
};

// Some uses of tasks
std::vector<std::unique_ptr<Task>> tasks;

void push(std::unique_ptr<Task> task) {
    tasks.emplace_back(std::move(task));
}

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

 > Yeah, jsut implement that interface and voilà! All types works!
 
The thing is, not all types can implement your interface. For example, you have some library that has a class like that:

```c++
struct SomeRunner {
    void run() { /* ... */ }
};
```

You cannot change that class. It won't ever extend from your interface. You have to make an adapter to make it work in your code.

Also, there another kind of classes that cannot possibly extend the interface: lambdas.

Yeah, lambdas! Your code is not compatible with them! Imagine writing something like that:
```c++
// push function from the example above
push([] { std::cout << "print something!"; });
```

Sadly, that won't work, because the lambda is not dynamically allocated, and it don't extends the `Task` class.
