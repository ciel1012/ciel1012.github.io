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

### CG头文件

- UnityStandardConfig.cginc：用于存放标准着色器配置相关的代码

- UnityStandardCore.cginc：用于存放标准着色器的主要代码（如顶点着色函数、片段着色函数等相关函数）

- UnityStandardInput.cginc：用于存放标准着色器输入结构相关的工具函数与宏

- UnityStandardMeta.cginc：用于存放标准着色器meta通道中会用到的工具函数与宏

- UnityStandardShadow.cginc：用于存放标准着色器阴影贴图采样相关的工具函数与宏

- UnityStandardUtils.cginc：用于存放标准着色器共用的一些工具函数

- UnityStandardBRDF.cginc：用于存放标准着色器处理BRDF材质属性相关的函数与宏

### Standard Shader

standard shader中两个SubShader语义块，分别对应LOD 300和LOD150。

其中实现Forward渲染的有两个Pass。FOEWARD(LightMode = ForwadBase)针对主光源，FORWARD_DELTA(LightMode = ForewardAdd)针对其他光源。

实现两个Pass的顶点片元着色器包含在UnityStandardCoreForward.cginc文件中。

### UnityStandardCoreForward

用于存放标准着色器配置相关的代码，有两个分支，UnityStandardCoreForwardSimple.cginc中是简化实现；UnityStandardCore.cginc中是标准实现。出于学习目的，这里只分析标准实现。

### UnityStandardCore

主要分析ForwardBase。

### vertForwardBase

顶点着色器

```glsl
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

```glsl
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

