---
layout:     post
title:      "Performance of Built-in Shaders"
subtitle:   "\"Unity内置着色器的性能\""
date:       2019-11-11
author:     "Ciel"
header-img: "img/post-bg-unity.jpg"
catalog: true
mathjax: true
tags:
    - Unity
---

#  Unity内置着色器的性能 

> 项目中，除了自定义shader也常常用内置shader。同时内置shader代码也有很好的参考价值。因此有必要了解内置着色器的性能表现。这里将[官方的说明](https://docs.unity3d.com/Manual/shader-Performance.html)翻译了一下。

## 性能考虑

着色器的性能主要取决于两个方面:着色器本身以及项目或特定摄像机使用的渲染路径(Rendering Path)。

### 渲染路径和着色器性能

从Unity支持的渲染路径来看，延迟着色(Deferred Shading)和顶点光照(Vertex Lit)路径具有最可预测的性能。在延迟着色中，每个对象通常只绘制一次，不管什么灯光影响它。类似地，在顶点光照中，每个对象通常只绘制一次。因此，着色器的性能差异主要取决于它们使用了多少纹理以及它们进行了哪些计算。

### 前向渲染中着色器性能

在前向渲染(Forward rendering)路径中，着色器的性能取决于着色器本身和场景中的灯光。下面的部分将解释细节。从性能的角度来看，有两种基本的着色器，顶点光照(Vertex-Lit)和像素光照(Pixel-Lit)。

在Forward rendering path中，顶点光照的着色器总是比像素光照的着色器消耗低。这些着色器通过计算基于网格顶点的光照来工作，同时使用所有的光照。正因为如此，无论物体上有多少光照射，它都只需要绘制一次。

像素光照的着色器在绘制的每个像素处计算最终照明。因此，对象必须绘制一次才能获得环境光和主方向灯，每增加一盏灯多一次。因此公式为N次渲染，其中N为最终照射在物体上的像素光的数量。这增加了CPU处理和向显卡发送命令的负载，以及处理顶点和绘制像素的负载。屏幕上像素发光物体的大小也会影响它的绘制速度。物体越大，就绘制越慢。

因此像素光照的着色器是有性能代价的，但是这种代价允许一些很好的效果，比如:阴影、法线映射、好看的高光和灯光cookies。

请记住，灯光可以被强制设置为像素(important)模式或顶点/SH(not important)模式。照在像素光照着色器上的任何顶点光将基于对象的顶点或整个对象进行计算，并且不会增加与像素光关联的渲染成本或视觉效果。

##  一般的着色器性能 

除了内置的着色器(Legacy Shaders)，它们的复杂度大致是这样的:

- Unlit：这只是一个纹理，不受任何光照的影响。
- VertexLit：顶点光照
- Diffuse：漫反射
- Normal mapped：这比漫反射要消耗高一点，它增加了一个纹理(法线贴图)和一些着色指令。
- Specular：漫反射增加了高光计算
- Normal Mapped Specular：漫反射增加了法线和高光计算，比Specular消耗高一点
- Parallax Normal mapped：漫反射增加了视差法线映射计算
- Parallax Normal Mapped Specular：漫反射增加了视差法线映射和高光计算

## 移动简化着色器性能

此外，Unity在Mobile类别下有几个针对移动平台的简化着色器。这些着色器也可以在其他平台上工作，所以如果你能接受它们的简化(例如近似镜面approximate specular，不支持材质的不同颜色等)，可以试着使用它们。

要查看每个着色器的具体简化，请查看来自内置着色器包的.shader文件，这些信息列在文件的顶部注释中。

Mobile Shader常见的改动如下：

- 没有材质颜色或主要颜色为着色器着色。
- 对于采用法线贴图的着色器，使用基础纹理的平铺tiling和偏移offset。
- 粒子着色器不支持AlphaTest或ColorMask。
- 有限的功能和照明支持。例如，一些着色器只支持一个方向的光。