# UnityShader 变种的控制

Unity Shader 有时候变种用多了 会出现到了游戏里面Shader文件内存太大，爆炸的情况。

有时候我们的 **#pragma shader_feature**  或者 **#pragma multi_compile**命令手动添加上去，这个比较直观看出来。

还有一种情况是，我们使用内建的快捷宏引入的比如

**#pragma multi_compile_fwdbase**

**#pragma multi_compile_shadowcaster**

**#pragma multi_compile_fwdadd **

**#pragma  multi_compile_fwdadd_fullshad** 

**#pragma multi_compile_fog** 

Unity 提供很多个 multi_compile，但是官方文档说明不是很多。很多Shader里面 官方文档也没有说明。

具体的添加变种可以添加一个见得Shader，加入这个命令。

Shader->属性栏->compile and show code 下拉 下面有个variants include show，就可以看到多少变种添加进去了。

```C#
// Total snippets: 1
// -----------------------------------------
// Snippet #0 platforms ffffffff:

1 keyword variants used in scene:

<no keywords defined>

```

默认情况添加一个Shader 是没有任何变种添加进去的。

我们添加一个

```c#
#pragma multi_compile_fwdbase
```

再看变种

```c#
// Total snippets: 1
// -----------------------------------------
// Snippet #0 platforms ffffffff:
Builtin keywords used: DIRECTIONAL DIRLIGHTMAP_COMBINED DYNAMICLIGHTMAP_ON LIGHTMAP_ON LIGHTMAP_SHADOW_MIXING LIGHTPROBE_SH SHADOWS_SCREEN SHADOWS_SHADOWMASK VERTEXLIGHT_ON

8 keyword variants used in scene:

DIRECTIONAL
DIRECTIONAL LIGHTPROBE_SH
DIRECTIONAL SHADOWS_SCREEN
DIRECTIONAL LIGHTPROBE_SH SHADOWS_SCREEN
DIRECTIONAL VERTEXLIGHT_ON
DIRECTIONAL LIGHTPROBE_SH VERTEXLIGHT_ON
DIRECTIONAL SHADOWS_SCREEN VERTEXLIGHT_ON
DIRECTIONAL LIGHTPROBE_SH SHADOWS_SCREEN VERTEXLIGHT_ON
```

就发现有这么多种变种，被引入到Shader里面。



如果知道某种效果是不需要用的。使用

```C#
 #pragma skip_variants VERTEXLIGHT_ON
```



再看下变种数量

```
// Total snippets: 1
// -----------------------------------------
// Snippet #0 platforms ffffffff:
Builtin keywords used: DIRECTIONAL DIRLIGHTMAP_COMBINED DYNAMICLIGHTMAP_ON LIGHTMAP_ON LIGHTMAP_SHADOW_MIXING LIGHTPROBE_SH SHADOWS_SCREEN SHADOWS_SHADOWMASK

4 keyword variants used in scene:

DIRECTIONAL
DIRECTIONAL LIGHTPROBE_SH
DIRECTIONAL SHADOWS_SCREEN
DIRECTIONAL LIGHTPROBE_SH SHADOWS_SCREEN
```

