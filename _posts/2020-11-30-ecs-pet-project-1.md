---
layout: post
title: Pet project: Unity and EnTT
---


#### Table of contents
- Part 1: Unity and C++ (this post)

# Unity and C++
I have always been curious about using C++ in Unity.

This isn't a new problem, at least for me personally. I've encountered several posts around the Internet about doing so, sadly they are just naive examples with a Unity project calling some arbitrary functions from a native dll written in C++. Most of these examples usually do nothing but returning an integer (`return 42;`), whether directly or by a dumb adding statement (`return a + b;`).

While they do introduce the concept of [_platform invocation_](https://docs.microsoft.com/en-us/dotnet/standard/native-interop/pinvoke) and _interoperability_<sup>1</sup>, these examples is close to useless. Let's put aside the fact that they're trivially simple. There are more important questions of how these knowledge should be applied into a complete project.
- What is a complete solution to handle name mangling problem, especially when it comes to classes, objects and so on?
- How about the opposite way, calling C# from C++?
- How do we apply a productive workflow for fast iteration, as Unity doesn't reload native dlls once they're loaded?
- C++ and C# handles objects in memory in different ways, how can we work arounds this?

> (1): _interoperability_ - short hand for _inter operablility_, the ability of different components to communicate to each other, in this case, C++ and C#.

Back in September, I encountered [an outstanding series by Jackson Dunstan about using C++ in Unity](https://www.jacksondunstan.com/articles/3938). The solution, or if I'm precise, the collection of little solutions, described in the series really enlightened me. Not only it solved many technical challenges I listed above, the author also did a terrific job explaining his ideas and intentions. This is an essential skill for many engineers and I am certainly benefitted by reading the series, as an engineer and a hobby problem solver.

I will recommend reading the series if you are interested, but here are the main points of the solution. First off, to call C# from C++, the author used `Marshal.GetFunctionPointerForDelegate` to get the C# function's address and pass this value to the C++ side. Calling C++ from C# is easier, all we need is the exact function name, marked with `extern "C"` and `dllexport`, and the DLL file's name.

So far, the two-way communication between C++ and C# only handles basic function calls - we need more than that. However, C++ cannot understand the managed C# types, even `String`, so that the author employed a way to [access a managed C# object with an `int`](https://www.jacksondunstan.com/articles/3908). The snippet inside the linked article is also very useful if we want to do this in an unmanaged - managed scenario in C#.

We cannot restart Unity Editor every time we make any change to the C++ code, it's a productivity killer. The author's solution here is very clever and I'm amazed. Relying on the fact that every operating system provides functions for loading, closing and using native libraries, the author used `[DllImport]` on these libraries. Our C++ functions are now loaded and unloaded manually by using the OS native libraries every time we press Play in the Unity Editor. Since the OS native libraries are shipped with the OS, it's very little chance that they're updated while we have Unity opens, especially on Windows where updates are made when the user restarts the computer.

The last major piece of the complete solution here is a bindings generator, which generates glue code in both C# and C++ based on information given by a JSON file. It's handy to have so that we can control what we want to expose to the C++ side. The details of the bindings generator is quite complicated and I shouldn't rant on it here.

Once again, I really recommend reading the series.

**TL;DR**: This is my technical experiment, and to be honest I have absolutely no idea what I'm doing.