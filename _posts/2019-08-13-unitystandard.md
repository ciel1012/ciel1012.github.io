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

# Unity Standard Shader的实现学习

> 分析Unity Standard Shader实现，基于Unity2018.4.0版本的源码。

内置shader源码可以从官网进行下载。



### Standard Shader

standard shader中两个SubShader语义块，分别对应LOD 300和LOD150。

其中实现Forward渲染的有两个Pass。FOEWARD(LightMode = ForwadBase)针对主光源，FORWARD_DELTA(LightMode = ForewardAdd)针对其他光源。

实现两个Pass的顶点片元着色器包含在UnityStandardCoreForward.cginc文件中。

### UnityStandardCoreForward

用于存放标准着色器配置相关的代码，有两个分支，UnityStandardCoreForwardSimple.cginc中是简化实现；UnityStandardCore.cginc中是标准实现。

### 其他CG头文件

- UnityStandardConfig.cginc：用于存放标准着色器配置相关的代码（宏）


