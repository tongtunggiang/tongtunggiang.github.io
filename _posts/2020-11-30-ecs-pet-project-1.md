---
layout: post
title: Pet project: ECS, Unity and EnTT
---


#### Table of contents
- Part 1: ECS, Unity and EnTT (this post)

# Unity and C++
I have always been curious about using C++ in Unity.

This isn't a new problem, at least for me personally. I've encountered several posts around the Internet about doing so, sadly they are just naive examples with your Unity project calling some arbitrary functions from a native dll written in C++. Most of these examples usually do nothing but returning an integer (`return 42;`), whether directly or by a dumb adding statement (`return a + b;`).

While they do introduce the concept of _platform invocation_ (P/Invoke) and _interoperability_, these examples is close to useless. Let's put aside the fact that they're trivially simple. There are more important questions of how these knowledge should be applied into a complete project.
- What is a complete solution to handle name mangling problem, especially when it comes to classes, objects and so on?
- How do you apply a productive workflow for fast iteration, as Unity doesn't reload native dlls once they're loaded?
- C++ and C# handles objects in memory in different ways, how can you work arounds this?

Back in September, I encountered [an outstanding series by Jackson Dunstan about using C++ in Unity](https://www.jacksondunstan.com/articles/3938). The solution described in the series completely blown my mind away.


**TL;DR**: This is my technical experiment, and to be honest I have absolutely no idea what I'm doing.