```glsl
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

VertexGIForward进行顶点GI的计算：

```glsl
inline half4 VertexGIForward(VertexInput v, float3 posWorld, half3 normalWorld)
{
    half4 ambientOrLightmapUV = 0;
    // Static lightmaps 烘焙GI的实现

    #ifdef LIGHTMAP_ON
        ambientOrLightmapUV.xy = v.uv1.xy * unity_LightmapST.xy + unity_LightmapST.zw;
        ambientOrLightmapUV.zw = 0;
    // Sample light probe for Dynamic objects only (no static or dynamic lightmaps) SH(球谐函数)的计算

    #elif UNITY_SHOULD_SAMPLE_SH
        #ifdef VERTEXLIGHT_ON
            // Approximated illumination from non-important point lights
            ambientOrLightmapUV.rgb = Shade4PointLights (
                unity_4LightPosX0, unity_4LightPosY0, unity_4LightPosZ0,
                unity_LightColor[0].rgb, unity_LightColor[1].rgb, unity_LightColor[2].rgb, unity_LightColor[3].rgb,
                unity_4LightAtten0, posWorld, normalWorld);
        #endif

        ambientOrLightmapUV.rgb = ShadeSHPerVertex (normalWorld, ambientOrLightmapUV.rgb);
    #endif
    //预计算实时GI

    #ifdef DYNAMICLIGHTMAP_ON
        ambientOrLightmapUV.zw = v.uv2.xy * unity_DynamicLightmapST.xy + unity_DynamicLightmapST.zw;
    #endif

    return ambientOrLightmapUV;
}
```

这里GI有三个实现分支。首先是烘焙GI，unity_LightmapST.xy和zw分别记录了LightMap的scale和offset值。第二个分支是SH(球谐函数)的计算，SH的计算在存在GI的情况下是不进行计算的，因为lightmap中已经包含了漫反射间接环境光照。所以在没有lightmap的情况下，进行SH计算。追求效率，用于SH计算的点光源被设置为了4个，同时QualitySetting中的pixel light count也是设置为4。第三个分支是预计算实时GI的实现。

ambientOrLightmapUV在启用光照贴图时xyzw分量存储光照贴图的UV。不启用的话，rgb(xyz)分量保存SH计算的颜色。

Shade4PointLights 同时处理四盏顶点光，后面的ShadeSHPerVertex 则是计算顶点的球谐光照，这两个函数位于UnityCG.cginc。具体可参考下面两篇文章：[球谐光照（spherical harmonic lighting）解析](https://gameinstitute.qq.com/community/detail/123183)、[Unity3D ShaderLab 之 Shade4PointLights 解读](https://zhuanlan.zhihu.com/p/27842876)

### fragForwardBaseInternal

片元着色器

```glsl
half4 fragForwardBaseInternal (VertexOutputForwardBase i)
{
    UNITY_APPLY_DITHER_CROSSFADE(i.pos.xy);

    FRAGMENT_SETUP(s)

    UNITY_SETUP_INSTANCE_ID(i);
    UNITY_SETUP_STEREO_EYE_INDEX_POST_VERTEX(i);
    //计算灯光衰减

    UnityLight mainLight = MainLight ();
    UNITY_LIGHT_ATTENUATION(atten, i, s.posWorld);
    //计算环境光遮蔽和全局光照

    half occlusion = Occlusion(i.tex.xy);
    UnityGI gi = FragmentGI (s, occlusion, i.ambientOrLightmapUV, atten, mainLight);
    //计算PBS

    half4 c = UNITY_BRDF_PBS (s.diffColor, s.specColor, s.oneMinusReflectivity, s.smoothness, s.normalWorld, -s.eyeVec, gi.light, gi.indirect);
    c.rgb += Emission(i.tex.xy);
    //雾效相关计算

    UNITY_EXTRACT_FOG_FROM_EYE_VEC(i);
    UNITY_APPLY_FOG(_unity_fogCoord, c.rgb);
    return OutputForward (c, s.alpha);
}
```

这里先初始化片元设置，并设置主光源。

UnityLight结构体定义在UnityLightingCommon.cginc中，记录了灯光的颜色、方向：

```glsl
struct UnityLight
{
    half3 color;
    half3 dir;
    half  ndotl; // 弃用:Ndotl现在是动态计算的，不再存储。不要使用它。
};
```

MainLight函数定义在UnityStandardCore.cginc中:

```glsl
UnityLight MainLight ()
{
    UnityLight l;

    l.color = _LightColor0.rgb;
    l.dir = _WorldSpaceLightPos0.xyz;
    return l;
}
```

Occlusion函数定义在UnityStandardInput.cginc中:

```glsl
half Occlusion(float2 uv)
{
#if (SHADER_TARGET < 30)
    // SM20: instruction count limitation
    // SM20: simpler occlusion
    return tex2D(_OcclusionMap, uv).g;
#else
    half occ = tex2D(_OcclusionMap, uv).g;
    return LerpOneTo (occ, _OcclusionStrength);
#endif
}
```

在SM2.0下，由于指令数限制，直接从Occlusion Map中采样返回g通道；在SM3.0下，采样后多做一步LerpOneTo计算。Occlusion Map一般是一张灰度图，Unity在这里只使用了它的Green通道。_OcclusionMap和_OcclusionStrength都是Standard Shader材质面板上的参数。

LerpOneTo函数定义在UnityStandardUtils.cginc：

```glsl
half LerpOneTo(half b, half t)
{
    half oneMinusT = 1 - t;
    return oneMinusT + b * t;
}
```

LerpOneTo类似Lerp计算，返回值为1-_OcclusionStrength+occ*_OcclusionStrength。其实也就等价于Lerp(1,occ,_OcclusionStrength)。

FragmentGI函数定义在UnityStandardCore.cginc中：

```glsl
inline UnityGI FragmentGI (FragmentCommonData s, half occlusion, half4 i_ambientOrLightmapUV, half atten, UnityLight light, bool reflections)
{
    UnityGIInput d;
    d.light = light;
    d.worldPos = s.posWorld;
    d.worldViewDir = -s.eyeVec;
    d.atten = atten;
    #if defined(LIGHTMAP_ON) || defined(DYNAMICLIGHTMAP_ON)
        d.ambient = 0;
        d.lightmapUV = i_ambientOrLightmapUV;
    #else
        d.ambient = i_ambientOrLightmapUV.rgb;
        d.lightmapUV = 0;
    #endif

    d.probeHDR[0] = unity_SpecCube0_HDR;
    d.probeHDR[1] = unity_SpecCube1_HDR;
    #if defined(UNITY_SPECCUBE_BLENDING) || defined(UNITY_SPECCUBE_BOX_PROJECTION)
      d.boxMin[0] = unity_SpecCube0_BoxMin; // .w holds lerp value for blending
    #endif
    #ifdef UNITY_SPECCUBE_BOX_PROJECTION
      d.boxMax[0] = unity_SpecCube0_BoxMax;
      d.probePosition[0] = unity_SpecCube0_ProbePosition;
      d.boxMax[1] = unity_SpecCube1_BoxMax;
      d.boxMin[1] = unity_SpecCube1_BoxMin;
      d.probePosition[1] = unity_SpecCube1_ProbePosition;
    #endif

    if(reflections)
    {
        Unity_GlossyEnvironmentData g = UnityGlossyEnvironmentSetup(s.smoothness, -s.eyeVec, s.normalWorld, s.specColor);
        // Replace the reflUVW if it has been compute in Vertex shader. Note: the compiler will optimize the calcul in UnityGlossyEnvironmentSetup itself
        #if UNITY_STANDARD_SIMPLE
            g.reflUVW = s.reflUVW;
        #endif

        return UnityGlobalIllumination (d, occlusion, s.normalWorld, g);
    }
    else
    {
        return UnityGlobalIllumination (d, occlusion, s.normalWorld);
    }
}
```

FragmentCommonData结构体定义在UnityStandardCore.cginc中，包含需要的一些常规数据：

```glsl
struct FragmentCommonData
{
    half3 diffColor, specColor;
    // //注意:smoothness & oneMinusReflectivity用于优化目的，主要用于DX9 SM2.0级。
    // Most of the math is being done on these (1-x) values, and that saves a few precious ALU slots.
    half oneMinusReflectivity, smoothness;
    float3 normalWorld;
    float3 eyeVec;
    half alpha;
    float3 posWorld;

#if UNITY_STANDARD_SIMPLE
    half3 reflUVW;
#endif

#if UNITY_STANDARD_SIMPLE
    half3 tangentSpaceNormal;
#endif
};
```

UnityGI结构体定义在UnityLightingCommon.cginc中，记录了一个UnityLight和一个间接光照信息的UnityIndirect。

```glsl
struct UnityGI
{
    UnityLight light;
    UnityIndirect indirect;
};
```

UnityIndirect结构体定义在UnityLightingCommon.cginc中，变量为漫反射颜色和镜面反射颜色:

```glsl
struct UnityIndirect
{
    half3 diffuse;
    half3 specular;
};
```

UnityGIInput结构体定义在UnityLightingCommon.cginc中：

```glsl
struct UnityGIInput
{
    UnityLight light; // pixel light, sent from the engine

