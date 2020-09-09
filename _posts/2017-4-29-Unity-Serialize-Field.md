---
layout: post
title: Unity’s [SerializeField] demystified
---

Hi there, Unity developers!

As you work with Unity Editor, you use the Inspector window pretty frequently. From here, you can do many things in order to have your Unity game works as intended, such as modifying graphics quality, increasing the value of gravity, or changing a texture’s import settings. But I believe most of us spend most of our time with Inspector window to tweak our `GameObject`s and their values. For programmers, in Unity 101 lessons, we learned that a `public` attribute would show up in the Inspector window.

However, making an attribute `public` is not always healthy for your code…

## `SerializeField` comes to rescue!
So first off, let me explain why public attributes are one of the potential risks for your project. They can be accessed from nearly anywhere outside your class, so they encourage two bad things: coupling code and complexity. This means, in short, that you will have to look at more than one place to find the source of an error, and as a result, the development process of the game is slowed down.

But I cannot deny that editing a value in the Inspector window rather is easier, more designer-friendly and less time-consuming than in a code file. I asked this matter when I was an intern student, and one of my seniors told me to use `[SerializeField]`, and hurray, my private variables show up nicely in the Inspector windows.

In later stage of my career, I figured out that this magical tag is not anything magical at all. This relies on a technique which is called _serialization_.

## What the hell is serialization?
Let us consider an example. When the player quits from your game, you want the information of his current location in the world, his current HP, his current weapons,… to be stored, and when the player plays your game next time, he will continue playing from that. In order to do this, you have to write those vectors, integers, floats, strings… into a file, and load this file when the game is opened again.

The process of storing objects into a file for future uses is called _serialization_. In the opposite way, when you open the file and reconstruct the objects, the process is called _deserialization_.

Unity uses the same approach: all the tweaks and modifications on your prefabs and game objects are stored in a file (.prefab or .scene), and they are loaded and processed when you the game is played. The Inspector window is just a tool to make this work easier and less painful by providing the ability to observe and edit serialized values. By default, public attributes are serialized. The private attributes, if they are needed to be serialized, have to use the `SerializeField` tag.

## Let’s take a closer look at a Unity .prefab file
I believe after reading the whole bunch of text above, an example would make you understand the concept much better. I write a C# script, which contains nothing but a few serialized private variables:

```csharp
using UnityEngine;
 
public class TestSerialize : MonoBehaviour
{
    [SerializeField]
    private int privateSerializedInteger;
 
    [SerializeField]
    private float privateSerializedFloat;
 
    [SerializeField]
    private string privateSerializedString;
}
```
Then I create a prefab, attach the script above to it. The serialized attributes are given some values. This is how it looks on Unity Inspector panel:

![](https://raw.githubusercontent.com/tongtunggiang/tongtunggiang.github.io/master/assets/images/SF_Editor.png)

I will open the .prefab file in a text editor. Voila! The variable names and their corresponding values are saved in the .prefab file as shown below. If your prefab file is not displayed correctly in the text editor, your project’s serialization settings is probably set to binary mode. To view your prefab in a text editor, go to _Edit -> Project Settings -> Editor_ and change the _Asset Serialization_’s Mode to _Force Text_.
```
--- !u!114 &7874586077219231128
MonoBehaviour:
  
  privateSerializedInteger: 10
  privateSerializedFloat: 10
  privateSerializedString: Hello
```

# Conclusion
In this post I have explained the advantage of the `[SerializeField]` tag on your project for both programmers and game designers, the underlying mechanism of it and a quick example. Hopefully you find this helpful and interesting.

May the `[SerializeField]` be with you!
