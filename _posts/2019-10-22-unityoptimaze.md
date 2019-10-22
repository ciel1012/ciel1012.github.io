---
layout:     post
title:      "Optimizing Graphics Performance"
subtitle:   "\"Unity常见优化总结\""
date:       2019-10-22
author:     "Ciel"
header-img: "img/post-bg-unity.jpg"
catalog: true
mathjax: false
tags:
    - Unity
---

# Unity Optimizing 

## CPU：减少draw call

- 手动合并mesh或使用unity的批处理（一个mesh中有1000三角面优于1000个只有1个面的mesh）。
  - 静态合批
  - 动态合批
  - GPU Instancing

- 在物体中使用更少的材质，把不同的纹理放到一个更大的纹理图集中。

- 使用更少导致对象被渲染多次的效果(如反射，阴影和像素光)。

请注意，合并两个不同材质的的对象无法提升性能。需要多种材质的最常见原因是两个mesh没有共享相同纹理。为了优化CPU性能，请确保合并的所有对象共享相同纹理。

当在Forward rendering path中使用多个像素灯时，可能会出现合并对象没有意义的情况。请参阅下面的光照性能部分，以了解如何管理它。

## GPU：优化模型几何形状

- 尽量减少不必要的三角面

- 尽量减少UV贴图接缝和硬边（硬边处顶点数翻倍）

请注意，图形硬件必须处理的顶点的实际数目通常与3D应用程序报告的数目不同。建模应用程序通常显示组成模型的不同角点的数量(称为几何顶点数)。然而，对于一个显卡，为了渲染的目的，一些几何顶点需要被分割成两个或更多的逻辑顶点。如果一个顶点有多个法线、UV坐标或顶点颜色，则必须对其进行分割。因此，Unity中的顶点数通常高于3D应用程序给出的顶点数。

虽然模型中的几何图形数量主要与GPU相关，但Unity中的一些功能也在CPU上处理模型(例如mesh skinning)。

## GPU：纹理压缩和mipmaps

使用压缩纹理可以减小纹理的大小。这样可以缩短加载时间，减少内存占用，并显着提高渲染性能。 压缩纹理仅使用未压缩32位RGBA纹理所需的一部分内存带宽。

为3D场景中使用的纹理始终启用Generate mipmaps。mipmap纹理使GPU可以对较小的三角形使用较低分辨率的纹理，这与纹理压缩的方式类似。可以限制GPU渲染时传输的纹理数据量。如果当一个texel(纹理像素)被认为是1:1映射到渲染的屏幕像素时，就不要启用。比如UI元素或2D游戏中的纹理。

## GPU：编写高性能着色器

可以手动优化着色器以减少计算和纹理读取，以便在低端GPU机器上获得良好的性能。例如，一些内置的Unity着色器有mobile版本，它们更快，只是有一些限制。

- 数学函数(如pow、exp、log、cos、sin、tan)是非常消耗资源的函数，因此尽量避免使用它们。考虑使用查找纹理作为复杂数学计算的替代(如果适用的话)。

- 避免编写自己的操作（例如normalize，dot，inversesqrt）。Unity的内置选项可确保驱动程序可以生成更好的代码。

- Alpha Test（剔除）操作通常会使片段着色器变慢。clip()在不同的平台上具有不同的性能特征:
  - 通常在大多数平台上使用它可以删除完全透明的像素，获得一点提升。
  - 然而在iOS和一些Android设备的PowerVR GPU上，alpha test是资源密集型的。不要尝试在这些平台上使用它来进行性能优化，因为它会导致游戏运行速度比平时慢。

- 浮点数精度在PC GPU中差异很小，但是对于移动端优化非常重要。
  - 对于世界空间位置和纹理坐标，使用float。
  - 对于其他所有东西(vectors，HDR颜色等)，一般用half，只有在必要时才增加。
  - 对于非常简单的纹理数据或普通颜色值，可以使用fixed。
  - 所有PC GPU总是以float来计算所有东西，所以float/half/fixed的结果在底层是完全相同的。这可能会使测试变得困难，因为很难看到half/fixed精度是否足够，所以要在目标设备上测试以获得准确的结果。
  - Mobiles GPU有实际的半精度支持。这通常更快，而且计算时消耗更少。

  - Fixed通常只对较老的Mobile GPU有用。大多数现代GPU(那些可以运行OpenGL ES 3或Metal的GPU)内部处理fixed和half完全相同。

- 在可能的情况下，将计算从像素着色器代码移到顶点着色器代码中，或者将它们完全移出着色器并在脚本中设置值。

- Surface Shaders非常适合编写与灯光交互的着色器。然而它们的默认设置为了兼顾效果并不是性能最优的。调整这些特定的设置，可以使着色器运行更快或至少更小（去掉变种）:
  - approxview指令用于使用视图方向(即高光)的着色器，使得每个顶点的视图方向都标准化，而不是每个像素。这是近似的，但通常已经足够好了。
  - halfasview的Specular shader更快。每个顶点计算并归一化半向量(照明方向和视图向量的中间部分)，照明函数接收半向量作为参数而不是视图向量。
  - noforwardadd使着色器在前向渲染中只支持一个单向光。其余的光仍然可以作为每个顶点光或球面谐波的影响。这使着色器更小，并确保它总是在一个通道渲染，即使有多个灯。
  - noambient在着色器上禁用环境照明和球面谐光。这可以使性能稍微快一些。

