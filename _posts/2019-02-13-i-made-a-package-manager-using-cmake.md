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

Why is that? And why on earth using CMake as a scripting language? Today I'll explain my needs at the time, the thought process behind it and show you the result.

<!--more-->

## Preface

I would like to say that I'm not an expert in build systems. I may have done some obvious errors or oversight. I just had a need that I solved using to tools I knew.

Also, simplicity both in setup and maintenance. I was working with a team that had very little experience with C++, and didn't know how to manage dependencies. We also had multiple platforms and different setup.

At the time, supporting how we setup the developpement was important. I was developping a framework and an app that used it. The rest of the team was only developping only the app and for them the framework was a dependency like any other. Simply asking to the package manager to take update everything should be one command and should update the framework it's latest revision.

For me the app should use the framework I had compiled in another directory. Not installing the framework on every changes was  important for me, and simply pushing on the master branch should be all I had to do to make the update available to the rest of the team.

Both vcpkg and conan didn't seem to propose all these properties, or didn't seemed simple enough for the whole team to use. The situation might have changed since the last time I tried this setup with these tools.

## A Dependency Problem

At first, dependencies were not a problem. I used Linux and my mate too. We wanted something? We just had to pick it from the repo, write a small `FindXYZ.cmake` file if needed then use it.

We used libraries like `glfw`, `jsoncpp`, `glm` and a few others. But one day, I made a small framework to support our project. So we had the main project that depended on some libraries, and our framework that also had dependencies. Clearly, just with our code we had a multi-level dependency problem. Our workflow was to build them one by one in order. Luckily, most of them was header only, so it wasn't that bad, but required some knowledge of the setup to do it correctly.

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
 2. Run CMake and build it
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

I decided to store all the package data into a json file, since it's easy to read from a script and to write by hand.

As for the language to be written in, I needed somthing that could run and both linux and windows, had the less steps to setup and provide me with the basic tools to build stuff.

Luckily, I know a language that respond to all these criteria: CMake. It's shipped with Visual Studio, so there's no additionnal steps to install it. It runs on all the platforms I need, and has all the tools I need to build and interact with the system.

So... A script to install CMake dependencies in CMake it is!

And... that will simply be taking the bash script, port it to cmake and read the json file to fill the data I previously hardcoded?

Haha, ha... ha... So naive.

It's true, I picked the same strategy as my previous script: Generating a `CMakeLists.txt` file, then run CMake to see if everything can be included.

However, I migrated from a 70 line script to a... 900 line CMake monster.

## 1. Reading The JSON

> WARNING: I do not recommend anyone to parse json in CMake. Do it if you like suffering like me.

