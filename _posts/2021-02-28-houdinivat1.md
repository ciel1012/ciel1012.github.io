---
layout:     post
title:      "[Houdini] VAT Base"
subtitle:   "\"Houdini Vertex Animation Texture烘焙及使用方法\""
date:       2021-02-28
author:     "Ciel"
header-img: "img/post-bg-houdini.jpg"
catalog: true
mathjax: true
tags:
    - Houdini
---

# Houdini Vertex Animation Texture

> Houdini的SideFXLabs中提供了一个将模型顶点动画数据烘焙到贴图的工具，通过它可以很方便的将模拟计算出来的动画导入到引擎中

参考教程链接：

https://www.youtube.com/watch?v=khCfJdg2pyw

使用的软件版本和测试效果：

Houdini：18.5.351，unity：2019.4.16，unreal：4.26

![](/img/in-post/houdini-vat/houdini-unity-vat.gif)

![](/img/in-post/houdini-vat/houdini-unreal-vat.gif)

### 基本步骤

1. 创建模拟计算的sop，生成变化的mesh
2. 使用Labs中的vertex animation texture节点工具烘焙
3. 资源导入到引擎
4. 设置材质shader及参数

步骤其实很简单，只是其中还有一些细节需要注意

### Houdini中创建模型动画

这一步有很多方法，VAT主要用于烘焙houdini中各种模拟计算出来的动画，比如刚体破碎，液体，布料等。这个不是本文重点所以就不细讲了，可以参考上面视频链接里的液体模拟方法

由于版本问题，视频中的fluidsource节点在houdini18.5中已经没有了，所以下面几步可以这样做：新建一个v属性

![](/img/in-post/houdini-vat/vat-source2.jpg)

使用pyrosource转为烟雾模拟的点，然后用volumerasterizeattributes转为体素

![](/img/in-post/houdini-vat/vat-source3.jpg)

dop内部使用volume source

![](/img/in-post/houdini-vat/vat-source1.jpg)

整个思路就是粒子模拟转换为poly并减面，最终得到一个预备模型动画。

![](/img/in-post/houdini-vat/vat-source4.jpg)

接下来的步骤在两个主流引擎里细节不一样所以分开来讲

### Unity中使用方法：

####  - VAT节点

对于unity，VAT节点设置如下，点击render将资源烘焙好

![](/img/in-post/houdini-vat/vat-unity1.jpg)

注意事项：

- Target Prim Count设置的是导出的最大三角形数量，需要低于预备模型prim数。Target Texture Size设置的是贴图的像素宽度，节点会自动算出out的贴图大小。
- 勾选Pack normals into Position Alpha会将模型的法线贴图压缩到Position贴图的alpha通道（基于世界空间，不是切线空间），这样可以省下一张贴图，但是会有精度损失。我测试的时候液体模拟效果还可以接受，但是对于布料模拟来说这样法线贴图精度就太低了。

烘焙得到三个文件夹：material中有.mat文件和.json文件(记录mat对应的参数) ；mesh中有.fbx文件；texture中有.exr图片文件

#### - 材质设置

user interface改为normal模式可以在最下面看到unity package路径，点击拷贝

![](/img/in-post/houdini-vat/vat-unity2.jpg)

打开unity的package manager，设置vat package

![](/img/in-post/houdini-vat/vat-unity3.jpg)

不过这个package只含有URP使用的shader graph，

![](/img/in-post/houdini-vat/vat-unity4.jpg)

对应路径下还有其他版本的shader，测试了下应该是老版本VAT对应的shader，解码算法与资源对应不上。目前VAT已经更新到2.1版本了，如果使用buildin管线或需要自定义shader需要参考URP文件夹下Vat_Utilies里的源码写shader。

![](/img/in-post/houdini-vat/vat-unity5.jpg)

#### - 资源设置

贴图设置在VAT中非常重要，基本设置如下

![](/img/in-post/houdini-vat/vat-unity6.jpg)

正常情况下unity会自动使用json文件数据赋值给material，如果使用自定义shader可能需要手动填写对应参数。

shader和贴图资源都设置正确的话就能看到顶点动画效果了

![](/img/in-post/houdini-vat/vat-unity7.jpg)

### Unreal中使用方法：

（发现unreal的工程没同步，晚点再填坑吧_(:з」∠)_）

#### - VAT节点

#### - 材质设置

#### - 资源设置

# 拓展: VAT混合

将其中一个模型的初始位置烘焙的另一个模型的顶点色上，并烘焙两套VAT，修改shader可以做到顶点动画混合效果，如下图：

![](/img/in-post/houdini-vat/vat-blend.gif)

