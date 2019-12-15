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

Why is that? And why on earth using CMake as a scripting language? Today I'll explain my needs at the time, the thought process behind, my implementation choices it and show you the result.

<!--more-->

## Preface

I would like start by saying that I'm not an expert in build systems. I may have done some obvious errors or oversight. I just had a need that I solved using to tools I knew.

Also, I needed simplicity both in setup and maintenance. I was working with a team that had very little experience with C++, and didn't know how to manage dependencies. We also had multiple platforms and different setup.

At the time, supporting how we setup the developpement was important. I was developping a framework and an app that used it. The rest of the team was only developping only the app and for them, the framework was a dependency like any other. Simply asking to the package manager to update everything should be one command and should update the framework to it's latest revision, and also update the framework's dependencies.

On my side, the app should use the framework I had compiled in another directory. Not installing the framework on every changes was important for me, and simply pushing on the master branch should be all I had to do to make the update available to the rest of the team.

Both vcpkg and conan didn't seem to propose all these properties, or didn't seemed simple enough for the whole team to use. The situation might have changed since the last time I tried this setup with these tools.

## A Dependency Problem

At first, dependencies were not a problem. I used Linux and my mate too. We wanted something? We just had to pick it from the system package manager, write a small `FindXYZ.cmake` file if needed and then use the library.

We used libraries like `glfw`, `jsoncpp`, `glm` and a few others. But one day, I made a small framework to support our project. So we had the main project that depended on some libraries, and our framework that also had dependencies. We had a multi-level dependency problem. Our workflow was to build the missing libraries one by one in order. Luckily, most of them was header only, so it wasn't that bad, but required some knowledge of the setup to do it correctly.

When we needed a new dependency that wasn't in the system repository, we simply added it to the list of packages to build beforehand... in the right order... And be careful to signal our teammates! Well, it was becoming quite tedious, something had to be done.

## Submodules

The idea of using submodules to ship dependencies was considered at first. I could fetch the submodule of only the libraries I needed. However, it turns out git is not a package manager. Submodule was hellish to support, and updates would have been a pain.

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

However, I migrated from a 70 line shell script to a... 1000 line CMake monster.

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

Sadly, external project cannot be used either in script mode. Also, it don't quite make sense to use it there since external project downloads at build time.

The solution was to run git manually to download the dependency, then build it.

I then realized after doing all this that there was `fetchContent` that did what I wanted. However, I use more git commands than what `fetchContent` exposes so I kept the raw git.

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
      set(${return-value} "${revision-current}" PARENT_SCOPE)
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

The solution was to simply disable it. When building packages I pass `-DCMAKE_FIND_PACKAGE_NO_PACKAGE_REGISTRY=ON`. When doing that it became much simpler to reuse package installed by other project in a more predicatble way.

## 6. Linking Workspaces Together

I really wanted to support my own workflow where I developped the framework and the app, side by side, without having to install the framework everytime I made a change in it. To do that, the package manager have to discover the build directory of the framework. It turns out CMake can already do that if the project supports it by exporting its build tree. Do support it in your own project, add these line to the installtion part of the script:
```cmake
# Normal package exportation when installing
install(
  EXPORT mylibTargets
  FILE mylibTargets.cmake
  NAMESPACE mylib::
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/mylib
)

# Export the build tree
export(
  EXPORT mylibTargets
  FILE "${CMAKE_CURRENT_BINARY_DIR}/mylibTargets.cmake"
  NAMESPACE mylib::
)
```
After that, project can set the `mylib_DIR` or `CMAKE_PREFIX_PATH` to that directory, and `find_package` will be able to see the library.

How does it helps for reusing packages?

Well, I don't support reusing packages across unrelated projects, but if the package manager can read the build tree of another project that itself uses the same package manager, surely it could read it's own metadata and use the packages that are already there right?

To do that properly, when running CMake on a project that uses subgine-pkg, I create a file in the build tree that looks like this:
```cmake
# in a file called subgine-pkg-mylib-default.cmake in its build directory
set(found-pkg-mylib-prefix-path "~/some-prefix/build;/path/to/mylib/subgine-pkg-modules/") # needed prefix path
set(found-pkg-mylib-module-path "")                                                        # module path (if needed)
set(found-pkg-mylib-manifest-path "/path/to/mylib/sbg-manifest.json")                      # path to manifest file
```

