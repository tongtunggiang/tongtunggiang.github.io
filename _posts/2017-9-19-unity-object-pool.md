---
layout: post
title: A simple Object Pool for better performance for your Unity games
---

With game programmers, optimization and performance are interesting things to discuss. We all love to squeeze our hardware as much as possible, to achieve more stunning graphics, more attracting gameplay, with the best performance (damn that’s greedy). Object Pool is one of the common technique of optimizing game performance, and today I’ll show you my object pool implementation. It’s simple, easy to do and most of all, its real-life use is very close to Unity’s mechanism of creating and destroying objects.

# A quick look at the pattern’s intention

At its simplest, the pattern looks for reusing the frequently-used objects, rather than constructing and destructing them continually. Some examples of frequently-used objects are bullets of the player or tiny creeps that keep coming to bite him, no matter how much he has slain.

You would ask “Why reusing?”. First off, `Instantiate` and `Destroy` are expensive calls. Just turn on your Profiler, and take a look at how many seconds they cost when they are called. If your game object is complex, you should prepare for a more time-consuming function call.

Secondly, they create _garbage_. C# is a language where you just need to allocate the memory for an object without minding about deallocating this memory chunk. C# provides a _garbage collector_ to automatically collect the chunks that are no longer in use for you. These chunks are called garbage. The more garbage in your memory, the more work for the garbage collector. And the more work it does, the worse your game’s performance, simply put.

And last but not least, they cause _memory fragmentation_. Dynamically allocated objects are stored in a zone in your RAM, called the _heap memory_. Basically, your game would scan its heap, to find a chunk large enough for the object that would be created. This is not cheap.

