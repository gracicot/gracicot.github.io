---
layout: default
title:  "I made a package manager using CMake"
date:   2019-02-13 3:00:00 +0500
categories: cmake
excerpt_separator: <!--more-->
---

> What?? Another one?

_Likely the first reaction of most C++ programmers_

But I still made one. And I made it using CMake scripts. I'm currently using it for a hobby project a small team and I are working on, and I'm likely keeping it and maintaining it for a little while.

Why is that? And why on earth using cmake as a scripting language? Today I'll explain my needs at the time, the thought process behind it and show you the result.

<!--more-->

## A Dependency Problem

At first, dependencies were not a problem. I used Linux and my mate too. We wanted something? We just had to pick it from the repo, write a small `FindXYZ.cmake` file if needed then use it.

We used libraries like `glfw`, `jsoncpp`, `glm` and a few others. But one day, I made a library to support our project. And the project itself was mostly a library, plus a few side projects and proof of concepts. Clearly, just with our code we had a multi-level dependency problem. Our workflow was to build them one by one in order. Luckily, most of them was header only, so it wasn't that bad, but required some knowledge of the setup to do it correctly.

Then I tried to use a library that wasn't package by my distribution. We simply added it to the list of packages to build beforehand... in the right order... Well, it was becoming quite tedious, something had to be done.

## Submodules

The idea of using submodules to ship dependencies was considered at first. I could fetch the submodule of only the libraries I needed. However, it turns out git is not a dependency manager. Submodule was hellish to support, and updates would have been a pain.

We choose not to use git submodules as a package manager. It Didn't felt *right*.

## install_missing.sh

My first solution I had to automate a repetitive task was of course to create a script. I wanted something simple: Something to install our missing dependencies.

I did not wanted to be intrusive in our workflow, so I wanted to keep the ability to use system libraries. So I needed a way, in bash, to know whether a package was installed or not. The problem was we were using multiple linux distributions, and multiple versions of them.

Long story short, it became hell quickly. I did not wanted to depend on any specific distribution, or if it was a linux system or not. To solve this, I turned the problem around and ask myself *Why do I want to know what package is installed?* The response was to know if I can use them in my CMake scripts. So in the end, I did not wanted to know which package is installed, but what libraries are usable with CMake!

To check if I can use a particular package in CMake is quite straighforward:

 1. Generate a CMakeLists.txt that tries to use `find_package(xyz REQUIRED)`
 2. Optionally, check if every targets are available for use
 3. Read the output for errors

That way, I could detect any libraries in a cross platform manner. If a package failed to be found, then I just had to install it in the script!

The installation process was also simple:

 1. If the library is not found, clone the library's repo
 2. Run cmake and build it
 3. Finally, install it.

And that's it! Or... is it?

There were many problems with that approach.

First, it was a `sh` script. Not that it's wrong, but it would be wrong to say it's truely multiplatform. Also, all the dependency data was in the script, including the arguments to pass to cmake, git repository and even package name.

Then, there was a distribution problem. How to deal with multiple project? Add the script to the git repository? Then when updating dependencies or the script itself I need to repeat the update everywhere? And if I choose the gitsubmodule approach to distribute the script, how do I represent multiple projects with different dependencies?

## Something Better?

We had to deal with these issues and make a choice. We already wasted too much time on configuration!

We started by distributed the script by copy it in all of our projects and libraries, and modified the script for different dependency set. Of course, it became hard to track which project was on which version of the script, and errors were creeping in different versions of the script.

After experimenting, we decided to ship the script in his own submodule. We added the union of all our dependencies in it. It became much easier to maintain it.

Clearly, we needed something better. Something that would be reliable, easy to use and easier to add new libraries than modifying a script.

## A Rewrite

To expose the clean and simple interface I wanted, I first needed to decouple the list of libraries to install from the logic of installing them.
