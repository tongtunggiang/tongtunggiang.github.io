---
layout: post
title: "Doing indirect draw call in Unreal"
---

I've been trying to find a good guide to do indirect drawing in Unreal, but I couldn't find it. There are a few examples in the engine though, for example [Niagara](https://dev.epicgames.com/documentation/en-us/unreal-engine/creating-visual-effects-in-niagara-for-unreal-engine) has a lot of reliance on indirect draw calls in its logic and [Lidar](https://dev.epicgames.com/documentation/en-us/unreal-engine/lidar-point-cloud-plugin-overview-in-unreal-engine) is a notable mention. I've followed the examples and can get indirect draw calls working, so here's a quick summary of how to do it in Unreal.

This tutorial assumes some knowledge about the rendering framework in Unreal, especially the [mesh drawing pipeline](https://dev.epicgames.com/documentation/en-us/unreal-engine/mesh-drawing-pipeline-in-unreal-engine) and the [component-proxy relationship](https://dev.epicgames.com/documentation/en-us/unreal-engine/components-in-unreal-engine#sceneproxy). Knowing how to [create a global shader](https://dev.epicgames.com/documentation/en-us/unreal-engine/adding-global-shaders-to-unreal-engine) is also helpful for this demonstration in particular.

> The full snippet can be found [here](https://gist.github.com/tongtunggiang/2d3d737248a2b2dafcc7c1df136585af). ~~I've tested it roughly on 5.7~~ The code quality is absolutely excellent™ and you should totally use this code completely unedited when you need to ship tomorrow 🤌.

To start with, I have a specialised actor type containing a component, nothing fancy here.
```cpp
UCLASS()
class UIndirectComponent : public UPrimitiveComponent
{
    ...
	virtual FPrimitiveSceneProxy* CreateSceneProxy() override;
    ...
};

UCLASS(Placeable)
class AIndirectActor : public AActor
{
    ...
public:
	UPROPERTY()
	UIndirectComponent* IndirectComponent;
};
```

The only thing that perhaps worth mentioning so far is that the component is a `UPrimitiveComponent` with a custom scene proxy, and to ensure that it is included correctly in the scene I'll just add some absurdly big bound for it
```cpp
class FIndirectSceneProxy final : public FPrimitiveSceneProxy
{
    ...
}

FPrimitiveSceneProxy* UIndirectComponent::CreateSceneProxy()
{
	return new FIndirectSceneProxy(this);
}

FBoxSphereBounds UIndirectComponent::CalcBounds(const FTransform& BoundTransform) const
{
	// Some fake very big bounds
	FBox Box(FVector(0,0,0), FVector(1000, 1000, 1000));
	return FBoxSphereBounds(Box);
}
```

Referring to the mesh draw pipeline, the scene proxy creates the mesh batch(es), and the key thing to do here is that when creating the mesh batch, the mesh batch doesn't contain any primitives (i.e. `NumPrimitives == 0`), and have an indirect buffer that its pointed to. 
```cpp
class FIndirectSceneProxy final : public FPrimitiveSceneProxy
{
    ...
	virtual void GetDynamicMeshElements(const TArray<const FSceneView*>& Views, const FSceneViewFamily& ViewFamily, uint32 VisibilityMap, class FMeshElementCollector& Collector) const
	{
		for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
		{
			if (FMeshBatch* MeshBatch = CreateMeshBatch(Collector))
			{
				Collector.AddMesh(ViewIndex, *MeshBatch);
			}
		}
	}
	FMeshBatch* CreateMeshBatch(class FMeshElementCollector& Collector) const
	{
		if (/* something isn't right*/)
		{
			return nullptr;
		}
		FMeshBatch& MeshBatch = Collector.AllocateMesh();
        ...
		FMeshBatchElement& MeshBatchElement = MeshBatch.Elements[0];
        ...
		MeshBatchElement.NumPrimitives = 0;
		MeshBatchElement.IndirectArgsBuffer = IndirectArgsBuffer;
		MeshBatchElement.IndirectArgsOffset = 0;
		return &MeshBatch;
	}
    ...
}
```

Setting `NumPrimitive` to 0 and attach an indirect buffer makes this part of the rendering code in `FMeshDrawCommand::SubmitDrawEnd` (MeshPassProcessor.cpp) get called eventually - I won't include details herehough as it would need to get through a few layers of abstractions.
```cpp
if (MeshDrawCommand.NumPrimitives > 0 && !bDoOverrideArgs)
{
    ...
}
else
{
    RHICmdList.DrawIndexedPrimitiveIndirect(
        MeshDrawCommand.IndexBuffer,
        bDoOverrideArgs ? SceneArgs.IndirectArgsBuffer : MeshDrawCommand.IndirectArgs.Buffer,
        bDoOverrideArgs ? SceneArgs.IndirectArgsByteOffset : MeshDrawCommand.IndirectArgs.Offset
    );
}
```

That's the core idea. The indirect buffer can be of type `FRHIDrawIndirectParameters` or `FRHIDrawIndexedIndirectParameters` (see RHI.h), depending on what kind of draw you want to do. You are also responsible for managing the life time of this indirect draw buffer and populating its values. Here I'm keeping it simple and just allocate the buffer when the scene proxy allocates its render thread resources. I'll have a compute pass that writes to this buffer later.
```cpp
void FIndirectSceneProxy::CreateRenderThreadResources(FRHICommandListBase& RHICmdList)
{
    FRHIBufferCreateDesc IndirectBufferCreateDesc =
        FRHIBufferCreateDesc::CreateVertex<uint32>(
            TEXT("CustomIndirectArgs"), uint32(sizeof(FRHIDrawIndirectParameters) / sizeof(uint32))
        )
        .AddUsage(EBufferUsageFlags::UnorderedAccess | EBufferUsageFlags::DrawIndirect)
        .SetInitialState(ERHIAccess::IndirectArgs);
    IndirectArgsBuffer = RHICmdList.CreateBuffer(IndirectBufferCreateDesc);
    IndirectArgsBufferUAV = RHICmdList.CreateUnorderedAccessView(
        IndirectArgsBuffer,
        FRHIViewDesc::CreateBufferUAV()
        .SetTypeFromBuffer(IndirectArgsBuffer)
        .SetFormat(PF_R32_UINT)
        .SetNumElements(uint32(sizeof(FRHIDrawIndirectParameters) / sizeof(uint32)))
    );
}
```

From here you can do all sort of funky stuff, for example generating the vertex buffers on the GPU then sneak in some GPU side culling logic.

Here I have a very simple shader that just fill in the pre-allocated vertex buffer with the positions of a single triangle, then draw only one instance of it.
```hlsl
OutInstanceVertices[0] = 0.0f;
OutInstanceVertices[1] = 1.0f;
OutInstanceVertices[2] = 0.0f;
OutInstanceVertices[3] = 0.0f;
OutInstanceVertices[4] = -1.0f;
OutInstanceVertices[5] = 0.0f;
OutInstanceVertices[6] = 0.0f;
OutInstanceVertices[7] = 1.0f;
OutInstanceVertices[8] = 0.0f;

OutIndirectArgs[0] = 3; // VertexCountPerInstance
OutIndirectArgs[1] = 1; // InstanceCount
OutIndirectArgs[2] = 0; // StartVertexLocation
OutIndirectArgs[3] = 0; // StartInstanceLocation
```

Below is the C++ counterpart of the global shader. If you're going down the route of global shader make sure to checkout Unreal's guide as there are a couple of gotchas such as module loading phase needs to be set to `PostConfigInit` or the shader path needs to be registered if you decide to put the shader into a plugin etc. In this example I just put it into the game folder.
```cpp
class FPopulateVertexAndIndirectBufferCS : public FGlobalShader
{
public:
	static constexpr uint32 ThreadGroupSize = 64;

	DECLARE_GLOBAL_SHADER(FPopulateVertexAndIndirectBufferCS);
	SHADER_USE_PARAMETER_STRUCT(FPopulateVertexAndIndirectBufferCS, FGlobalShader);

	BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
		SHADER_PARAMETER_UAV(RWBuffer<float>, OutInstanceVertices)
		SHADER_PARAMETER_UAV(RWBuffer<uint>, OutIndirectArgs)
	END_SHADER_PARAMETER_STRUCT()
};

IMPLEMENT_GLOBAL_SHADER(FPopulateVertexAndIndirectBufferCS, "/Project/Private/PopulateVertexAndIndirectBuffer.usf", "MainCS", SF_Compute);
```

Since I want to write into the vertex buffer on the GPU, I also need vertex buffers with UAVs, and a custom initialisation flow for the vertex factory to actually ties it into the scene proxy. The shader needs to be invoked somehow, so I invoked them in a global [scene view extension](https://dev.epicgames.com/community/learning/knowledge-base/0ql6/unreal-engine-using-sceneviewextension-to-extend-the-rendering-system). Including the code for these parts here feels a bit out of scope, so please refer to the full snippet. Anyway here's the required results: we have a triangle that is drawn indirectly using the default WorldGridMaterial:
![](https://raw.githubusercontent.com/tongtunggiang/tongtunggiang.github.io/master/assets/images/indirect (1).png)

... which we can verify using a graphics debugger:
![](https://raw.githubusercontent.com/tongtunggiang/tongtunggiang.github.io/master/assets/images/indirect (2).png)
![](https://raw.githubusercontent.com/tongtunggiang/tongtunggiang.github.io/master/assets/images/indirect (3).png)


That's it. While the core idea is very simple: "Have the mesh batch containing no primitives, then Unreal does the rest", but the devil is always in the details which unfortunately are dependent on your project's needs. How are you going to populate the related buffers: create them on the CPU then upload to the GPU, or do you want to calculate them on the GPU on the fly? Do you really need an actor or even a component or can you get away with creating the proxy directly instead, like what the [Fast Geo Streaming plugin](https://portal.productboard.com/epicgames/1-unreal-engine-public-roadmap/c/2022-fast-geometry-streaming-plugin-experimental-) does? But hopefully I helped you to solve one small part of the problem which is how to invoke the indirect draw call.