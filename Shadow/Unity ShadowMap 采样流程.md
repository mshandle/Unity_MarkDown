# Unity ShadowMap 采样流程

[TOC]

Unity里面表面计算物体表面阴影。总的分成三部分。

两大类是 全局光照提供的阴影，第二部分实时计算部分，最后一类是混合阴影。

Unity提供很多宏，里面各种穿插其实不好理解。

我们这块内容最后集中在 实时计算部分和混合阴影部分。后面有个非常重要的宏**SHADOWS_SCREEN**

Unity通过这个判断是否要实时计算的部分，比如灯光设置成 **Real Time** 或者 **MIX** 都会开启这个宏。如果Bake的话这个宏就不会被定义。所以我们这次讨论的内容都集中在**SHADOWS_SCREEN** 这个宏被定义的情况



## 顶点结构体

第一步要描述V2f 顶点着色器到像素器结构体。

```C#
   struct v2f
  {
       float2 uv : TEXCOORD0;
       SHADOW_COORDS(2)
       UNITY_FOG_COORDS(3)
       float4 pos : SV_POSITION;
  };
```



Unity 提供好几个宏

- **SHADOW_COORDS**
- **UNITY_SHADOW_COORDS**
- **UNITY_LIGHTING_COORDS**
- **LIGHTING_COORDS**



### SHADOW_COORD

 先从最简单的 开始

```C#
// ---- Screen space direction light shadows helpers (any version)
#if defined (SHADOWS_SCREEN)

    #if defined(UNITY_NO_SCREENSPACE_SHADOWS)
		//TODO:
    #else // UNITY_NO_SCREENSPACE_SHADOWS
		//TODO:
    #endif

    #define SHADOW_COORDS(idx1) unityShadowCoord4 _ShadowCoord : TEXCOORD##idx1;
    #define SHADOW_ATTENUATION(a) unitySampleShadow(a._ShadowCoord)
#endif
```

文件头一开始，先判断是否使用 屏幕空间阴影，如果没有使用。就不定义。如果有就定义一个**unityShadowCoord4** 结构体，这个结构一般定义为float4

搜索全部的定义

```c#
// -----------------------------
//  Light/Shadow helpers (4.x version)
// -----------------------------
// This version computes light coordinates in the vertex shader and passes them to the fragment shader.

// ---- Spot light shadows
#if defined (SHADOWS_DEPTH) && defined (SPOT)
#define SHADOW_COORDS(idx1) unityShadowCoord4 _ShadowCoord : TEXCOORD##idx1;
#define TRANSFER_SHADOW(a) a._ShadowCoord = mul (unity_WorldToShadow[0], mul(unity_ObjectToWorld,v.vertex));
#define SHADOW_ATTENUATION(a) UnitySampleShadowmap(a._ShadowCoord)
#endif

// ---- Point light shadows
#if defined (SHADOWS_CUBE)
#define SHADOW_COORDS(idx1) unityShadowCoord3 _ShadowCoord : TEXCOORD##idx1;
#define TRANSFER_SHADOW(a) a._ShadowCoord.xyz = mul(unity_ObjectToWorld, v.vertex).xyz - _LightPositionRange.xyz;
#define SHADOW_ATTENUATION(a) UnitySampleShadowmap(a._ShadowCoord)
#define READ_SHADOW_COORDS(a) unityShadowCoord4(a._ShadowCoord.xyz, 1.0)
#endif

// ---- Shadows off
#if !defined (SHADOWS_SCREEN) && !defined (SHADOWS_DEPTH) && !defined (SHADOWS_CUBE)
#define SHADOW_COORDS(idx1)
#define TRANSFER_SHADOW(a)
#define SHADOW_ATTENUATION(a) 1.0
#define READ_SHADOW_COORDS(a) 0
#else
#ifndef READ_SHADOW_COORDS
#define READ_SHADOW_COORDS(a) a._ShadowCoord
#endif
#endif
```

```c#
SHADOWS_SCREEN
```

默认情况 SHADOWS_SCREEN 是一直存在的，它表示是否接受阴影。不管是用ScreenSpace Shadowmap还是ShadowMap。除非再Quality里面关闭阴影，这个宏才不会启动。而且Shader不添加这个宏是不能产生任何阴影效果的。通过代码也可以看的出来。

```C#
 #pragma shader_feature _ SHADOWS_SCREEN
```

```
SHADOWS_DEPTH //这个是生成深度图时候开启的
```



### UNITY_SHADOW_COORDS

UNITY_SHADOW_COORDS 它的定义比较集中，但是你会发现。拆解 不管宏定义如何，UNITY_SHADOW_COORDS 跟 SHADOW_COORDS 定义一模一样。所以这两个混合用是没问题的。