- 在某些平台上(主要是iOS和Android设备上的移动GPU)，使用ColorMask去掉某些通道(如ColorMask RGB)可能会占用大量资源，因此只有在真正需要时才使用它。

## Lighting：光照性能

- 使用Lightmapping，优势是效率比实时光高，可以烘焙GI，并对lightmapper结果做平滑处理，效果更好。

可以使用简单的技巧代替添加灯光到场景中。比如在shader中计算边缘照明效果，而不是多打一盏补光。

在前向渲染中，逐像素动态照明为每个受影响像素增加了大量渲染工作，并可能导致对象被多次渲染。避免在移动或低端PC GPU上使用一个以上的Pixel Light照亮单个对象。对于静态对象应使用Lightmap预渲染，而不是每帧计算光照。每个顶点的动态光照可以为顶点转换增加大量的工作，所以尽量避免多个光照同时照射一个物体的情况。

避免合并相距足够远的网格，以使其受到不同组的像素光的影响。使用像素照明时，每个网格必须渲染的次数与照亮像素的次数相同。如果合并两个相距很远的网格，则会增加合并对象的有效尺寸。 在渲染期间会考虑所有照亮此合并对象任何部分的像素光，因此会增加需要进行的渲染遍数。通常，渲染合并对象必须经过的次数是每个单独对象的通过次数的总和，因此通过合并网格无法获得任何效果。

在渲染期间，Unity会找到网格周围的所有灯光，并计算其中哪些灯光对网格的影响最大。 "Quality"窗口上的设置用于修改最终有多少灯作为像素灯，以及多少灯为顶点灯。每盏灯根据其距网格的距离和照明强度来计算其重要性。纯粹从游戏环境来看，某些灯比其他灯更重要。 因此，每个Light都有一个"Render Mode"设置，可以将其设置为"Important"或"Not Important"。 标为"Not Important"的灯光具有较低的渲染开销。

优化每个像素的照明节省了CPU和GPU的工作:CPU有更少的绘制调用要做，GPU需要处理的顶点更少，需要对所有附加对象渲染进行栅格化的像素也更少。

## LOD和per-layer culling distances

剔除对象是一种减少CPU和GPU负载的有效方法。

在不影响玩家体验情况下，剔除远距离的小物体。有以下几种方法：

使用LOD系统（Level of Detail）

手动设置相机上的每层剔除距离（per-layer culling distances） 

使用Camera.layerCullDistances脚本功能将小物体放到单独的图层中并设置每层的剔除距离

## 实时阴影

实时阴影有相当高的渲染开销。任何投射阴影的对象首先必须被渲染到shadow map中，然后该shadow map被用来渲染可能接收阴影的对象。启用实时阴影甚至比pixel/vertex光对性能的影响更大。

Soft shadows比hard shadows有更大的渲染开销，但这只影响GPU，并不会导致更多的CPU工作。

通过设置Shadow Distance来减少必须实时渲染的阴影数量。超出这个距离的阴影根据光照设置不渲染或者使用shadowmask渲染。

## Checklist:

- 当为PC构建时(取决于目标GPU)，将顶点数保持在每帧200K和3M以下。

- 使用内置的着色器的简化和近似版本（Mobile或Unlit）。

- 尽量减少每个场景中的不同材质数量，并在不同对象之间尽可能多地共享材质。

- 在非移动对象上设置静态属性，以允许内部优化，如静态批处理。

- 只有一个(最好是directional)像素光影响场景，而不是多个。

- 使用烘焙光照而不是动态光照

- 尽可能使用压缩的纹理格式，使用16位纹理而不是32位纹理。

- 尽量避免使用雾。

- 使用遮挡剔除（occlusion culling）在复杂的静态场景中，减少可见几何图形和绘制调用的数量。

- 使用天空盒来"伪造"远处的几何图形。

- 使用像素着色器或纹理组合器混合多个纹理，而不是使用多pass方法。

- 尽可能使用half精度的变量。

- 尽量减少使用复杂的数学运算，如在像素着色器中的pow, sin和cos。

- 每个片段使用更少的纹理。


参考资料

[[https://docs.unity3d.com/Manual/OptimizingGraphicsPerformance.html]{.underline}](https://docs.unity3d.com/Manual/OptimizingGraphicsPerformance.html)

[[https://docs.unity3d.com/Manual/LightPerformance.html]{.underline}](https://docs.unity3d.com/Manual/LightPerformance.html)

[[https://docs.unity3d.com/Manual/SL-DataTypesAndPrecision.html]{.underline}](https://docs.unity3d.com/Manual/SL-DataTypesAndPrecision.html)

[[https://docs.unity3d.com/Manual/SL-ShaderPerformance.html]{.underline}](https://docs.unity3d.com/Manual/SL-ShaderPerformance.html)