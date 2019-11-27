# Unity CommandBuffer 简单使用。

内置渲染管线固定很多流程。最近处理一个阴影问题。看闪耀暖暖的流程，是自己定义一个阴影的流程。自己渲染**Camera Space**  + **LightSpace ShadowMap**。他们想法是可以调整 LightSpace ShadowMap Pass 那个正交投影矩阵。提高ShadowMap利用率，第二可以制作半透明物体的接收阴影的问题。第二步，他们会制作一张**Camera Space Depth**。这个就是传统的ScreenSpace Shadow 原理。最后对这张屏幕空间阴影做**模糊处理**。

## 大概流程

我们做点简单的，比如取到这张ScreenSpace ShadowMap。做几次模糊，后面渲染用到材质贴图直接使用这张图片。这里就是要使用到Command Buffer这个功能。**Unity要设置Project Setting->Graphics->Cascaded Shadows 勾选，才会开启屏幕空间阴影流程。**



## Command BufferOverview

官方文档

 [file:///H:/Program%20Files/Unity/Editor/2018.4.7f1/Editor/Data/Documentation/en/Manual/GraphicsCommandBuffers.html](file:///H:/Program Files/Unity/Editor/2018.4.7f1/Editor/Data/Documentation/en/Manual/GraphicsCommandBuffers.html) 

大概意思 Command buffer 就是一个渲染队列。可以插入Camera Rendering 渲染队列   [Camera.AddCommandBuffer](https://docs.unity3d.com/2018.4/Documentation/ScriptReference/Camera.AddCommandBuffer.html) ，插入点可以通过 [CameraEvent enum](https://docs.unity3d.com/2018.4/Documentation/ScriptReference/Rendering.CameraEvent.html). 枚举。 除了Camera Rendering。还有Light Rendering， [Light.AddCommandBuffer](https://docs.unity3d.com/2018.4/Documentation/ScriptReference/Light.AddCommandBuffer.html)  它的插入点枚举是 [Rendering.LightEvent](https://docs.unity3d.com/2018.4/Documentation/ScriptReference/Rendering.LightEvent.html)  。这里 大概说明如果是渲染部分。可以通过[CameraEvent enum](https://docs.unity3d.com/2018.4/Documentation/ScriptReference/Rendering.CameraEvent.html).比如要插入在不透明物体渲染完，可以使用 [AfterForwardOpaque](https://docs.unity3d.com/2018.4/Documentation/ScriptReference/Rendering.CameraEvent.AfterForwardOpaque.html) 值。



## 具体流程

第一步取到ScreenSpace ShadowMap。新建一个插入在合成屏幕空间完毕。

```c#
 var ScreenSpaceShadowMapCm = new CommandBuffer();
 ScreenSpaceShadowMapCm.name = "Grab screen ShadowMap";

 mainLight.AddCommandBuffer(LightEvent.AfterScreenspaceMask, ScreenSpaceShadowMapCm);
```



第二步 我们要取出这张图ScreenSpace ShadowMap,Unity 没有直接方法可以取得RenderTexture对象。但是有一个内置纹理描述对象**BuiltinRenderTextureType**

我们插入在LightEvent.AfterScreenspaceMask后面，当前的RenderTexture，就是我们要的ScreenSpace ShadowMap。



```c#
 int screenCopyID = Shader.PropertyToID("_ScreenCopyTexture");
    ScreenSpaceShadowMapCm.GetTemporaryRT(screenCopyID, -1, -1, 0, FilterMode.Bilinear);
    ScreenSpaceShadowMapCm.Blit(BuiltinRenderTextureType.CurrentActive, screenCopyID);
```
通过Blit可以拷贝出来。对这张图片做各种处理。也直接给这张图片取个别名。在后面Shader引用

```C#
 ScreenSpaceShadowMapCm.SetGlobalTexture("ScreenCopyTexture", BuiltinRenderTextureType.CurrentActive);
```



FrameDebuger 看下

![](C:\Users\wubaoqing\Desktop\QQ截图20191126133653.png)

在Shadows.CollectShadows 后面插入了。我们想要的渲染效果

![](C:\Users\wubaoqing\Desktop\QQ截图20191126133946.png)



而且我们也拷贝到了，想到的ScreenSpace ShadowMap。后面就正常的模糊流程。



Command Buffer 里面还可以插入单个MeshRender的渲染。具体API都有说明。