    float3 worldPos;
    half3 worldViewDir;
    half atten;
    half3 ambient;

    // interpolated lightmap UVs are passed as full float precision data to fragment shaders
    // so lightmapUV (which is used as a tmp inside of lightmap fragment shaders) should
    // also be full float precision to avoid data loss before sampling a texture.
    //出于精度考虑使用float来避免光照贴图采样精度丢失

    float4 lightmapUV; // .xy = static lightmap UV, .zw = dynamic lightmap UV

    #if defined(UNITY_SPECCUBE_BLENDING) || defined(UNITY_SPECCUBE_BOX_PROJECTION) || defined(UNITY_ENABLE_REFLECTION_BUFFERS)
    float4 boxMin[2];
    #endif
    #ifdef UNITY_SPECCUBE_BOX_PROJECTION
    float4 boxMax[2];
    float4 probePosition[2];
    #endif
    // HDR cubemap properties, use to decompress HDR texture
    float4 probeHDR[2];
};
```

UNITY_SPECCUBE_BLENDING/UNITY_SPECCUBE_BOX_PROJECTION/UNITY_ENABLE_REFLECTION_BUFFERS定义在UnityStandardConfig.cginc中：

```glsl
// "platform caps" 定义: 它们由TierSettings控制(编辑器将确定值并将其传递给编译器)

