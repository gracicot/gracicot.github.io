---
layout: default
title:  "I made a package manager using CMake"
date:   2019-02-13 3:00:00 +0500
categories: cmake
excerpt_separator: <!--more-->
---

> What?? Another one?

_Likely the first reaction of most C++ programmers_

But I still made one. And I made it using CMake scripts. I'm currently using it for a hobby project a small team and I are working on, and I'm likely keeping it and maintaining it for a while.

Why is that? And why on earth using cmake as a scripting language? Today I'll explain my needs at the time, the thought process behind it and show you the result.

<!--more-->

## A Dependency Problem

At first, dependencies were not a problem. I used Linux and my mate too. We wanted something? We just had to pick it from the repo, write a small `FindXYZ.cmake` file if needed then use it.

We used libraries like `glfw`, `jsoncpp`, `glm` and a few others. But one day, I made a library to support our project. And the project itself was mostly a library, plus a few side projects and proof of concepts. Clearly, just with our code we had a multi-level dependency problem. Our workflow was to build them one by one in order. Luckily, most of them was header only, so it wasn't that bad, but required some knowledge of the setup to do it correctly.

Then I tried to use a library that wasn't package by my distribution. We simply added it to the list of packages to build beforehand... in the right order... Well, it was becoming quite tedious, something had to be done.

## install_missing.sh

My first solution I had to automate a repetitive task was of course to create a script. I wanted something simple: Something to install our missing dependencies.