But of course, that for is not written by the user of the tool. CMake running in the user's project must output that file in its build directory and must setup the prefix path to the right places. This is all done by injecting a script I call a profile. More on that later.

So if I find a dependency, I can just use `find_file`, since if the package is found through the prefix path, the find file command should find the metadata file, and include it to get the variables values!
```cmake
find_file(sbg-package-config-file subgine-pkg-${${dependency}.name}-${current-profile}.cmake)
include("${sbg-package-config-file}")
list(APPEND prefix-paths-to-use ${found-pkg-${${dependency}.name}-prefix-path})
```

Here you go! Now, the `find_package` command will also find packages from a project configured with subgine-pkg!

This helped me reduce the amount of duplicated package and setup the projects faster, thus removing waits in between iterations and make them lighter for my system.

## 7. Profiles

One could imagine needing dependencies to be built for many different incompatible setups. For example, on windows I'll need a build for debug and release. On linux, I might want a clang and a GCC build, or a build with sanitizers enabled. To do that and support all dependencies to be built with the same option, I came with profiles. Inside the `subgine-pkg-modules/` directory, I have an installation subdirectory for each profiles. So the structure look like this:
```
subgine-pkg-modules
├── profile-a
│   ├── lib1
│   └── lib2
└── profile-b
    ├── lib1
    └── lib2
...
```
Then, when compiling all packages for each modules, I can output a file that contains the a prefix path pointing to the right profile installation path.

For each profile the package manager supports a set of CMake argument, including CXX flags, toolchain files and others. This mean cross compiling dependencies is possible, although I never tested it.

### Generating Profiles
Using the command line interface, setuping a profile looks like this:
```sh
$ subgine-pkg setup my-profile -DCMAKE_CXX_FLAGS="-fsanitize=address" ... any other cmake args ...
```
Then, build the profile:
```sh
$ subgine-pkg install my-profile
```
This will build all packages with provided CMake argument, and also generate a file that adds required prefix paths and other variables.

The `update` command also takes which profile it should run for. I plan to also add support for profiles in the `clean` command, but the current behavior of operating in all profiles works for now.

### A missing profile feature

What I would love to do would be for the main project to run with the same arguments as the profile. Sadly I currently need the project name in the generated files. I also don't know if all variables such as toolchain files and other can be set programmatically before `project()` calls. I'll have to dig a bit deeper for that one.

## 8. Using The Packages From CMake

After that configuring and building and installing of all the packages, there must be a way to find those packages and use them from the user's project.

There's a CMake feature almost made for this: code injection. The variable `CMAKE_PROJECT_INCLUDE` tells to CMake to include a particular after the `project()` command is called. So without even changing our CMake project file, we can integrate our package manager!

```sh
$ cmake .. `-DCMAKE_PROJECT_INCLUDE=subgine-pkg-modules/default-module.cmake
```

And the day you want to switch to Conan, simply change which file you include there to the one Conan generates!

The profile don't contain much wizardry. It's quite simple in fact. There is not much to do to in this file. Before anything, we set some metadata to be available for the CMake project that uses the profile:

```cmake
# We set some variables to let know the
# CMake script that subgine-pkg has been included.
set(subgine-pkg-${PROJECT_NAME} ON)
set(subgine-pkg ON)

# We let the current CMake script know what profile has been used.
set(subgine-pkg-${PROJECT_NAME}-profile "${current-profile}")
set(subgine-pkg-profile "${current-profile}")
```

Then, we add the prefix where the libraries are installed. if the user specified any prefix or module path while calling setup, we also set it there:
```cmake
# Installation prefix of the libraries
list(APPEND CMAKE_PREFIX_PATH "${CMAKE_CURRENT_SOURCE_DIR}/subgine-pkg-modules/prefixes/${current-profile}/")