// UNITY_SPECCUBE_BOX_PROJECTION:                   TierSettings.reflectionProbeBoxProjection
// UNITY_SPECCUBE_BLENDING:                         TierSettings.reflectionProbeBlending
// UNITY_ENABLE_DETAIL_NORMALMAP:                   TierSettings.detailNormalMap
// UNITY_USE_DITHER_MASK_FOR_ALPHABLENDED_SHADOWS:  TierSettings.semitransparentShadows
```

[TierSettings](https://docs.unity3d.com/ScriptReference/Rendering.TierSettings.html)控制启用时自动生成相关宏定义，具体参考Unity Scripting API。

FragmentGI函数主要可以分为两个部分，先是填充UnityGIInput结构体，然后计算反射，调用UnityGlobalIllumination函数。

UnityGIInput的前几个变量直接赋值即可。如果启用了静态光照贴图或者动态光照贴图，环境光为0，然后获得光照贴图的UV。否则的话，ambient直接使用VertexGIForward计算的rgb值。

然后是反射探针的相关计算，这部分计算需要涉及到一系列变量声明。

Reflection Probes定义在UnityShaderVariables.cginc中：

```glsl
UNITY_DECLARE_TEXCUBE(unity_SpecCube0);
UNITY_DECLARE_TEXCUBE_NOSAMPLER(unity_SpecCube1);

CBUFFER_START(UnityReflectionProbes)
    float4 unity_SpecCube0_BoxMax;
    float4 unity_SpecCube0_BoxMin;
    float4 unity_SpecCube0_ProbePosition;
    half4  unity_SpecCube0_HDR;

    float4 unity_SpecCube1_BoxMax;
    float4 unity_SpecCube1_BoxMin;
    float4 unity_SpecCube1_ProbePosition;
    half4  unity_SpecCube1_HDR;
CBUFFER_END
```

UNITY_DECLARE_TEXCUBE：声明了一个TextureCube类型的对象。UNITY_DECLARE_TEXCUBE_NOSAMPLER：声明了一个TextureCube类型的对象（无Sampler）。CBUFFER_START&CBUFFER_END：声明了一块常量缓冲区。以上的宏定义在HLSLSupport.cginc文件中。

常量缓冲中的对应变量保存后，如果反射为真，先计算反射环境数据（镜面反射，天空）。

Unity_GlossyEnvironmentData结构体和UnityGlossyEnvironmentSetup函数定义在UnityImageBasedLighting.cginc中：

```glsl
struct Unity_GlossyEnvironmentData
{
    // - 延迟渲染只有一个cubema
    // - 前向渲染的情况下可以有两个混合的cubemap（不常用，应该会被弃用）


    // Surface properties use for cubemap integration
    half    roughness; // 注意:这是感知粗糙度，但由于兼容性，这个名称不能更改:(
    half3   reflUVW;
};
Unity_GlossyEnvironmentData UnityGlossyEnvironmentSetup(half Smoothness, half3 worldViewDir, half3 Normal, half3 fresnel0)
{
    Unity_GlossyEnvironmentData g;

    g.roughness /* perceptualRoughness */   = SmoothnessToPerceptualRoughness(Smoothness);
    g.reflUVW   = reflect(-worldViewDir, Normal);

    return g;
}
```

UnityGlobalIllumination函数位于UnityGlobalIllumination.cginc文件中，同名的函数有四个（形参不同）。有两个函数实现是旧版本的，并在注释上说明了只是为了旧版本兼容即将被移除，所以这里只看最新的函数实现。

```glsl
inline UnityGI UnityGlobalIllumination (UnityGIInput data, half occlusion, half3 normalWorld)
{
    return UnityGI_Base(data, occlusion, normalWorld);
}

