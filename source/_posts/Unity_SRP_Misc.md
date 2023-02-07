---
title: Unity SRP Misc
date: 2023-01-31 10:01:00
categories:
- [Unity, SRP]
tags:
- Unity
- SRP
---

## 渲染指定物体

在SRP里面一个非常重要的函数是`ScriptableRenderContext.DrawRenderers`，通过它可以帮我们完成整个场景的渲染。但是这个函数用起来很方便，因为它封装了很多细节。但是在游戏UI中经常会用`RenderTexture`来混合UI和模型，就需要将个别物体渲染到`RenderTexture`。如果我们只想渲染某一两个物体时，操作起来就比较麻烦，因为首先我们需要将需要渲染的物体`ScriptableRenderContext.Cull`出来，这个剔除操作是针对整个场景的，必然有额外的消耗。特别是我们已经知道要渲染哪个物体的情况下，我们可能希望跳过剔除这一步。但是跳过剔除，`ScriptableRenderContext.DrawRenderers`函数就用不了了。

这时候我们可以使用`CommandBuffer.DrawRanderer`函数，针对单个物体进行渲染。具体操作就是获取所有物体的`Render`组件、材质，使用`CommandBuffer`渲染。示例代码如下：
```C#
public class RenderData{
    public Renderer renderer;
    public Material material;
    public float sortingFudge;
    public int materialIndex;
}
int Comparison(RenderData a, RenderData b){
    if(a.material.renderQueue == b.material.renderQueue){
        if(a.sortingFudge == b.sortingFudge){
            string aBlendMode = a.material.GetTag("BlendMode", false);
            string bBlendMode = b.material.GetTag("BlendMode", false);
            if(aBlendMode == bBlendMode)
                return a.materialIndex - b.materialIndex;
            else
                return aBlendMode == "AlphaBlend" ? -1 : 1;
        }
        else
        {
            return a.sortingFudge >b.sortingFudge ? -1 : 1;
        }
    }
    else
    {
        return a.material.renderQueue - b.material.renderQueue;
    }
}
List<RenderData> renderList = new List<RenderData>();
List<Renderer> tempRenders = new List<Renderer>();
void CalculateRenderData(GameObject go){
    tempRenders.Clear();
    renderList.Clear();
    go.GetComponentsInChildren<Renderer>(tempRenders);
    foreach(var render in tempRenders){
        float sortingFudge = 0;
        if(render is SkinnedMeshRenderer sRender){
            sRender.updateWhenOffscreen = true;
        }
        else if(render is ParticleSystemRenderer pRender){
            sortingFudge = pRender.sortingFudge;
        }
        var materials = render.sharedMaterials;
        int index = -1;
        foreach(var material in materials){
            index = index + 1;
            if(material == null) continue;
            RenderData renderData = new RenderData{
                renderer = render,
                material = material,
                materialIndex = index,
                sortingFudge = sortingFudge,
            };
            renderList.Add(renderData);
        }
    }
}
ScriptableRenderContex context;
Camera camera;
void RenderSingleObject(GameObject go, RenderTexture rt){
    CalculateRenderData(go);
    Camera.SetupCurrent(camera);
    CommandBuffer cmd = CommandBufferPool.Get();
    context.SetupCameraProperties(camera);
    cmd.SetRenderTarget(rt);
    cmd.ClearRenderTarget(true, true, Color.clear);
    foreach(var data in renderList){
        if(data.material.passCount > 0){
            for(int i = 0; i< data.material.passCount; ++i){
                cmd.DrawRenderer(data.renderer, data.material, data.materialIndex, i);
            }
        }
    }
    context.ExecuteCommandBuffer(cmd);
    CommandBufferPool.Release(cmd);
}
```

这里有几个点需要注意：
- 首先是`Camera.SetupCurrent(camera)`，如果没有这行代码的话，粒子系统是不会渲染的[参考](https://forum.unity.com/threads/commandbuffers-and-particle-systems.965936/)。
- `ScriptableRenderContext.DrawRenderers`支持`SRPBatch`，但是其他绘制函数不一定支持，比如这里的`CommandBuffer.DrawRanderer`,这是当前方法的一个弊端。但是Unity2022版本增加了[BatchRendererGroup](https://docs.unity3d.com/2022.1/Documentation/Manual/batch-renderer-group-how.html)据说是可以实现`SRPBatch`,参考[这里](https://forum.unity.com/threads/how-to-get-srp-batching-performance-while-using-command-buffer-drawmesh.1186063/)，但是WebGL目前还不支持。`BatchRendererGroup`的原理是在`SRPBatch`上扩展的自定义合批方法。
- [这里](https://forum.unity.com/threads/how-to-cull-gameobject-from-camera-without-using-layers.800202/)也有对剔除的探讨，但是好像并没有什么有价值的东西，暂且留个入口。
- 另外，`Animator`组件和`ParticleSystem`组件上都带有`CullingMode`属性，一般来说当物体不可见时，我们希望动画和粒子系统都停止，从而减少性能消耗。`CullingMode`就是用来控制物体不可见时的行为。当使用`ScriptableRenderContext.DrawRenderers`绘制时，会自动标记物体是否可见的状态，所以通常设置为`Automatic`,也就是说不可见时停止，可见时照常运行。但是使用`CommandBuffer.DrawRanderer`绘制时，并不会标记这些可见状态，所以需要强制设置为`Always Simulate`，当然有些粒子系统设置为`Automatic`也能正常运行，这个应该是Unity的bug。
- 当然，我非要用`ScriptableRenderContext.DrawRenderers`渲染某一个物体也不是不行，就是要把剔除流程走一遍。如果我们需要将多个物体分别渲染到多个纹理上呢？是不是就的把整个流程走多遍？有一种做法，可以只走一边剔除流程，就可以渲染多个模型到多个贴图上。`FilteringSettings.renderingLayerMask`这个参数就是可以用来控制本次渲染渲染指定渲染层的模型，我们可以为不同的模型设置不同的层，渲染多张纹理的时候把`FilteringSettings.renderingLayerMask`设置到对应的层。渲染层总共有32层，也就是最多可分32个独立渲染的物体。我之前想过渲染一次设置一次模型的渲染层，这样看起来就可以渲染无数多个独立渲染的物体，但是实际上这是行不通的，因为整个指令是`CommandBuffer`缓存执行，也就是说动态修改物体的渲染层是无效的，`CommandBuffer`指令只知道最后一次设置的渲染层。