Yes, this is what I wanted to do. Luckily, I found this wonderful git repository: [`sbellus/json-cmake`](https://github.com/sbellus/json-cmake). I had a base to work with and improve for my needs.

But cmake had no arrays or objects. Only, plain variable. How can one access and traverse the json structure?

The answer is obviously dynamic variable names!

Let's take a look at the syntax:

```cmake
set(foods "0;1;2")
set(foods_0 "Potato")
set(foods_1 "Tomato")
set(foods_2 "Pistachio")

foreach(food_idx ${foods})
    message("${foods_${food_idx}}")
endforeach()
```
>     Potato
>     Tomato
>     Pistachio

What's happening there?

First we setup our variable names. Examine the pattern here closely. Each variable has the same name followed by `_N` were `N` is the index in the array.

Then, inside the foreach, we use this syntax:

```cmake
${foods_${food_idx}}
```

It turns out that we can expand variable names inside a variable expansion. The expansion process look like this:

```cmake
${foods_${food_idx}} --> ${foods_1} --> Tomato
```

Another neat property of CMake variables is that their names can contain dots. It's a no brainer to use those to encode the json structure into variable names:

So this json:

```js
{
    "dependencies": [
        {
            "name": "nlohmann_json",
            "target": "nlohmann_json::nlohmann_json",
            "repository": "https://github.com/nlohmann/json.git",
            "tag": "v3.6.1",
            "options": "-DJSON_BuildTests=Off"
        },
        {
            "name": "stb"
            /* ... */
        }
    ]
}
```
Become these variables:
```cmake
set(manifestfile.dependencies              "0;1")
set(manifestfile.dependencies_0            "name;target;repository;tag;options")
set(manifestfile.dependencies_0.name       "nlohmann_json")
set(manifestfile.dependencies_0.target     "nlohmann_json::nlohmann_json")
set(manifestfile.dependencies_0.repository "https://github.com/nlohmann/json.git")
set(manifestfile.dependencies_0.tag        "v3.6.1")
set(manifestfile.dependencies_0.options    "-DJSON_BuildTests=Off")
set(manifestfile.dependencies_1            "name")
set(manifestfile.dependencies_1.name       "stb")
# ...
```
Then traversed like that:
```cmake
foreach(dependency-id ${manifestfile.dependencies})
    message("Dependency #${dependency-id}")
    set(dependency manifestfile.dependencies_${dependency-id}) # <-- (1)
    message("The name of the dependency: ${${dependency}.name}") # <-- (2)
    foreach(dependency-member ${${dependency}})
        message("${dependency-member}: ${${dependency}.${dependency-member}}") # <-- (3)
    endforeach()
endforeach()
```
Here at `(1)` we create a variable with a special value. It contain the prefix of every variable names that precedes the member of our dependency object. Also, dereferencing the variable name like this: `${${dependency}}` will yield the value `name;target;repository;tag;options`. A list of every members.

Then at `(2)`, we dereference our variable name, but followed by a postfix that is equal to a dot followed by a member name.

At `(3)` we use a variable name to member of our json object to display all of them in a loop.

Notice my choice of words here: We dereference the variable name. The name of the variable become some sort of a pointer. When the name of the variable is auto generated, then a variable containing that name is our only way to reference to that variable.

If we continue with this analogy, `dependency-member` is some sort of a pointer to member of our object.

The CMake JSON library has been tweaked a bit to generate variables with this structure. 

## 2. Looking For Existing Libraries

Just like my `install_missing.sh` script did, I don't want to be redundant and install libraries that already are available in the current system. I want to download and install them just if they are missing.

To check if a package is available in CMake, one can usually just use `find_package`, but sadly, CMake scripts cannot define targets, and config file are not meant to be ran in script mode. Another technical reason is that the find module or the library's CMake config file might trigger errors in our package manager script. Running any CMake script in the same process as this one is undesirable.

So I went with the same way as my shell script: generate a `CMakeLists.txt` file and try to run CMake on it to get a result.

The file I'm generating look similar to this:

```cmake
file(WRITE "./${${dependency}.name}/CMakeLists.txt" "
cmake_minimum_required(VERSION ${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION})
find_package(${${dependency}.name} VERSION ${${dependency}.version} REQUIRED)
if(NOT TARGET ${${dependency}.target})
    message(SEND_ERROR \"Package ${${dependency}.name} not found\"
)
endif()
")

execute_process(
    COMMAND ${CMAKE_COMMAND} . -DCMAKE_PREFIX_PATH=${project-prefix-paths}
    WORKING_DIRECTORY "./.${${dependency}.name}-test"
    RESULT_VARIABLE result-check-dep
    OUTPUT_QUIET
    ERROR_QUIET
)
	
if (${result-check-dep} EQUAL 0)
    set(check-dependency-${${dependency}.name}-result ON PARENT_SCOPE)
else()
    set(check-dependency-${${dependency}.name}-result OFF PARENT_SCOPE)
endif()
```

In short, this small piece of CMake script generate a minimal CMake script that check if a package is found and if a target is exported by the package.

If we set the exact same prefix path as we use in our project, we can be sure the same packages will be found.

## 3. Downloading And Installing Packages

I first went with `ExternalProject`. This is an awesome tool to download, configure and install libaries, I just have to call `ExternalProject_add` then...

>     CMake Error at /usr/share/cmake-3.14/Modules/ExternalProject.cmake:1016 (define_property):
>       define_property command is not scriptable

_Again! Why?_

Sadly, external project cannot be used either in script mode.

The solution was to run git manually to download the dependency, then build it.

> For the sake of simplicity, we will assume that the recipe is simply to run CMake to build and install the package without any additional steps. We will get back on this later.

```cmake
execute_process(
    COMMAND ${CMAKE_COMMAND}  .. -DCMAKE_BUILD_TYPE=${build-type} -DCMAKE_INSTALL_PREFIX=${modules_path}
    WORKING_DIRECTORY "./.${${dependency}.name}/${build-directory-name}"
)

execute_process(
    COMMAND ${CMAKE_COMMAND} --build . --target install
    WORKING_DIRECTORY "./.${${dependency}.name}/build"
)
```
Here, notice that we supply an installation path to CMake. In this case, we set it to a directory inside the main project we install dependency for. This way, the main system is not affected and we can have many project with each thier own dependency set.

Then at that point I realized that I could use external project with this delegated CMake thing, but I decided against it. I needed to manage which version would be checked out and also be able to update branch packages.

When this is done, we can even look if the package has been installed correctly by trying to find the package again.

## 4. Updating Dependencies

CMake already come with a way to specify a requested version for a package when using `find_package`. The package itself will check if it's compatible with the requested version, no a predefind match. This is powerful since different libraries may have different policies reguarding when breaks happen.

This is all handled in the generated `CMakeLists.txt` to check if a dependency exists.

If a dependency is not found and the repository has been downloaded before, we simply have to checkout the right tag (or branch) that satisfies the version constraint:

```cmake
execute_process(
    COMMAND ${GIT_EXECUTABLE} checkout ${${dependency}.tag}
    WORKING_DIRECTORY "./.${${dependency}.name}"
)
```

It may also happen that we are not using a tag, but a branch. In this case, the branch should be updated, reguardless whether the dependency is found or not. This again, stays relatively simple:

```cmake
execute_process(
    COMMAND ${GIT_EXECUTABLE} pull
    WORKING_DIRECTORY "./.${${dependency}.name}"
)
```

If the branch has been updated, we then build and install. To save time and energy, we only do that when new commit was added to the branch since the last update.

This part was very important for my projects and the people that helped us: I could focus on the framework and add dependencies without worrying about updating each machine everytime a less technical person want to start working. The tool would update both the framework and the libraries we depend on.

### Bugs When Updating Packages

With the model I've been working on, I choose the specify two commands: `subgine-pkg update` and `subgine-pkg install`. The update were to pull every branch repository, and the install would simply download and install missing dependency.

Everything was nice until I hit that error:

>     error: pathspec 'v3.5.2' did not match any file(s) known to git

What happened here?

As it turns out, simply using `git checkout <version>` is not enough. If the new version has been created after cloning the repository and we tell the package manager to install missing package, it will see the outdated package as *not found*. It will also see the existing repository, so no clone needed. Then, the package manager would try to checkout the new version.

The checkout failed because we must fetch updates before.

Adding this code did the job:

```cmake
execute_process(
    COMMAND ${GIT_EXECUTABLE} fetch
    WORKING_DIRECTORY "./.${${dependency}.name}"
)
```

So I had to manage updates even when installing it seemed. So the only difference left between the `update` and `install` command is that `update` will pull repositories set to a branch instead of a tag.

## 5. Ensuring A Valid State

When running the tool, it's easy to know when to rebuild. When a package has been updated, rebuild and that's it?

Unfortunately, no. What happens when a package fails to build or install correctly? The next time the package manager runs, it should be able to recover, see that a package failed the last time and try again.

At first, it did not happened. When writing the package manager, the logic was simple: if the options for a package, if the version changed or the branch updated, whatever, save those new options and rebuild! The problem is that it won't do well when a package failed to build for whatever reason. When invoking the package manager again, it's gonna see that nothing has been updated in that invocation. This is bad, since the package manager will finish without error, but some packages may be out of date!

To fix this, the package manager has to keep something around to know if a package hasn't been built correctly. Our implementation simply save the commit id of the last successfully installed version, and the options that that installed version was built with.

To do that correctly, I made myself a nice little command for that:
```cmake
function(dependency_current_revision dependency return-value)
	execute_process(
		COMMAND ${GIT_EXECUTABLE} rev-parse HEAD
		WORKING_DIRECTORY "${sources-path}/${${dependency}.name}"
		OUTPUT_VARIABLE revision-current
		RESULT_VARIABLE revision-current-result
		ERROR_VARIABLE revision-current-error
	)
	
	if ("${revision-current-result}" EQUAL 0)
		set(${resubgine-pkg-revision.txtturn-value} "${revision-current}" PARENT_SCOPE)
	else()
		message(FATAL_ERROR "Git failed to retrieve current revision with output: ${revision-current-error}")
	endif()
endfunction()
```
Then use it like that when an installation is completed successfully:
```cmake
dependency_current_revision(${dependency} current-revision)
file(WRITE "${${dependency}.name}/build/subgine-pkg-revision.txt" "${current-revision}")
```

Keeping that file in the `build` directory of the dependency has a nice property of being alonside other build artifacts. If there's a need to delete all build artifact, the last built version is also deleted.

### Avoiding The Package Registry

At first I found the package registry quite handy at first. Especially on windows, where there was no system package manager, I could just build a library somewhere and use it elsewhere, no need to install. I first assumed the package registry existed when I made the package manager, but reading the state of the packages and verifying the state was becoming quite difficult.

A package could be found installed on the system, as a subdirectory of `CMAKE_PREFIX_PATH` or, with the package registry, *anywhere* else. Even in the package directory of another project using subgine-pkg!

Then, that repository in the other project could had some problem building the packages or could have its process interrupted or whatever. For a package to be found by the package registry, it only need to be configured. If it's not compiled correctly, a *build time error* will occur, so my package manager would have to check the state of the other package manager's project to be sure everything has been built correctly. Or even worse, you could get back to the other project  and the package manager could be confused by the corretly built package in the other project while it reads its corrupted state. Bad bad bad!

The solution was to simply disable it. When building packages I pass `-DCMAKE_FIND_PACKAGE_NO_PACKAGE_REGISTRY=ON`. When doing that it became much simpler to reuse package installed by other project in a more predicatble way. More on that later.