```C#
// Shadowmap helpers.
#if defined( SHADOWS_SCREEN ) && defined( LIGHTMAP_ON )
    #define HANDLE_SHADOWS_BLENDING_IN_GI 1
#endif
```



```c#
#if defined(HANDLE_SHADOWS_BLENDING_IN_GI) // handles shadows in the depths of the GI function for performance reasons
#   define UNITY_SHADOW_COORDS(idx1) SHADOW_COORDS(idx1)
#elif defined(SHADOWS_SCREEN) && !defined(LIGHTMAP_ON) && !defined(UNITY_NO_SCREENSPACE_SHADOWS) // no lightmap uv thus store screenPos instead
    // can happen if we have two directional lights. main light gets handled in GI code, but 2nd dir light can have shadow screen and mask.
    // - Disabled on ES2 because WebGL 1.0 seems to have junk in .w (even though it shouldn't)
#   if defined(SHADOWS_SHADOWMASK) && !defined(SHADER_API_GLES)
#       define UNITY_SHADOW_COORDS(idx1) unityShadowCoord4 _ShadowCoord : TEXCOORD##idx1;
#   else
#       define UNITY_SHADOW_COORDS(idx1) SHADOW_COORDS(idx1)
#   endif
#else
#   define UNITY_SHADOW_COORDS(idx1) unityShadowCoord4 _ShadowCoord : TEXCOORD##idx1;
#endif
```

Unity官方可能会代码匹配吧，统一用UNITY开头，为后续的**UNITY_TRANSFER_SHADOW** 配套。

### UNITY_LIGHTING_COORDS & LIGHTING_COORDS

最后一个比较复杂，刚刚讨论那个两种情况，都是再光源是方向光的情况下，如果是点光源或者聚光下。用刚刚两个宏效果是没效果的，如果要测试点光源的阴影我们要新建一个Pass 

```
Pass
{
	Tags{ "LightMode" = "Forwardadd" }
     CGPROGRAM
     #pragma multi_compile_fwdadd_fullshadows
	 
	 #pragma vertex vert
     #pragma fragment frag
     ENDCG
}
```

LightMode 要改成 Farwardadd 并且添加 #pragma multi_compile_fwdadd_fullshadows 添加多个变体宏。

UNITY_LIGHTING_COORDS宏本身定义比较简单

```c#
#define UNITY_LIGHTING_COORDS(idx1, idx2) DECLARE_LIGHT_COORDS(idx1) UNITY_SHADOW_COORDS(idx2)
```

跟UNITY_SHADOW_COORDS 定义多了 DECLARE_LIGHT_COORDS。

```C#
#ifdef POINT
#   define DECLARE_LIGHT_COORDS(idx) unityShadowCoord3 _LightCoord : TEXCOORD##idx;
#endif

#ifdef SPOT
#   define DECLARE_LIGHT_COORDS(idx) unityShadowCoord4 _LightCoord : TEXCOORD##idx;
#endif

#ifdef DIRECTIONAL
#   define DECLARE_LIGHT_COORDS(idx)
#endif

#ifdef POINT_COOKIE
#   define DECLARE_LIGHT_COORDS(idx) unityShadowCoord3 _LightCoord : TEXCOORD##idx;
#endif

#ifdef DIRECTIONAL_COOKIE
#   define DECLARE_LIGHT_COORDS(idx) unityShadowCoord2 _LightCoord : TEXCOORD##idx;
#endif
```



从定义可以看到，因为除了方向光以外，其他类型光源计算的时候，需要一些灯光的辅助信息。要在结构体里面，多一个字段。

```c#
struct v2f
{
	float2 uv : TEXCOORD0;
	float4 world:TEXCOORD1;
	UNITY_LIGHTING_COORDS(2,3)
	float4 pos : SV_POSITION;
	UNITY_VERTEX_OUTPUT_STEREO
};
```

LIGHTING_COORDS 定义跟UNITY_LIGHTING_COORDS 基本一模一样。



```C#
#define LIGHTING_COORDS(idx1, idx2) DECLARE_LIGHT_COORDS(idx1) SHADOW_COORDS(idx2)
```



唯一区别，这里使用的SHADOW_COORDS，从上面看到，SHADOW_COORDS跟UNITY_SHADOW_COORDS其实是一样的。



## 顶点中计算采用阴影图的UV

我们定义了结构体，后面就得在顶点着色器计算一些东西。这里面会技术路线的偏差，正常ShadowMap流程，在顶点着色器中把世界坐标变换到灯光坐标系，后面像素着色器采样阴影图。如果是ScreenMap ShadowMap，就直接计算出屏幕位置，采样阴影图就好。

所以这里面有两种变种。有一个非常重要的宏定义

```
// UNITY_NO_SCREENSPACE_SHADOWS - no screenspace cascaded shadowmaps
```

