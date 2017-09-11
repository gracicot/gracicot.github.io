---
layout: default
title:  "Concept-Model Idiom Part One: A new look at polymorphism"
date:   2017-07-11 18:11:01 +0500
categories: conceptmodel
excerpt_separator: <!--more-->
---

# Concept-Model Idiom Part One: A new look at polymorphism

Ah, good old Oriented Object Programming. Have a problem? Just make an interface, and implement it! Simple as that heh? Well no. Polymorphism as done today with classic OOP is intrusive, forces polymorphic behavior even in place where it's not really need it, almost always imply heap allocation, needs to carry the vtable pointer and the list goes on. Fortunatly, other pattern and idiom exixt. Here's an approach that might change how you see polymophism, and the problem it tries to solve. Let me introduce the Concept-Model idiom, also called runtime concept, or virtual concept.

I'm not the inventor of this idiom. I'm sure everyone doing OOP ended up doing something that look like it in some way. The first time I saw the idea in a concrete way was in the Sean Parent's talk [Inheritance Is The Base Class of Evil](https://channel9.msdn.com/Events/GoingNative/2013/Inheritance-Is-The-Base-Class-of-Evil). Thanks to him, I manage to use it and practice it a lot. I want to share what I've learned by playing around with this pattern.

<!--more-->

Let's start with a simple example:

```c++
// Our interface
struct Task {
    virtual void execute() = 0;
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

