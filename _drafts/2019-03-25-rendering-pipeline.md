---
layout: post
title: "The Graphics Rendering Pipeline"
subtitle: " \"《Real-Time Rendering 4th》笔记——第二章\""
date: 2019-03-25 6:00:00 
author: "Ciel"
header-img: "img/post-bg-rtr.jpg"
tags:
 - Real-Time Rendering
 - 笔记
---

# The Graphics Rendering Pipeline

> 渲染管线基本结构包括四个阶段：应用、几何处理、光栅化、像素处理。图形渲染管线的主要功能就是决定在给定虚拟相机、三维物体、光源、照明模式，以及纹理等诸多条件的情况下，生成或绘制一幅二维图形的过程。

![\img\in-post\real-time-rendering\2-1](\img\in-post\real-time-rendering\2-1.png)

1. [The Application Stage（应用程序阶段）](#theapplicationstage)

2. [Geometry Processing（几何处理阶段）](#geometryprocessing)

3. [Rasterization（光栅化阶段）](#rasterization)

4. [Pixel Processing（像素处理阶段）](#pixelprocessing)

### TheApplicationStage

- 应用程序阶段将需要绘制的图元输入到几何阶段，以及实现一些软件方式来实现的方法。主要方法有碰撞检测、加速算法、输入检测、动画、力反馈、纹理动画、变幻仿真、几何变形，以及一些不在其他阶段执行的计算，如层次视锥裁剪等加速算法。

- 对于被渲染的每一帧，应用程序阶段将摄像机位置，光照和模型的图元输出到几何阶段。

### GeometryProcessing

几何阶段主要负责大部分多边形操作和顶点操作，还可以分为以下几个阶段：

![\img\in-post\real-time-rendering\2-2](\img\in-post\real-time-rendering\2-2.png)

1. **Vertex Shading 顶点着色**

   - 顶点着色器有两个主要任务：计算顶点位置和计算程序想要的任何顶点输出数据，如法线和纹理坐标。

   - 模型的顶点和法线是通过模型变换得到的。对象的坐标称为模型坐标，当模型转换应用于这些坐标后，模型就位于世界坐标或世界空间下。当模型各自进行模型转换后，就存在于同一空间中。

   - 仅对相机或视点可以看到的模型进行绘制。相机在世界空间中有一个位置方向用来放置和校准相机。

   - 为了便于投影和裁剪，要对相机和所有模型进行视点变换。变换的目的是把相机放到原点进行视点校准，使其朝向Z轴负方向，Y轴指向上方，X轴指向右边。视点变换后，实际位置和方向依赖于当前API。称上述空间为相机空间或观察空间。

   - 确定材质上的光照效果的这种操作被称为着色（shading）。着色过程涉及在对象上的各个点处计算着色方程（shading equation）。通常，这些计算中的一些在几何阶段期间在模型的顶点上执行（vertex shading），而其他计算可以在每像素光栅化（per-pixel rasterization）期间执行。可以在每个顶点处存储各种材料数据，诸如点的位置，法线，颜色或计算着色方程所需的任何其它数字信息。顶点着色的结果（其可以是颜色，向量，纹理坐标或任何其他种类的阴着色数据）计算完成后，会被发送到光栅化阶段以进行插值操作。

   - 通常，着色计算通常认为是在世界空间中进行的。在实践中，有时需要将相关实体（诸如相机和光源）转换到一些其它空间（诸如模型或观察空间）并在那里执行计算，也可以得到正确的结果。

   【总结】顶点着色阶段的目的在于确定模型上顶点处材质的光照效果。

2. **Projection 投影变换**

   ![\img\in-post\real-time-rendering\2-3](\img\in-post\real-time-rendering\2-3.png)

   - 在光照处理之后，渲染系统就开始进行投影操作，即将视体变换到一个对角顶点分别是(-1,-1,-1)和(1,1,1)单位立方体（unit cube）内，这个单位立方体通常也被称为规范立方体（Canonical View Volume，CVV）。

   - 目前主要有两种投影方法：正交投影（orthographic projection，或称 parallel projection）、透视投影（ perspective projection）。两种投影方式的主要异同点：

     - 正交投影的可视体通常是一个矩形，可以把视体变换为单位立方体。主要特性是平行线在变换之后彼此仍然平行，这种变换是平移与缩放的组合。

     - 透视投影中，越远离相机的物体，在投影后看起来越小。更进一步来说，平行线将在地平线处会聚。透视投影的变换其实就是模拟人类感知物体的方式。

     - 正交投影和透视都可以通过4X4的矩阵来实现，在任何一种变换之后都可以认为模型位于归一化处理之后的设备系统中。

   - 虽然这些矩阵变换是从一个可视体变换到另一个，但它们仍被称为投影，因为在完成显示后，Z坐标将不会再保存于的得到的投影图片中。通过这样的投影方法，就将模型从三维空间投影到了二维的空间中。

   【总结】投影阶段就是将模型从三维空间投射到了二维空间中的过程。

3. **Clipping 裁剪**

   - 

4. **Screen Mapping 屏幕映射**

   - 

### Rasterization

1. **Triangle Setup 三角形设置**

   - 

2. **Triangle Traversal 三角形遍历**

   - 

### PixelProcessing

1. **Pixel Shading 像素着色**

2. **Merging 合并**