是否是屏幕空间，这个宏发现在PC或者手机平台怎么设置都不会出现在FrameDebuger KeyWord。

正常情况如果project Setting->Graphics->Cascaded Shadows 勾选就开启**SCREENSPACE_SHADOWS**，如果不勾选就会添加KeyWord **UNITY_NO_SCREENSPACE_SHADOWS** 但是用FrameDebuger 测试看不到这个KeyWord。

但是在实际代码中添加测试代码

```c#
fixed4 frag(v2f i) : SV_Target
{
#if defined(UNITY_NO_SCREENSPACE_SHADOWS)
	return fixed4(1, 0, 0, 1);
#else
	return fixed4(0, 1, 0, 1);
#endif
}
```



这个宏是有效果的。不懂为什么这样。



这里这步 就要计算采样阴影的UV。代码比较简单

```C#

v2f vert (appdata v)
{
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);
	o.uv = TRANSFORM_TEX(v.uv, _MainTex);
	TRANSFER_SHADOW(o);
	UNITY_TRANSFER_FOG(o,o.vertex);
}
```

```C#
TRANSFER_SHADOW(o);
```

这样就可以了。

Unity内建好几个宏。

- **TRANSFER_SHADOW**

- **UNITY_TRANSFER_SHADOW**

- **UNITY_TRANSFER_LIGHTING**

- **TRANSFER_VERTEX_TO_FRAGMENT**

  

### TRANSFER_SHADOW

从最简单的开始**TRANSFER_SHADOW**

搜索全部的定义

```C#
// ---- Screen space direction light shadows helpers (any version)
#if defined (SHADOWS_SCREEN)
    #if defined(UNITY_NO_SCREENSPACE_SHADOWS)
        UNITY_DECLARE_SHADOWMAP(_ShadowMapTexture);
        #define TRANSFER_SHADOW(a) a._ShadowCoord = mul( unity_WorldToShadow[0], mul( unity_ObjectToWorld, v.vertex ) );

    #else // UNITY_NO_SCREENSPACE_SHADOWS;
        #define TRANSFER_SHADOW(a) a._ShadowCoord = ComputeScreenPos(a.pos);
    #endif
#endif
```



```C#
//  Light/Shadow helpers (4.x version)
// -----------------------------
// This version computes light coordinates in the vertex shader and passes them to the fragment shader.

// ---- Spot light shadows
#if defined (SHADOWS_DEPTH) && defined (SPOT)
#define TRANSFER_SHADOW(a) a._ShadowCoord = mul (unity_WorldToShadow[0], mul(unity_ObjectToWorld,v.vertex));
#endif

// ---- Point light shadows
#if defined (SHADOWS_CUBE)
#define TRANSFER_SHADOW(a) a._ShadowCoord.xyz = mul(unity_ObjectToWorld, v.vertex).xyz - _LightPositionRange.xyz;
#endif

// ---- Shadows off
#if !defined (SHADOWS_SCREEN) && !defined (SHADOWS_DEPTH) && !defined (SHADOWS_CUBE)
#define TRANSFER_SHADOW(a)
#else

#endif
```



通过代码分析，常见的情况就是两种 。一种是传统的。

```
 #define TRANSFER_SHADOW(a) a._ShadowCoord = mul( unity_WorldToShadow[0], mul( unity_ObjectToWorld, v.vertex ) );
```

先变换到世界坐标，再变换到ShadowSpace 或者Light坐标系。 后面用tex2dproj 采样，如果这步不懂去看下ShadowMap原理。

第二种就是计算屏幕空间位置

```
#define TRANSFER_SHADOW(a) a._ShadowCoord = ComputeScreenPos(a.pos);
```

这里有个特殊情况，在屏幕空间阴影的时候，有一个潜规则的代码。

```C#
 ComputeScreenPos(a.pos);
```



这里计算用的参数，直接认为叫**pos**，默认生成的材质，vertex.如果没修改，代码里面直接用这个宏会报错。



### UNITY_TRANSFER_SHADOW