inline UnityGI UnityGlobalIllumination (UnityGIInput data, half occlusion, half3 normalWorld, Unity_GlossyEnvironmentData glossIn)
{
    UnityGI o_gi = UnityGI_Base(data, occlusion, normalWorld);
    o_gi.indirect.specular = UnityGI_IndirectSpecular(data, occlusion, glossIn);
    return o_gi;
}
```

从FragmentGI函数看到，无反射时使用UnityGI_Base直接返回。有反射时，先调用UnityGI_Base得到o_gi，然后调用UnityGI_IndirectSpecular修改o_gi.indirect.specular值。

UnityGI_Base函数位于UnityGlobalIllumination.cginc文件中:

```glsl
inline UnityGI UnityGI_Base(UnityGIInput data, half occlusion, half3 normalWorld)
{
    UnityGI o_gi;
    ResetUnityGI(o_gi);

    // Base pass with Lightmap support is responsible for handling ShadowMask / blending here for performance reason
    #if defined(HANDLE_SHADOWS_BLENDING_IN_GI)
        half bakedAtten = UnitySampleBakedOcclusion(data.lightmapUV.xy, data.worldPos);
        float zDist = dot(_WorldSpaceCameraPos - data.worldPos, UNITY_MATRIX_V[2].xyz);
        float fadeDist = UnityComputeShadowFadeDistance(data.worldPos, zDist);
        data.atten = UnityMixRealtimeAndBakedShadows(data.atten, bakedAtten, UnityComputeShadowFade(fadeDist));
    #endif

    o_gi.light = data.light;
    o_gi.light.color *= data.atten;

    #if UNITY_SHOULD_SAMPLE_SH
        o_gi.indirect.diffuse = ShadeSHPerPixel(normalWorld, data.ambient, data.worldPos);
    #endif

    #if defined(LIGHTMAP_ON)
        // Baked lightmaps
        half4 bakedColorTex = UNITY_SAMPLE_TEX2D(unity_Lightmap, data.lightmapUV.xy);
        half3 bakedColor = DecodeLightmap(bakedColorTex);

        #ifdef DIRLIGHTMAP_COMBINED
            fixed4 bakedDirTex = UNITY_SAMPLE_TEX2D_SAMPLER (unity_LightmapInd, unity_Lightmap, data.lightmapUV.xy);
            o_gi.indirect.diffuse += DecodeDirectionalLightmap (bakedColor, bakedDirTex, normalWorld);

            #if defined(LIGHTMAP_SHADOW_MIXING) && !defined(SHADOWS_SHADOWMASK) && defined(SHADOWS_SCREEN)
                ResetUnityLight(o_gi.light);
                o_gi.indirect.diffuse = SubtractMainLightWithRealtimeAttenuationFromLightmap (o_gi.indirect.diffuse, data.atten, bakedColorTex, normalWorld);
            #endif

        #else // not directional lightmap
            o_gi.indirect.diffuse += bakedColor;

            #if defined(LIGHTMAP_SHADOW_MIXING) && !defined(SHADOWS_SHADOWMASK) && defined(SHADOWS_SCREEN)
                ResetUnityLight(o_gi.light);
                o_gi.indirect.diffuse = SubtractMainLightWithRealtimeAttenuationFromLightmap(o_gi.indirect.diffuse, data.atten, bakedColorTex, normalWorld);
            #endif

        #endif
    #endif

    #ifdef DYNAMICLIGHTMAP_ON
        // Dynamic lightmaps
        fixed4 realtimeColorTex = UNITY_SAMPLE_TEX2D(unity_DynamicLightmap, data.lightmapUV.zw);
        half3 realtimeColor = DecodeRealtimeLightmap (realtimeColorTex);

        #ifdef DIRLIGHTMAP_COMBINED
            half4 realtimeDirTex = UNITY_SAMPLE_TEX2D_SAMPLER(unity_DynamicDirectionality, unity_DynamicLightmap, data.lightmapUV.zw);
            o_gi.indirect.diffuse += DecodeDirectionalLightmap (realtimeColor, realtimeDirTex, normalWorld);
        #else
            o_gi.indirect.diffuse += realtimeColor;
        #endif
    #endif

    o_gi.indirect.diffuse *= occlusion;
    return o_gi;
}
```

依次实现ShadowMas计算，SH计算（非静态GI非动态GI），静态GI，动态GI。ResetUnityGI函数位于UnityGlobalIllumination.cginc文件中，用来初始化UnityGI结构体。

UnityGI_IndirectSpecular函数位于UnityGlobalIllumination.cginc文件中:

```glsl
inline half3 UnityGI_IndirectSpecular(UnityGIInput data, half occlusion, Unity_GlossyEnvironmentData glossIn)
{
    half3 specular;

    #ifdef UNITY_SPECCUBE_BOX_PROJECTION
        // we will tweak reflUVW in glossIn directly (as we pass it to Unity_GlossyEnvironment twice for probe0 and probe1), so keep original to pass into BoxProjectedCubemapDirection
        half3 originalReflUVW = glossIn.reflUVW;
        glossIn.reflUVW = BoxProjectedCubemapDirection (originalReflUVW, data.worldPos, data.probePosition[0], data.boxMin[0], data.boxMax[0]);
    #endif

    #ifdef _GLOSSYREFLECTIONS_OFF
        specular = unity_IndirectSpecColor.rgb;
    #else
        half3 env0 = Unity_GlossyEnvironment (UNITY_PASS_TEXCUBE(unity_SpecCube0), data.probeHDR[0], glossIn);
        #ifdef UNITY_SPECCUBE_BLENDING
            const float kBlendFactor = 0.99999;
            float blendLerp = data.boxMin[0].w;
            UNITY_BRANCH
            if (blendLerp < kBlendFactor)
            {
                #ifdef UNITY_SPECCUBE_BOX_PROJECTION
                    glossIn.reflUVW = BoxProjectedCubemapDirection (originalReflUVW, data.worldPos, data.probePosition[1], data.boxMin[1], data.boxMax[1]);
                #endif

                half3 env1 = Unity_GlossyEnvironment (UNITY_PASS_TEXCUBE_SAMPLER(unity_SpecCube1,unity_SpecCube0), data.probeHDR[1], glossIn);
                specular = lerp(env1, env0, blendLerp);
            }
            else
            {
                specular = env0;
            }
        #else
            specular = env0;
        #endif
    #endif

    return specular * occlusion;
}
```

BoxProjectedCubemapDirection函数位于UnityStandardUtils.cginc中：

```glsl
inline float3 BoxProjectedCubemapDirection (float3 worldRefl, float3 worldPos, float4 cubemapCenter, float4 boxMin, float4 boxMax)
{
    // Do we have a valid reflection probe?
    UNITY_BRANCH
    if (cubemapCenter.w > 0.0)
    {
        float3 nrdir = normalize(worldRefl);

        #if 1
            float3 rbmax = (boxMax.xyz - worldPos) / nrdir;
            float3 rbmin = (boxMin.xyz - worldPos) / nrdir;

            float3 rbminmax = (nrdir > 0.0f) ? rbmax : rbmin;

        #else // Optimized version
            float3 rbmax = (boxMax.xyz - worldPos);
            float3 rbmin = (boxMin.xyz - worldPos);

            float3 select = step (float3(0,0,0), nrdir);
            float3 rbminmax = lerp (rbmax, rbmin, select);
            rbminmax /= nrdir;
        #endif

        float fa = min(min(rbminmax.x, rbminmax.y), rbminmax.z);

        worldPos -= cubemapCenter.xyz;
        worldRefl = worldPos + nrdir * fa;
    }
    return worldRefl;
}
```

Emission函数位于UnityStandardInput.cginc中，雾效相关计算位于UnityCG.cginc中。这里不过多讨论，重点放在UNITY_BRDF_PBS函数上。

### UNITY_BRDF_PBS

首先是在UnityPBSLighting.cginc中，有三种BRDF的模型实现：

```glsl
// Default BRDF to use:
#if !defined (UNITY_BRDF_PBS) // 允许在自定义着色器中显式覆盖BRDF
    // still add safe net for low shader models, otherwise we might end up with shaders failing to compile
    #if SHADER_TARGET < 30 || defined(SHADER_TARGET_SURFACE_ANALYSIS) // only need "something" for surface shader analysis pass; pick the cheap one
        #define UNITY_BRDF_PBS BRDF3_Unity_PBS
    #elif defined(UNITY_PBS_USE_BRDF3)
        #define UNITY_BRDF_PBS BRDF3_Unity_PBS
    #elif defined(UNITY_PBS_USE_BRDF2)
        #define UNITY_BRDF_PBS BRDF2_Unity_PBS
    #elif defined(UNITY_PBS_USE_BRDF1)
        #define UNITY_BRDF_PBS BRDF1_Unity_PBS
    #else
        #error something broke in auto-choosing BRDF
    #endif