![I took this image from the amazing site gameprogrammingpatterns.com. I love you, Bob Nystrom!
](http://gameprogrammingpatterns.com/images/object-pool-heap-fragment.png)

It can get _even get worse_. If your game cannot find a place to put the object, it has to ask the operating system for expanding its heap. Sometimes it is fast, sometimes it is slow. The operating system is busy, so it cannot guarantee an immediate response to your game.

I don’t mean to frighten you, but it can _even get worse_, in some extreme cases. Sometimes, when your RAM is full, the operating system cannot expand the heap for your game – “Hey dude, I’m sorry but I’m out of memory”. Then you will see your game crashes into dust…

So, the Object Pool pattern looks to prevent this, by reusing objects rather than creating and destroying them. This approach provides some benefits:

- Activating and deactivating objects are much cheaper than creating and destroying. Imagine “crafting a chair every time you want to sit” vs. “craft a chair once, then put it somewhere in your home and take it out when you need to use it”, and you will get what I say.
- No more headaches that are caused by the garbage collector.
- No more worry of memory fragmentation for pooled objects.

# The Pool class
At its core, the pattern has the Pool class for creating a number of reusable objects. This class is derived from MonoBehaviour since it is designed to be attached to a game object in your scene.

```csharp
public class Pool : MonoBehaviour
{
    [SerializeField]
    GameObject prefab;
 
    [SerializeField]
    int initialPoolsize = 10;
}
```

This is how it looks like when you use this class in your scene. Just create an empty game object, attach thePool component to it, drag the prefab of the objects that you want to use and set the initial pool size. This value would determine how large the initial chunk of objects is.
![](https://i.imgur.com/fXiWkmQ.png)

```csharp
public class Pool : MonoBehaviour
{   
    Stack pooledInstances;
    List aliveInstances;
 
    void Awake()
    {
        pooledInstances = new Stack();
        for (int i = 0; i &lt; initialPoolsize; i++)
        {
            GameObject instance = Instantiate(prefab);
            instance.transform.SetParent(transform);
            instance.transform.localPosition = Vector3.zero;
            instance.transform.localScale = Vector3.one;
            instance.transform.localEulerAngles = Vector3.zero;
            instance.SetActive(false);
 
            pooledInstances.Push(instance);
        }
 
        aliveInstances = new List();
    }
}
```
When the pool object is created, I would instantiate objects from the provided prefabs, and deactivate them and put them in the pool. I use a stack for conveniently take the object out of the container, but you are free to use an array or a queue. I also maintain a list of alive objects, which are taken out from the stack. With that in mind, we start to implement the function for activating objects.

```csharp
public class Pool : MonoBehaviour
{
    public GameObject Spawn(Vector3 position, 
        Quaternion rotation, 
        Vector3 scale, 
        Transform parent = null,
        bool useLocalPosition = false,
        bool useLocalRotation = false)
    {
        if (pooledInstances.Count &lt;= 0) // Every game object has been spawned, so we have to spawn a new one and add it to the pool.
        {
            GameObject newlyInstantiatedObject = Instantiate(prefab);
 
            newlyInstantiatedObject.transform.SetParent(parent);
 
            if (useLocalPosition)
                newlyInstantiatedObject.transform.localPosition = position;
            else
                newlyInstantiatedObject.transform.position = position;
 
            if (useLocalRotation)
                newlyInstantiatedObject.transform.localRotation = rotation;
            else
                newlyInstantiatedObject.transform.rotation = rotation;
 
            newlyInstantiatedObject.transform.localScale = scale;
 
            aliveInstances.Add(newlyInstantiatedObject);
            return newlyInstantiatedObject;
        }
 
        GameObject obj = pooledInstances.Pop();
 
        obj.transform.SetParent(parent);
 
        if (useLocalPosition)
            obj.transform.localPosition = position;
        else
            obj.transform.position = position;
 
        if (useLocalRotation)
            obj.transform.localRotation = rotation;
        else
            obj.transform.rotation = rotation;
        obj.transform.localScale = scale;
 
        obj.SetActive(true);
 
        aliveInstances.Add(obj);
 
        return obj;
    }
}
```

The pool would try to take the first available object, in my case, that is the topmost one in the stack, and assign it the given transform parameters. If the stack is currently empty, there is no other choice but to create a new one by calling `Instantiate` function.

Note that the spawned object (either by taking from the pool or instantiating) is put in the `aliveInstances` array. This is useful when we deactivate the object.

```csharp
public class Pool : MonoBehaviour
{
    public void Kill(GameObject obj)
    {
        int index = aliveInstances.FindIndex(o =&gt; obj == o);
        if (index == -1)
        {
            Destroy(obj);
            return;
        }
 
        obj.SetActive(false);
 
        obj.transform.SetParent(transform);
        obj.transform.localPosition = Vector3.zero;
        obj.transform.localScale = Vector3.one;
        obj.transform.localEulerAngles = Vector3.zero;
 
        aliveInstances.RemoveAt(index);
        pooledInstances.Push(obj);
    }
```

As you can see, the `aliveInstances` array is scanned to see if the object obj is instantiated by this pool or not. If it is, we reset its transform and put it back to the waiting stack. Otherwise, it is destroyed normally.

# Who manages these Pool objects?
I admit, this Pool class alone so far is a piece of useless shit. It is not easy to access the pool, given that you have a prefab object in your hand. It is also very inconvenient and inefficient if you have to perform a search for `availablePool` objects via `FindObjectOfType`. So having an object that manages currently `availablePool` objects is a handy solution for this.

The first solution that comes to my mind for a global access point is the Singleton pattern. However, it is more like a global variable for me, and it encourages tight coupling between classes. Therefore, it’s much better to have a solution that can [_“Provide a global point of access to a service without coupling users to the concrete service”_](https://gameprogrammingpatterns.com/service-locator.html) (thank you once again, Bob Nystrom!).

The implementation of the `PoolManager` class would look like this:

```csharp
public class PoolManager : MonoBehaviour
{
    private static Dictionary&lt;GameObject, Pool&gt; currentPools;
 
    private void Awake()
    {
        currentPools = new Dictionary&lt;GameObject, Pool&gt;();
 
        Pool[] childrenPools = gameObject.GetComponentsInChildren();
        for (int i = 0; i &lt; childrenPools.Length; i++)
        {
            currentPools[childrenPools[i].Prefab] = childrenPools[i];
        }
    }
 
    public static GameObject Spawn(GameObject prefab,
        Vector3 position, Quaternion rotation, Vector3 scale,
        Transform parent = null, bool useLocalPosition = false, bool useLocalRotation = false)
    {
        // Locate the pool that provides the prefab
        if (!currentPools.ContainsKey(prefab))
        {
            return SpawnNonPooledObject(prefab, position, rotation, scale, parent, useLocalPosition, useLocalRotation);
        }
 
        return currentPools[prefab].Spawn(position, rotation, scale, parent, useLocalPosition, useLocalRotation);
    }
 
    public static void Kill(GameObject obj, bool surpassWarning = false)
    {
        foreach (KeyValuePair&lt;GameObject, Pool&gt; pool in currentPools)
        {
            if (pool.Value.IsResponsibleForObject(obj))
            {
                pool.Value.Kill(obj);
                return;
            }
        }
 
        Destroy(obj);
    }
}
```

Since all the details are kept inside thePool class, the `PoolManager` class simply just need to find the correct pool of the provided prefab, which is quite easy as we use the prefab objects as dictionary keys. If the prefab object has not been pooled, we would simply call `Instantiate` function as usual. The deactivating process is similar, as the dictionary is scanned to find the pool that is responsible for the provided object. If there is no pool takes the responsibility, or in other words, the game object is not spawned by the pool, it would be destroyed.

Spotted the `ObjectPool`s object, the parent of all pools? That is the object where we attach the `PoolManager` component. When the scene is constructed, this object would get all the `Pool` component in its children and initialize the dictionary.

# Final touches

The PoolManager class makes things much easier for us. Now we can access the appropriatePool object conveniently, without being coupled into that concretePool instance.

Anyways, I like something more convenient. I want something that can be called easily as a part of the `GameObject` class, so that other programmers may even don’t know of the `Pool` class’s existence. Luckily, C# provides a feature called _extension methods_. With that in mind, I create a static class which serves as the extension of Unity’s `GameObject` class. This is totally a matter of personal preferences – you can safely skip this section without harming.

```csharp
public static class GameObjectExtensions
{
    public static GameObject Spawn(this GameObject prefab, Vector3 worldPosition)
    {
        return PoolManager.Spawn(prefab,
            worldPosition, Quaternion.identity, prefab.transform.localScale,
            null);
    }
 
    public static GameObject Spawn(this GameObject prefab)
    {
        return PoolManager.Spawn(prefab, 
            Vector3.zero, Quaternion.identity, prefab.transform.localScale,
            null);
    }
 
    public static GameObject Spawn(this GameObject prefab, Transform parent)
    {
        return PoolManager.Spawn(prefab,
            Vector3.zero, Quaternion.identity, prefab.transform.localScale,
            parent, true, true);
    }
 
    public static GameObject Spawn(this GameObject prefab,
        Vector3 position, Quaternion rotation, Vector3 scale,
        Transform parent = null,
        bool useLocalPosition = false, bool useLocalRotation = false)
    {
        return PoolManager.Spawn(prefab, position, rotation, scale,
            parent, useLocalPosition, useLocalRotation);
    }
 
    public static void Kill(this GameObject obj)
    {
        PoolManager.Kill(ob);
    }
}
```

The static class exposes a set of functions for activating and deactivating objects. The bodies of those functions are trivial since those functions are just wrappers for the `PoolManager`.

With the existence of this static class, I can spawn a new instance of a prefab or deactivate it in a natural way:
```csharp
// Spawn a new instance at (0, 0, 0) world position
GameObject instance = prefab.Spawn();
 
// Spawn a new instance at current object's position and take it as the parent node
GameObject instance = prefab.Spawn(this.transform);
 
// Deactivate the instance
instance.Kill();
```

#A few usage notes

The pattern’s implementation is ready to use. You just need to create a `PoolManager` object and create concrete pools as its child and you are all set. When you need to spawn those registered prefabs, you just need to call `Spawn` function as I do above. Happy spawning!

![](https://dailylolpics.com/wp-content/uploads/2017/09/animals-12.jpg)

There are some pitfalls, though. First of all, you have to reiterate your initialize functions (`Start` or `Awake`) because these functions will not be called when the object is reused the second time and beyond. I’d suggest you move all your variables for health or other states to the `OnEnable` function, which is called when the object is activated. Or even better, make an explicit function call to some custom events (`OnSpawn`) so that you can be in absolute control of that.

Also, be very wary of objects with components that rely on objects’ transforms. For example, I have a script that moves the object forward:
```csharp
public class Movement : MonoBehaviours
{
    public float moveSpeed;
    Vector3 forwardVector;
 
    void OnEnable()
    {
        forwardVector = transform.forward;
    }
 
    void Update()
    {
        transform.position += forwardVector * moveSpeed * Time.deltaTime;
    }
}
```

… and I spawn an instance of a prefab with Movement component like this
```csharp
GameObject instance = prefab.Spawn(); // Let's assume prefab object has Movement component
instance.eulerAngles = new Vector3(70f, 34f, 41f);
```
… then the rotational change will have no effect to the moving direction of the object. Due to OnEnable is called immediately after the object is activated, the forwardVector variable is set to the value of transform.forward before it’s changed. You should pay very close attention to this detail.

And that’s all for this post. I hope it is useful for you and your game. The complete source files can be found [here](https://github.com/TongTungGiang/UnityObjectPool).