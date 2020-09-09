---
layout: post
title: Create a multi-function button in Unity using State Design Pattern
---

## The problem

Today at work, I was assigned a task of implementing a button – it seems to be a trivial task. There is an interesting point of this button, though. It reacts differently in different situations, that’s why I call this button “multi-function”.

For example, imagine your game has an inventory box, where the player stores his or her items. For a game mechanism issue, the player can only carry one item at a time. The player can also buy a new item, upgrade or equip himself an existing one. The multi-function button handles all these tasks. Let’s take the Sniper Fury’s Amory screen as an example:

![](https://i.imgur.com/J7vWutt.gif)

Spotted that shinny little button in the lower right corner? That’s what we are aiming at: a single button for multiple purposes.

## The initial solution

The easiest way to handle this, is using if conditional statements. You use the current situation as the input, and perform the respective action(s) with that situation.

```csharp
using UnityEngine;
 
public enum Situation
{
    Buy,
    Equip,
    Upgrade
}
 
public class MultiFunctionButton : MonoBehaviour
{
    public Situation situation;
 
    public void OnClick()
    {
        if (situation == Situation.Buy)
            Buy();
        if (situation == Situation.Equip)
            Equip();
        if (situation == Situation.Upgrade)
            Upgrade();
    }
 
    private void Buy()
    {
        Debug.Log("Buy");
    }
 
    private void Equip()
    {
        Debug.Log("Equip");
    }
 
    private void Upgrade()
    {
        Debug.Log("Upgrade");
    }
}
```

However, this solution is very inflexible. As you can see, the above code is mere functional, it deals nothing with the appearance of the button, such as changing the sprite or changing the text. You will have to handle these details in real life problems, it’s no doubt about that.

And what if, you need another type of function that the button has to perform, such as give it away, or decorate it with funny sticker? Iterating through every place to add new functionality is really the pain in the ass, and to be honest, very error-prone.

## The improved solution
This leads us to the improved solution: State Pattern. Basically, it is the solution where you encapsulate the situation-specific details into objects. The button knows nothing of these details. Let the code speaks itself.

Firstly, I make a common interface for the situations, or state, which provides a method for receiving click event.

```csharp
public interface IState
{
    void OnClick();
}
 
public class EquipState : IState
{
    public void OnClick()
    {
        Debug.Log("Equip");
    }
}
 
public class BuyState : IState
{
    public void OnClick()
    {
        Debug.Log("Buy");
    }
}
 
public class UpgradeState : IState
{
    public void OnClick()
    {
        Debug.Log("Upgrade");
    }
}
```

Then, in the class that I attached to the button, I will only have a variable representing current state of the button (which can be “Buy”, “Upgrade” or “Equip”).

```csharp
public class MultiFunctionButton : MonoBehaviour
{
    public IState currentState;
 
    public void OnClick()
    {
        currentState.OnClick();
    }
}
```

Compare to the previous solution, this solution is nice, neat and clean. The `MultiFunctionButton` class only needs to tell its current state that “Hey, I just have been clicked!”, and let the state object handles the work itself, with the details that only the state object knows.

When the needs of adding new state arrive, the `MultiFunctionButton` class remains unchanged – you just need to add another state class derived from IState. Even better, the `MultiFunctionButton` class doesn’t even notice that there is a change. This means this part of your code is well decoupled, clean, and easy to maintain.

I then assign the `MultiFunctionButton` class’ OnClick method to the `OnClick` event of the Canvas button, and let us see the result:

![](https://i.imgur.com/ZI27OK8.gif)

# Final words
You might notice that I used a public variable in `MultiFunctionButton` class. This is not a good practice at all – I don’t like my variables being exposed nakedly without any protection, but in the scope of this blog (which is only for demonstrating an cool idea in my opinion), this is not harmful to me. In real problems, however, I highly recommend using a layer of protection to the variable, specially when it affects directly to the functionality of the object it belongs.

Finally, I am curious to hear what you think about this. If you find there can be any improvement to my solution, please don’t hesitate, just put it in the comment box.

Cheers,