# Prefixes and module paths sent to the `subgine-pkg setup ...` command
list(APPEND CMAKE_MODULE_PATH "some;paths")
list(APPEND CMAKE_PREFIX_PATH "some;other;paths")
```

To be correct, variables such as `somelib_DIR` and `somelib_ROOT` should also be considered there but it's not supported yet.

## 9. Reusing Installed Packages

When two workspace are linked together by the package manager, would it be nice if the dependencies would not be installed multiple times?

We support doing that by emmitting some instructions in the profile file that write some required metadata to link workspaces and their dependencies together.

So inside the `wathever-profile.cmake`, you'll see something like this:
```cmake
file(WRITE "${PROJECT_BINARY_DIR}/subgine-pkg-${PROJECT_NAME}-${current-profile}.cmake" "
set(found-pkg-${PROJECT_NAME}-prefix-path \"${CMAKE_PREFIX_PATH}\")
set(found-pkg-${PROJECT_NAME}-module-path \"${CMAKE_MODULE_PATH}\")
set(found-pkg-${PROJECT_NAME}-manifest-path \"${PROJECT_SOURCE_DIR}/sbg-manifest.json\")
")
```
> Okay... what is going on there?

We create a CMake file that is going to be read by other projects. We carefully set metadata about subgine-pkg itself.

We set all required variable for this particular build tree need for it to be correctly found by `find_package`. If there's one prefix path missing, a dependency may not be found and the package cannot be used!

Since the build directory can be anywhere and completely separated from the source directory, we also tell the other projects where to find the manifest file.

After that instruction, we try to find this kind of file that could have been outputted by our dependencies:
```cmake
# We try to find the file from available prefix paths
find_file(subgine-pkg-setup-file-${dependency-name} subgine-pkg-${dependency-name}-${current-profile}.cmake)

# If we indeed find a file, it means the other project is a workspace that uses subgine-pkg
if(NOT "${subgine-pkg-setup-file-${dependency-name}}" STREQUAL "subgine-pkg-setup-file-${dependency-name}-NOTFOUND")
    # including the file will make `found-pkg-xyz` variables available
    include("${subgine-pkg-setup-file-${dependency-name}")
    if(NOT "${found-pkg-${dependency-name}-prefix-path}" STREQUAL "")
        list(APPEND CMAKE_PREFIX_PATH "${found-pkg-${dependency-name}-prefix-path}")
    endif()
    if(NOT "${found-pkg-${dependency-name}-module-path}" STREQUAL "")
        list(APPEND CMAKE_MODULE_PATH "${found-pkg-${dependency-name}-module-path}")
    endif()
endif()
```
Here we try to find a file with the same profile name as our current profile. We consider that if the prefix path don't point to the another profile with same profile name, it's probably a mistake. That file exist only when creating and compiling your own projects inside different directories and is only found if a prefix path point to it. The user is in control on the profile name and thier arguments so we assume the same profile name means compatible.

## Result

The result? In a few weeks of work before the project started, I was able to create a small tool that worked just fine for our needs and enabled us to work. It's a recursive package manager, so package requiring other packages was possible and even required for our setup as we intended.

[![asciicast](https://asciinema.org/a/288196.svg)](https://asciinema.org/a/288196)

The package manager is capable is picking up built packages from other worspaces, installed packages (optional) and can also update our installed libraries.

The package manager is still useful to me. I still use it and for well built CMake packages, it work like a charm for my needs.

## What I've Learned

CMake has a lot built-in to manage packages, versions, install them at the right place and more. What CMake does not have is a  tool to download packages and build them with a config. This is my attempt of making that missing tool.

I also learned that not all project support building with CMake. I told you I assumed the recipe? Yeah, to use some project I had to wrap them in a CMake project to use them with my package manager.

Also, there's a lot of projects that uses CMake, but are not `cmake --build . --target install` friendly, or just don't expose a CMake package at all. Some exposes a CMake package, but not their version number, so I had to add a `"ignore-version"` option on a dependency, and then upgrading that library was a pain since the package manager would not know it was out of date. Relying on CMake is great if everyone uses it correctly.

To use some packages, I simply contributed to some libraries I needed. I think it's the best way to get more library with a quality CMake setup: make that quality setup and contribute to make a better world! I learned how to make libraries useable with CMake in a better way doing these contributions, and I gained a lot of experiences porting some proejcts too.

I learned a lot making mistakes doing that tool. It really easy to get something working quickly, but to make something robust and reliable it quite a challenge and I still correting small bugs months after the project is finished.

The CMake language has made it easy to get started, but as the tool is becoming more complex, I think using C++ directly would have been great. Making it cross platform may be harder though and I was constrained by time.

Overall I'm happy I made this project. I mean, it's not perfect but it's been really useful for me, and saved me a lot of time. I would like to hear why that wools would ne be suitable for you or how can I improve it.
