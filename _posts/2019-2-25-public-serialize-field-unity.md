---
layout: post
title: Why you should prefer [SerializeField] over public variables
---

When I published a post about [how [SerializeField] attribute works under the hood](https://tongtunggiang.github.io/2017/Unity-Serialize-Field/) earlier, I made a claim that public fields can cause potential risks. Experienced programmers usually see this as something obvious, as the Earth goes around the Sun. However, with new or self-taught programmers, the threats of public fields are somewhat intangible. If you feel something like that, worry not, because I am here to discuss why I made the claim, and why many other programmers happily agree on this with me.

# “Those two seem exactly the same for me”

At first, making a variable public and making it privately serialized seem to be equivalent. These two ways expose the variable’s value in the Unity Inspector so that anyone can go there and make some change on that value, say, from 5 to 10. That means whether you’re doing this:
```csharp
public class Character : MonoBehaviour
{
    [SerializeField]
    private int healthPoint;
}
```
… or this:
```csharp
public class Character : MonoBehaviour
{
    public int healthPoint; // Bad idea, really
}
```
… then you still have the same result in the Unity Inspector.

Imagine that one day, your colleague Dave is going to use your code to do something for his feature. The difference starts to show. If you use the public keyword on your variable, then Dave’s piece of code below is perfectly legal. But it won’t be if you use `[SerializeField]` and make your variable private – which means it is only accessible from the scope of your class.
```csharp
public class Foo: MonoBehaviour
{
    void Update()
    {
        Character c;
        Debug.Log(c.healthPoint);
    }
}
```

# Something public means anyone can do anything they want with it
The difference seems to be trivial – Dave can use this variable on his code, or not, end of story. What is so big of a deal about this? Why should I hide my data fields from my colleagues like Dave?

So your data field is public. Dave just fetches whatever value it holds to do his job, say, display the health value on a health bar. It’s okay because it does not affect your job. But Chuck has other ideas. He wants to do a powerup which doubles the character’s health when being triggered, so he just needs to grab your variable and make the _direct_ change on it – because you made it public:
```csharp
Character c;
c.healthPoint = c.healthPoint * 2;
```
This quickly runs into a problem. As you can see, there is no limit of your character’s HP value, it can increase until it reaches the max value of the int value type. This is not good, so you have to add a variable, and let’s call it max HP. The previous HP variable now represents the _current_ health of your character, it cannot be less than zero or go pass the max value defined. Then say, when the character’s health is 80/100 and he triggers Chuck’s powerup, the character’s health will be 100/100, instead of 160/100 (which, by the way, is complete nonsense).
```csharp
public class Character : MonoBehaviour
{
    public int healthPoint;
    public int maxHealthPoint;
}
 
 
// Chuck's code
Character c;
c.healthPoint = c.healthPoint * 2;
if (c.healthPoint &gt; c.maxHealthPoint)
    c.healthPoint = c.maxHealthPoint;
```
Chuck’s case rings a bell about how vulnerable your class is with public fields. Software requirements, including games, do not stand still throughout the development phase – in fact, they change _way more frequently_ than you think they do. Making your data fields public may provide convenient access to them, but the approach is very weak in reacting to requirement changes. Now, let us see how things might change if you use the opposite approach, using private data fields while keeping the ability to edit the value through Unity Inspector when necessary.

# It's time for some secrets
```csharp
public class Character : MonoBehaviour
{
    [SerializeField]
    private int maxHealthPoint;
 
    private int healthPoint; // Notice that I removed [SerializeField] on purpose
 
    public int HealthPoint { get { return healthPoint; } }
    public int GetHealthPoint() { return healthPoint; }
 
    public void Damage(float value)
    {
        healthPoint -= value;
        if (healthPoint &lt; 0) 
            healthPoint = 0; 
    } 
 
    public void Heal(float value) 
    { 
        healthPoint += value; 
        if (healthPoint &gt; maxHealthPoint)
            healthPoint = maxHealthPoint;
    } 
}
```
Now, you only provide editing ability to the max value of your character’s HP. The current value of his current HP is now completely hidden from outside modification, and it makes sense, as you don’t want everyone to change his current HP, especially in Play mode, as this could make some features collapsed. In addition, this value is now read-only, because the Character class only provides a property and a function for reading the actual HP value.

On a side note, if you are coming from a C++ background like me, then you will be familiar with the getter function. Actually, I _prefer_ getter functions than the property, because it says explicitly its purpose while the property seems pretty much like a variable. Furthermore, finding getter functions’ usage in the whole code base is way easier, when you don’t have to filter out reading calls like when you do with properties. But still, this is mostly the matter of personal preference, you are free to pick whatever you’re comfortable with, be it setter/getter functions or properties.

With that being said, when Dave or Chuck try to access your current HP value directly, they will have compilation errors thrown into their faces, which is a good thing. The type system does its job, it prevents any private data to be accessed from anywhere outside of their class scope. The only way for Dave or Chuck to read or write the character’s health value is through your predefined way. For example, Chuck’s powerup uses the Heal function, which guarantees that the character’s health will never surpass its maximum value, even when you pass a positive value three times the max value.

When the data is guaranteed to be changed only within the class scope, you have the total control to monitor, maintain and update the way that data is processed. If you take a look at Heal and Damage functions above, you can see that there is no handling for negative values. Which is, when accidentally (or maybe, purposefully) passing a negative value into the function, things might be messed up – the Damage function actually makes your character stronger, while Chuck’s powerup kills the player – so cruel!!

So that is clearly a bug, then all you have to do is adding negative-case handling code. Chuck and Dave’s code is remaining the same and untouched, how wonderful! Imagine how hard things would be if you use public variable. Dave would have to handle cases when current health value goes below zero. Chuck is even more miserable. He has to do everything by himself, which significantly makes his code more cluttered. Negative delta values, maximum HP, and I will not even mention when the game design document updates the character equipment, which would increase the character’s maximum health. When that happens, Chuck will very likely get rage and find you to smash his keyboard to your head.

Moreover, given that you use public max HP variable, and your Character class gradually grows with features like equipment, inventory or weapon handling code, and you decide to move health-related stuffs into another component. Guess how many errors you’ll be receiving? Maybe none, but chances are you’re going to have a wave of errors to work on because your public variables are used in a unknownably amount of code everywhere across your project. Even when you clear out the errors, it would take some serious hard testing and fixing to ensure things are going to work well as they did before you made your refactor.

# Conclusions
Everything happens for some reasons. Hiding and protecting your data with private access (or, if you wish to use a “more professional” term: encapsulating) is no exception. With all the risks above, I believe that making a variable public just to make use of Unity Inspector’s ability to edit value causes more harm than the benefits it brings. Private and `[SerializeField]` marked fields do the job just fine, while still keeping all the advantages of encapsulating.

I admit that when you first write out your code, making the variables public is way quicker than writing additional accessor code. But remember, a minute of laziness might cost you hours of additional work in the future.

So don’t make your variables public. If you do, you better have very good reasons to do so.