#endif
```

这三种实现定义在UnityStandardBRDF.cginc中。其中BRDF3_Unity_PBS是不基于微表面的Normalized Blinn-Phong BRDF，性能消耗最小效果最差。BRDF2_Unity_PBS是基于极简主义的[CookTorrance BRDF](http://www.thetenthplanet.de/archives/255)，是为移动平台而简化的BRDF。效果最好的BRDF1_Unity_PBS是主要基于物理的BRDF。这里主要讨论BRDF1。

BRDF1借鉴迪斯尼的工作成果，基于Torrance-Sparrow微面模型，公式为：$f(l,v)=\frac{D(h)F(v,h)G(l,v,h)}{4(n\cdot l)(n\cdot v)}$

```glsl
// Main Physically Based BRDF
// Derived from Disney work and based on Torrance-Sparrow micro-facet model
//
//   BRDF = kD / pi + kS * (D * V * F) / 4
//   I = BRDF * NdotL
//
// * NDF (depending on UNITY_BRDF_GGX):
//  a) Normalized BlinnPhong
//  b) GGX
// * Smith for Visiblity term
// * Schlick approximation for Fresnel
half4 BRDF1_Unity_PBS (half3 diffColor, half3 specColor, half oneMinusReflectivity, half smoothness,
    float3 normal, float3 viewDir,
    UnityLight light, UnityIndirect gi)
{
    float perceptualRoughness = SmoothnessToPerceptualRoughness (smoothness);
    float3 halfDir = Unity_SafeNormalize (float3(light.dir) + viewDir);

// NdotV should not be negative for visible pixels, but it can happen due to perspective projection and normal mapping
// In this case normal should be modified to become valid (i.e facing camera) and not cause weird artifacts.
// but this operation adds few ALU and users may not want it. Alternative is to simply take the abs of NdotV (less correct but works too).
// Following define allow to control this. Set it to 0 if ALU is critical on your platform.
// This correction is interesting for GGX with SmithJoint visibility function because artifacts are more visible in this case due to highlight edge of rough surface
// Edit: Disable this code by default for now as it is not compatible with two sided lighting used in SpeedTree.
#define UNITY_HANDLE_CORRECTLY_NEGATIVE_NDOTV 0

#if UNITY_HANDLE_CORRECTLY_NEGATIVE_NDOTV
    // The amount we shift the normal toward the view vector is defined by the dot product.
    half shiftAmount = dot(normal, viewDir);
    normal = shiftAmount < 0.0f ? normal + viewDir * (-shiftAmount + 1e-5f) : normal;
    // A re-normalization should be applied here but as the shift is small we don't do it to save ALU.
    //normal = normalize(normal);

    float nv = saturate(dot(normal, viewDir)); // TODO: this saturate should no be necessary here
#else
    half nv = abs(dot(normal, viewDir));    // This abs allow to limit artifact
#endif

    float nl = saturate(dot(normal, light.dir));
    float nh = saturate(dot(normal, halfDir));

    half lv = saturate(dot(light.dir, viewDir));
    half lh = saturate(dot(light.dir, halfDir));

    // Diffuse term
    half diffuseTerm = DisneyDiffuse(nv, nl, lh, perceptualRoughness) * nl;

    // Specular term
    // HACK: theoretically we should divide diffuseTerm by Pi and not multiply specularTerm!
    // BUT 1) that will make shader look significantly darker than Legacy ones
    // and 2) on engine side "Non-important" lights have to be divided by Pi too in cases when they are injected into ambient SH
    float roughness = PerceptualRoughnessToRoughness(perceptualRoughness);
#if UNITY_BRDF_GGX
    // GGX with roughtness to 0 would mean no specular at all, using max(roughness, 0.002) here to match HDrenderloop roughtness remapping.
    roughness = max(roughness, 0.002);
    float V = SmithJointGGXVisibilityTerm (nl, nv, roughness);
    float D = GGXTerm (nh, roughness);
#else
    // Legacy
    half V = SmithBeckmannVisibilityTerm (nl, nv, roughness);
    half D = NDFBlinnPhongNormalizedTerm (nh, PerceptualRoughnessToSpecPower(perceptualRoughness));
#endif

    float specularTerm = V*D * UNITY_PI; // Torrance-Sparrow model, Fresnel is applied later

#   ifdef UNITY_COLORSPACE_GAMMA
        specularTerm = sqrt(max(1e-4h, specularTerm));
#   endif

    // specularTerm * nl can be NaN on Metal in some cases, use max() to make sure it's a sane value
    specularTerm = max(0, specularTerm * nl);
#if defined(_SPECULARHIGHLIGHTS_OFF)
    specularTerm = 0.0;
#endif

    // surfaceReduction = Int D(NdotH) * NdotH * Id(NdotL>0) dH = 1/(roughness^2+1)
    half surfaceReduction;
#   ifdef UNITY_COLORSPACE_GAMMA
        surfaceReduction = 1.0-0.28*roughness*perceptualRoughness;      // 1-0.28*x^3 as approximation for (1/(x^4+1))^(1/2.2) on the domain [0;1]
#   else
        surfaceReduction = 1.0 / (roughness*roughness + 1.0);           // fade \in [0.5;1]
#   endif

    // To provide true Lambert lighting, we need to be able to kill specular completely.
    specularTerm *= any(specColor) ? 1.0 : 0.0;

    half grazingTerm = saturate(smoothness + (1-oneMinusReflectivity));
    half3 color =   diffColor * (gi.diffuse + light.color * diffuseTerm)
                    + specularTerm * light.color * FresnelTerm (specColor, lh)
                    + surfaceReduction * gi.specular * FresnelLerp (specColor, grazingTerm, nv);

    return half4(color, 1);
}
```

首先看函数的输入参数

- diffColor——漫反射颜色

- specColor——镜面反射颜色

- oneMinusReflectivity——1减去反射率

- smoothness——光滑度

- normal——法线方向

- viewDir——视线方向

- light——unity中光源信息结构体，包含光照颜色color和光源方向dir

- gi——unity中间接光照信息结构体，包含漫反射颜色diffuse和镜面反射颜色specular




