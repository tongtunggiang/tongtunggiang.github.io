---
layout: post
title: "A peek at Unreal Engine 4's reflection system: Part 1 - Introduction"
---

If you are like me, then you must be confused with UE4's cryptic macros all over their C++ code. At a later point I started digging more onto the source code and trying to understand these macros' purpose. This post series aims to explain some details of the reflection system that I have discovered. I will try to explain in basic terms, however, readers' prior C++ knowledge is recommended.

### In this series
- **[Part 1: Introduction](https://tongtunggiang.com/2021/ue-reflection1/)**
- [Part 2: More UProperty](https://tongtunggiang.com/2021/ue-reflection2/)

## The macros

To start off, I'll take an example from the official Unreal Engine documentation from Epic Games. As you can see, if we strip the macros in the class definition and the *.generated.h inclusion and ignore the weird class names, this class looks pretty standard. 

```cpp
// MyActor.h
#include "GameFramework/Actor.h"
#include "MyActor.generated.h"

UCLASS()
class AMyActor : public AActor
{
    GENERATED_BODY()

public:
    AMyActor();
    virtual void Tick( float DeltaSeconds ) override;

protected:
    virtual void BeginPlay() override;

private:
    UPROPERTY()
    int32 Something;
};
```

Let's jump into the macros, `UCLASS`, `UPROPERTY and` `GENERATED_BODY`, which leads us to ObjectMacros.h. These macros do absolutely nothing on the C++ side according to the comment. They're 'metadata' for the [Unreal Header Tool](https://docs.unrealengine.com/en-US/ProductionPipelines/BuildTools/UnrealHeaderTool/index.html) to generate additional C++ code. The `GENERATED_BODY` macro can be expanded to `MyActor_h_12_GENERATED_BODY`, with `MyActor_h` being the file ID(*) and `12` is the line number in which the class is declared - this can be handy for the next section.

```cpp
// ObjectMacros.h
// These macros wrap metadata parsed by the Unreal Header Tool, and are otherwise
// ignored when code containing them is compiled by the C++ compiler
#define UPROPERTY(...)
#define UCLASS(...)

#define BODY_MACRO_COMBINE_INNER(A,B,C,D) A##B##C##D
#define BODY_MACRO_COMBINE(A,B,C,D) BODY_MACRO_COMBINE_INNER(A,B,C,D)
#define GENERATED_BODY(...) BODY_MACRO_COMBINE(CURRENT_FILE_ID,_,__LINE__,_GENERATED_BODY);
```


(*) Simplified symbol, in real life it'll have the value of the relative position in the project directory. For example, the macro will expand to `MyProject_Source_MyProject_MyActor_h` if you have this default folder structure

```
MyProject
|- Source
    |- MyProject
        |- MyActor.h
|- MyProject.uproject
```

## The generated file
Let's look at the generated file, then. Note that it's not available if you don't have any intermediate file in the project, for example when you just perform a clean. Otherwise you'll find it it *Intermediate/Build/* folder. Personally I'll just use the IDE's Go to Explorer option.

```cpp
// AMyActor.generated.h
...
#define MyActor_h_12_INCLASS_NO_PURE_DECLS \
private: \
	static void StaticRegisterNativesAMyActor(); \
	friend struct Z_Construct_UClass_AMyActor_Statics; \
public:  \
	DECLARE_CLASS(AMyActor,...) \
...
#define MyActor_h_12_GENERATED_BODY \
public: \
	MyActor_h_12_INCLASS_NO_PURE_DECLS \
private: \
...
template<> UClass* StaticClass<class AMyActor>();
...
```

I've excluded many things in the generated file in order to keep the post concise. We already knew the `GENERATED_BODY` in the MyActor class body shall be replaced with `MyActor_12_GENERATED_BODY`. If we expand it all the way to what we saw in the generated file, we will have this rough result - without any UE4 macros. This should be how the C++ compiler sees our class once UHT finished its job.

```cpp
// AActor.h

// Part of MyActor.generated.h
template<> UClass* StaticClass<class AMyActor>()
{
    return AMyActor::StaticClass();
}
// End part of MyActor.generated.h

class AMyActor : public AActor
{
    // Part of GENERATED_BODY
private:
	static void StaticRegisterNativesAMyActor();
	friend struct Z_Construct_UClass_AMyActor_Statics;
    static UClass* StaticClass();
    // End part of GENERATED_BODY

public:
    AMyActor();
    virtual void Tick( float DeltaSeconds ) override;

protected:
    virtual void BeginPlay() override;

private:
    int32 Something;
};
```
You'll see that there is a `static UCLass* StaticClass()`, as part of the `DECLARE_CLASS` macro in the generated header's content.

## Usage

Now we have a way to globally fetch the class information at runtime, either using the static function of the class or using a global function template instance:

```cpp
// using static function
UClass* Class = AMyActor::StaticClass();
// or
// using global function template instance
UClass* Class = StaticClass<AMyActor>();
```

... which we can use to search for a variable or run a function with a string representings the name of the element. For example, you can find the iterating properties snippet [an Unreal blog](https://www.unrealengine.com/en-US/blog/unreal-property-system-reflection):

```cpp
for (TFieldIterator<UProperty> PropIt(Class); PropIt; ++PropIt)
{
    UProperty* Property = *PropIt;
    // Do something
}
```

## Conclusion
- UHT and the complex macro system did all the heavy lifting for the boiler plate code for reflection.
- You can remove the macros if you don't wish to use reflection.
- Runtime type information can be accessed with `AType::StaticClass`.

In the next post, I'll jump into how UE4 handle the properties in the C++ side. Stay tuned!
