---
layout: post
title: Implement a simple event and delegate system in C++ with member function pointers
---

Event and delegate are a great duo. Together they would help you eliminating the headache of scrolling through branch statements, improving your code’s readability. It is so helpful that Java’s core library has an implementation for `Observer` and `Observable`. C# even takes a step further, as there is an `event` keyword baked right into the language itself. In this post I’m gonna make a simple event and delegate system in C++.

# Draft out things that we need
While developing my pet project, a platformer game, I had to make a decision about the input system: polling vs. event driven. If you have not yet known about these two definitions, do not worry because it is extremely easy to understand. Imagine you and your dad, sitting in your family’s car, is going to a park. If you keep telling your dad “Hey dad did we get there” all the way from your home to the park, you’re _polling_. If you just sit there doing nothing, and suddenly your dad tells you “Hey we get there”, that’s _event driven_.

Events are extremely helpful when it comes to processing input. Let’s assume we have a Character class:
```cpp
// Character.h
 
class Character
{
public:
    void moveHorizontal();
    void moveVertical();
    void jump();
};
```

The class provides functions for moving the character itself horizontally and vertically, as well as making it jump. So if we have respective events, we just need to bind these functions to those events, and when an event is triggered, the callback function is called immediately. With that being said, the event would contain a list of event callbacks, and a function to trigger the event manually.

# Say hello to member function pointers
> “In C, a function itself is not a variable, but it is possible to define pointers to functions, which can be assigned, placed in arrays, passed to functions, returned by functions, and so on.”
> The C Programming Language, by B. Kernighan and D. Ritchie, Second Edition, page 118

