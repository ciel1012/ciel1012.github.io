---
layout:     post
title:      "Unity Standard Shader"
subtitle:   "\"unity标准着色器的实现学习\""
date:       2019-08-13
author:     "Ciel"
header-img: "img/post-bg-unity.jpg"
catalog: true
mathjax: true
tags:
    - Unity
---

> 分析Unity Standard Shader实现，基于Unity2018.4.0版本的源码。

内置shader源码可以从官网进行下载。

### Standard Shader

standard shader中两个SubShader语义块，分别对应LOD 300和LOD150。

其中实现Forward渲染的有两个Pass。FOEWARD(LightMode = ForwadBase)针对主光源，FORWARD_DELTA(LightMode = ForewardAdd)针对其他光源。

实现两个Pass的顶点片元着色器包含在UnityStandardCoreForward.cginc文件中。

### UnityStandardCoreForward

用于存放标准着色器配置相关的代码，有两个分支，UnityStandardCoreForwardSimple.cginc中是简化实现；UnityStandardCore.cginc中是标准实现。出于学习目的，这里只分析标准实现。

### UnityStandardCore

主要分析ForwardBase。

#### vertForwardBase

顶点着色器

```csharp
VertexOutputForwardBase vertForwardBase (VertexInput v)
{
    UNITY_SETUP_INSTANCE_ID(v);
    VertexOutputForwardBase o;
    UNITY_INITIALIZE_OUTPUT(VertexOutputForwardBase, o);
    UNITY_TRANSFER_INSTANCE_ID(v, o);
    UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(o);
    
    //将顶点坐标从局部坐标系转换到裁剪坐标系
    float4 posWorld = mul(unity_ObjectToWorld, v.vertex);
    #if UNITY_REQUIRE_FRAG_WORLDPOS
        #if UNITY_PACK_WORLDPOS_WITH_TANGENT
            o.tangentToWorldAndPackedData[0].w = posWorld.x;
            o.tangentToWorldAndPackedData[1].w = posWorld.y;
            o.tangentToWorldAndPackedData[2].w = posWorld.z;
        #else
            o.posWorld = posWorld.xyz;
        #endif
    #endif
    o.pos = UnityObjectToClipPos(v.vertex);
    
    //纹理坐标获取
    o.tex = TexCoords(v);
    //视线方向获取
    o.eyeVec.xyz = NormalizePerVertexNormal(posWorld.xyz - _WorldSpaceCameraPos);
    //法线坐标从局部坐标系转换到世界坐标系
    float3 normalWorld = UnityObjectToWorldNormal(v.normal);
    //Tangent向量从从局部坐标系转换到世界坐标系
    #ifdef _TANGENT_TO_WORLD
        float4 tangentWorld = float4(UnityObjectToWorldDir(v.tangent.xyz), v.tangent.w);

        float3x3 tangentToWorld = CreateTangentToWorldPerVertex(normalWorld, tangentWorld.xyz, tangentWorld.w);
        o.tangentToWorldAndPackedData[0].xyz = tangentToWorld[0];
        o.tangentToWorldAndPackedData[1].xyz = tangentToWorld[1];
        o.tangentToWorldAndPackedData[2].xyz = tangentToWorld[2];
    #else
        o.tangentToWorldAndPackedData[0].xyz = 0;
        o.tangentToWorldAndPackedData[1].xyz = 0;
        o.tangentToWorldAndPackedData[2].xyz = normalWorld;
    #endif

    //We need this for shadow receving
    UNITY_TRANSFER_LIGHTING(o, v.uv1);
    //环境或光照贴图纹理坐标计算
    o.ambientOrLightmapUV = VertexGIForward(v, posWorld, normalWorld);
    //视差视线计算
    #ifdef _PARALLAXMAP
        TANGENT_SPACE_ROTATION;
        half3 viewDirForParallax = mul (rotation, ObjSpaceViewDir(v.vertex));
        o.tangentToWorldAndPackedData[0].w = viewDirForParallax.x;
        o.tangentToWorldAndPackedData[1].w = viewDirForParallax.y;
        o.tangentToWorldAndPackedData[2].w = viewDirForParallax.z;
    #endif
    //雾效相关计算
    UNITY_TRANSFER_FOG_COMBINED_WITH_EYE_VEC(o,o.pos);
    return o;
}
```

VertexOutputForwardBase为顶点着色器输出到片段着色器的结构体，位于UnityStandardCore.cginc:

```csharp
struct VertexOutputForwardBase
{
    UNITY_POSITION(pos);    //定义于HLSLSupport.cginc :  float4 pos : SV_POSITION

    float4 tex                            : TEXCOORD0;
    float4 eyeVec                         : TEXCOORD1;    // eyeVec.xyz | fogCoord
    float4 tangentToWorldAndPackedData[3] : TEXCOORD2;    // [3x3:tangentToWorld | 1x3:viewDirForParallax or worldPos]
    half4 ambientOrLightmapUV             : TEXCOORD5;    // SH or Lightmap UV
    UNITY_LIGHTING_COORDS(6,7)

    // next ones would not fit into SM2.0 limits, but they are always for SM3.0+
#if UNITY_REQUIRE_FRAG_WORLDPOS && !UNITY_PACK_WORLDPOS_WITH_TANGENT
    float3 posWorld                     : TEXCOORD8;
#endif

    UNITY_VERTEX_INPUT_INSTANCE_ID
    UNITY_VERTEX_OUTPUT_STEREO
};
```

VertexInput为顶点着色器输入结构体，位于UnityStandardInput.cginc:

```csharp
struct VertexInput
{
    float4 vertex   : POSITION;
    half3 normal    : NORMAL;
    float2 uv0      : TEXCOORD0;
    float2 uv1      : TEXCOORD1;
#if defined(DYNAMICLIGHTMAP_ON) || defined(UNITY_PASS_META)
    float2 uv2      : TEXCOORD2;
#endif
#ifdef _TANGENT_TO_WORLD
    half4 tangent   : TANGENT;
#endif
    UNITY_VERTEX_INPUT_INSTANCE_ID
};
```

VertexInput结构体包含了模型空间的顶点坐标，纹理坐标，顶点法线和切线。纹理坐标有三个，第一个是贴图的纹理坐标，第二个是静态光照UV（Bake GI），第三个是动态光照UV（Precompute Realtime GI）。



#### fragForwardBaseInternal

片元着色器

### 其他CG头文件

- UnityStandardConfig.cginc：用于存放标准着色器配置相关的代码（宏）