```c#
#if defined(HANDLE_SHADOWS_BLENDING_IN_GI) // handles shadows in the depths of the GI function for performance reasons
#   define UNITY_SHADOW_COORDS(idx1) SHADOW_COORDS(idx1)
#   define UNITY_TRANSFER_SHADOW(a, coord) TRANSFER_SHADOW(a)
#elif defined(SHADOWS_SCREEN) && !defined(LIGHTMAP_ON) && !defined(UNITY_NO_SCREENSPACE_SHADOWS) // no lightmap uv thus store screenPos instead
    // can happen if we have two directional lights. main light gets handled in GI code, but 2nd dir light can have shadow screen and mask.
    // - Disabled on ES2 because WebGL 1.0 seems to have junk in .w (even though it shouldn't)
#   if defined(SHADOWS_SHADOWMASK) && !defined(SHADER_API_GLES)
#       define UNITY_TRANSFER_SHADOW(a, coord) {a._ShadowCoord.xy = coord * unity_LightmapST.xy + unity_LightmapST.zw; a._ShadowCoord.zw = ComputeScreenPos(a.pos).xy;}
UnityComputeForwardShadows(a._ShadowCoord.xy, worldPos, float4(a._ShadowCoord.zw, 0.0, UNITY_SHADOW_W(a.pos.w)));
#   else
#       define UNITY_TRANSFER_SHADOW(a, coord) TRANSFER_SHADOW(a)
#   endif
#else
#   if defined(SHADOWS_SHADOWMASK)
#       define UNITY_TRANSFER_SHADOW(a, coord) a._ShadowCoord.xy = coord.xy * unity_LightmapST.xy + unity_LightmapST.zw;
#       endif
#   else
#       if !defined(UNITY_HALF_PRECISION_FRAGMENT_SHADER_REGISTERS)
#           define UNITY_TRANSFER_SHADOW(a, coord)
#       else
#           define UNITY_TRANSFER_SHADOW(a, coord) TRANSFER_SHADOW(a)
#       endif
#   endif
#endif
```

```C#
// Shadowmap helpers.
#if defined( SHADOWS_SCREEN ) && defined( LIGHTMAP_ON )
    #define HANDLE_SHADOWS_BLENDING_IN_GI 1
#endif
```

**UNITY_TRANSFER_SHADOW ** 这个跟宏分析下来，在灯光模式在MIX的时候是一样的。但是实时模式下Real-Time模式下。又分成两种情况，是否有**SHADOWS_SHADOWMASK** 如果有，要计算多计算光照贴图，如果没有的就跟正常一样。用法就比TRANSFER_SAHDOW 多传一个第二层UV。

```C#
 v2f vert(appdata v)
{
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);
	o.world = mul(unity_ObjectToWorld, v.vertex);
	o.uv = TRANSFORM_TEX(v.uv0, _MainTex);
	UNITY_TRANSFER_SHADOW(o,v.uv1);
	return o;
}
```



### UNITY_TRANSFER_LIGHTING



看定义 其实跟UNITY_TRANSFER_SHADOW差不多，只是多了计算其他除了 方向光以外灯光类型的一些坐标信息。定义比较简单。

```C#
#define UNITY_TRANSFER_LIGHTING(a, coord) COMPUTE_LIGHT_COORDS(a) UNITY_TRANSFER_SHADOW(a, coord)
```



用法其实跟UNITY_TRANSFER_SHADOW 一样

```C#
 v2f vert(appdata v)
{
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);
	o.world = mul(unity_ObjectToWorld, v.vertex);
	o.uv = TRANSFORM_TEX(v.uv0, _MainTex);
	UNITY_TRANSFER_LIGHTING(o,v.uv1);
	return o;
}
```



查看下COMPUTE_LIGHT_COORDS，发现只是计算下，顶点在灯光空间中的位置。点光源和聚光灯都有距离概念。要有位置信息，方向没有距离概念，所以定义为空。



```C#
#ifdef POINT
#   define COMPUTE_LIGHT_COORDS(a) a._LightCoord = mul(unity_WorldToLight, mul(unity_ObjectToWorld, v.vertex)).xyz;
#endif

#ifdef SPOT
#   define COMPUTE_LIGHT_COORDS(a) a._LightCoord = mul(unity_WorldToLight, mul(unity_ObjectToWorld, v.vertex));
#endif

#ifdef DIRECTIONAL
#   define COMPUTE_LIGHT_COORDS(a)
#endif

#ifdef POINT_COOKIE

#   define COMPUTE_LIGHT_COORDS(a) a._LightCoord = mul(unity_WorldToLight, mul(unity_ObjectToWorld, v.vertex)).xyz;
#endif

#ifdef DIRECTIONAL_COOKIE
#   define COMPUTE_LIGHT_COORDS(a) a._LightCoord = mul(unity_WorldToLight, mul(unity_ObjectToWorld, v.vertex)).xy;
#endif
```



### TRANSFER_VERTEX_TO_FRAGMENT

最后一个，这个也支持多种灯光类型，但是它不考虑LightMap信息。只计算实时阴影部分。其他一样的。

```C#
#define TRANSFER_VERTEX_TO_FRAGMENT(a) COMPUTE_LIGHT_COORDS(a) TRANSFER_SHADOW(a)
```

用法，不需要LightMap信息，所以也不需要UV了。

```C#
 v2f vert(appdata v)
{
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);
	o.world = mul(unity_ObjectToWorld, v.vertex);
	o.uv = TRANSFORM_TEX(v.uv0, _MainTex);
	TRANSFER_VERTEX_TO_FRAGMENT(o);
	return o;
}
```