C++ takes a further step, by expanding function pointers to member functions. This is a fundamental brick for our event and delegate system, primarily with the `EventCallback` class. However, pointers to member functions and global functions are treated as two different incompatible types. Consider this function: `void func()`. If it is a global function, its type is `void (*) ()`. If it is a member function of class Foo, its type is `void (Foo::*)()` (notice the class name before the pointer mark `*`. However, `func()` is a static member function of class Foo, its type is `void (*) ()` again… (It’s crazy, I know)

So, to put it simply, if two classes `Foo` and `Bar` have a function named `func()` in their declarations, the pointers to these two functions are of two different types. That means, we need a templated class representing our event callbacks, so that we can use any member function of any class as the callback for our event. The declaration and definition of the `EventCallback` class look like this:
```cpp
// EventCallback.h
 
template<typename T>
class EventCallback : public IEventCallback
{    
public:    
	EventCallback(T* instance, void (T::*function)())
		: instance(instance), function(function) {}
 
	void operator () () { (instance->*function)(); }
 
private:
	void (T::*function)();
	T* instance;
};
```

This class acts as a _Proxy_ to the function pointer itself, it defines the operator () so that you can use the object exactly like how you call a function. More on this later.

We have a new problem here. As the templated class is copied and pasted to the place it is called with the typename parameter filled (for example, `EventCallback<Character>`), so `EventCallback<Character>` and `EventCallback<AnotherClass>` are considered two different types. We cannot store two different objects of different types to an array, which brutally violates our design above. To solve this, let’s create a common interface for event callbacks so that we don’t have care about whatever class containing the member function pointer:

```cpp
// EventCallback.h
class IEventCallback
{
public:
	virtual void operator() () = 0;
};
 
template<typename T>
class EventCallback : public IEventCallback
{    
public:    
	EventCallback(T* instance, void (T::*function)())
		: instance(instance), function(function) {}    
	virtual void operator () () override { (instance->*function)(); }    
private:
	void (T::*function)();
	T* instance;
};
```
C++, unlike other modern languages, doesn’t provide the term of interfaces, so I create an abstract class containing a pure virtual operator (), which cannot be instantiated. The templated class `EventCallback` now implements the `IEventCallback` interface.

There is one more thing that we need to add to our `EventCallback` class, though. That is the ability to compares with another `EventClass` object. This operator is the reason why I use direct member function pointers, rather than the `std::function` class provided by C++ STL.

```cpp
// EventCallback.h
class IEventCallback
{
public:
	// Other IEventCallback code
	virtual bool operator == (IEventCallback* other) = 0;
};
 
template
class EventCallback : public IEventCallback
{    
public:
	// Other EventCallback code
	virtual bool operator == (IEventCallback* other) override
	{
		EventCallback* otherEventCallback = dynamic_cast<EventCallback>(other);
		if (otherEventCallback == nullptr)
			return false;
 
		return 	(this->function == otherEventCallback->function) && 
			(this->instance == otherEventCallback->instance);
	}
};
```

The idea behind the comparison operator is simple and concise. Firstly, we perform a dynamic cast against the `other` parameter. If `this` and `other` are of two different types, which can be determined by the casting result, clearly they are not equal. Then we compare the addresses that two instance variables point to, to check whether these two pointers point to a same object in the memory. The same process goes to the two function pointers, to determine whether these two are the same member function.

# Now let there be Event!

As we have the implementation for our event callbacks, it’s about time to write the Event class. The declaration is quite straightforward. There is a default constructor and destructor for creating and destroying events. The `addListener` and `removeListener` functions are responsible for adding and removing delegates to and from the list of delegates (actions). And lastly, the class provides a fire function to trigger the event manually.

```cpp
//Event.h
 
#include "EventCallback.h"
 
class Event
{
public:
	Event();
	~Event();
 
	void addListener(IEventCallback* action);
	void removeListener(IEventCallback* action);
	void fire();
 
private:
	typedef std::vector<IEventCallback*> CallbackArray;
	CallbackArray actions;
};
```
The `actions` array contains the list of delegates that would be called when the event is triggered. I choose `std::vector` because it provides convenient mechanisms for add and removing elements. The definition of the `Event` class is as below:

```cpp
// Event.cpp
 
Event::Event() { }
Event::~Event() { }
 
void Event::addListener(IEventCallback* action)
{
	CallbackArray::iterator position = find(actions.begin(), actions.end(), action);
 
	if (position != actions.end())
	{
		Debug::LogWarning("Action existed in delegate list.");
		return;
	}
 
	actions.push_back(action);
}
 
void Event::removeListener(IEventCallback* action)
{
	CallbackArray::iterator position = find(actions.begin(), actions.end(), action);
 
	if (position == actions.end())
	{
		return;
	}
 
	actions.erase(position);
}
 
void Event::fire()
{
	for (IEventCallback* action : actions)
	{
		(*action)();
	}
}
```

# Apply to my game
I already have these classes ready: `Actor`, `InputHandler`. I create an `uselessFunction()` member for the `Actor` class, register it with the “Jump” action that `InputHandler` manages:

```cpp
Actor* actor = new Actor();
 
EventCallback* callback = new EventCallback(actor, &Actor::uselessFunction);
InputHandler::getInstance()->registerAction("Jump", callback);
```

This is the body of the `registerAction()` function, in case you’re interested:

```cpp
// InputHandler.h
#include "Event.h"
#include <map>
 
class InputHandler : public Singleton
{
public:
	void registerAction(const std::string& actionName, InputEventType type, IEventCallback* action);
 
private:
	std::map<std::string, Event*> inputEvents;
}
 
// InputHandler.cpp
void InputHandler::registerAction(const string &actionName, IEventCallback* action)
{
	if (inputEvents.count(actionName) < 1) 
	{
		Debug::LogError("There is no action named " + actionName); 
		return;
	}
	inputEvents[actionName]->addListener(action);
}
</map>
```

I bind the “Jump” action with Space key on my keyboard, and when I run my demo, voila! It reacts to my input events!

![](https://i.imgur.com/7wxcnjS.png)

# There are rooms for improvements
As you can see, my solution is very primitive, and there definitely are rooms for improvements. For example, using raw pointers for instances is quite dangerous, when the instance is already destructed at the moment of triggering the event, so that using smart pointers would be a wiser choice. I belive there are more, and if you have any suggestions, please don’t be hesitate to leave a comment below.
