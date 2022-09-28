---
layout:     post
title:      "[Houdini] Stylized Tree Generator"
subtitle:   "\"使用Houdini制作风格化树\""
date:       2022-09-14
author:     "Ciel"
header-img: "img/post-bg-houdini.jpg"
catalog: true
mathjax: true
tags:
    - Houdini

---

# 使用Houdini制作风格化树

> 网上有很多制作风格化树的教程，不过大多是建模软件手动处理。我分析了下制作过程，使用Houdini程序化去处理。

B站链接：https://www.bilibili.com/video/BV15L4y1q74a（视频是一年前的了，评论都是求教程的，那就写篇教程吧\_(:з」∠)\_）

## 渲染方式

### 1.插片模型，双面渲染

非常常用的制作方式，网上教程很多就不多说了，直接放截图（和视频里不一样是因为录完我又改了材质）

![截图.png](/img/in-post/stylized-tree/image1.png)

![截图.png](/img/in-post/stylized-tree/image2.png)

![截图.png](/img/in-post/stylized-tree/image3.png)

### 2.特殊UV模型，公告板渲染

渲染思路来源：<https://www.youtube.com/watch?v=iASMFba7GeI>

核心就是模型的每个四边面都要用整个UV空间，且使用硬边。每个面（叶子）当成独立公告板渲染。视频里使用的是unity，我用的是unreal。

![截图.png](/img/in-post/stylized-tree/image4.png)

**World Position Offset**

Unreal屏幕坐标原点在左上角，所以UV坐标从0-1转换到-1-1时，需要翻转V值

![截图.png](/img/in-post/stylized-tree/image5.png)

相机空间转换到模型空间，此时可以添加一个缩放参数（叶子尺寸）。然后再从模型空间转换到世界空间，添加顶点法线上的偏移增加一点变化。最后给一个整体的缩放值。

![截图.png](/img/in-post/stylized-tree/image6.png)

加入风力抖动

![截图.png](/img/in-post/stylized-tree/image7.png)

**Base Color**

基础色和偏色混色，混色Alpha根据顶点法线和顶点上烘焙的AO信息（如果有的话）计算

![截图.png](/img/in-post/stylized-tree/image8.png)

**Emissive Color**

自发光模拟边缘高光

![截图.png](/img/in-post/stylized-tree/image9.png)

**Opacity Mask**

叶子alpha蒙版

![截图.png](/img/in-post/stylized-tree/image10.png)

## 程序化建模

不同模型对应的是不同渲染方式，片状指的是用插片，块状指的是特殊处理UV的模型。

![截图.png](/img/in-post/stylized-tree/image11.png)

Houdini也有生成树的工具，虽然偏写实，但是有些思路可以借鉴参考（Labs工具中真的可以学到很多技巧）

Houdini Labs quick basic tree

![截图.png](/img/in-post/stylized-tree/image12.png)

整个工具并不复杂，分成下面几块。

![截图.png](/img/in-post/stylized-tree/image13.png)

### 树干曲线

输入初始主干曲线，设置pscale（树干粗细），lift（分支抬起角度）

![截图.png](/img/in-post/stylized-tree/image14.png)

选取主干可以生成分支的区域，生成几个一级分支就循环几次。

![截图.png](/img/in-post/stylized-tree/image15.png)

另一边设置分支曲线，和主干一样的处理，这里旋转曲线朝向Z轴，方便后续根据主干上点的法线方向copy to point分支曲线。

![截图.png](/img/in-post/stylized-tree/image16.png)

重新计算点的法线方向，添加一些随机变化，lift属性在此处用来控制倾斜角度，随机选择一个点作为本次循环分支。

![截图.png](/img/in-post/stylized-tree/image17.png)

使用主干上point的pscale属性缩放复制分支曲线，bend曲线，capture的Length是分支曲线的长度。

![截图.png](/img/in-post/stylized-tree/image18.png)

同理制作二级分支，这里加了个小处理是有一定概率不生长二级分支，是否有二级分支整体做了个switch。当然可以做成循环，做更多层分支，不过是做风格化树，枝干一般不会特别多层，所以我这里偷个小懒。

![截图.png](/img/in-post/stylized-tree/image19.png)

分支比主干pscale更小，合并主干和分支曲线。

![截图.png](/img/in-post/stylized-tree/image20.png)

最后给曲线加点noise，就完成了树干曲线。主干底部需要处理下最后的旋转，我这里还略微加粗了一点。

![截图.png](/img/in-post/stylized-tree/image21.png)

### 树干模型

sweep曲线得到树干模型

![截图.png](/img/in-post/stylized-tree/image22.png)

根据像素密度重新计算UV，调整UV大小旋转（匹配贴图），添加材质属性。

![截图.png](/img/in-post/stylized-tree/image23.png)

### 树叶大型

选取主干分支的末端

![截图.png](/img/in-post/stylized-tree/image24.png)

每个点周围继续撒点，控制撒点范围的shadpe和size，这会影响整个树冠造型。

![截图.png](/img/in-post/stylized-tree/image25.png)

Particle Fluid Surface节点可以根据点生成表面模型，重新计算表面法线，blur一下法线更柔和。得到树冠轮廓，之后可以用它来传递法线（参考球形法线）和AO。

![截图.png](/img/in-post/stylized-tree/image26.png)

在原来基础上再做些小球，给树冠造型更多小变化，得到树冠大型。

![截图.png](/img/in-post/stylized-tree/image27.png)

### 片状树叶

插片树叶模型，也可以导入外部模型。

![截图.png](/img/in-post/stylized-tree/image28.png)

根据树叶大型撒点复制树叶，调整树叶数量，大小，朝向。

![截图.png](/img/in-post/stylized-tree/image29.png)

用树冠轮廓传递法线和AO信息

![截图.png](/img/in-post/stylized-tree/image30.png)

### 块状树叶

树叶大型删除面只保留点，重新计算粒子流体表面后减面。

![截图.png](/img/in-post/stylized-tree/image31.png)

细分平滑，保证每个面都是四边面，计算AO和法线。

![截图.png](/img/in-post/stylized-tree/image32.png)

计算每个面的UV，使UV平铺整个UV空间，使用Vex直接指定UV值。

![截图.png](/img/in-post/stylized-tree/image33.png)

加了点随机，UV原点可以是四个顶点任意一点，这样UV会有旋转变化。

![截图.png](/img/in-post/stylized-tree/image34.png)

树叶材质路径

![截图.png](/img/in-post/stylized-tree/image35.png)

最后删除多余属性后合并输出，大功告成。

![截图.png](/img/in-post/stylized-tree/image36.png)

# 总结

除了调整HDA参数，调整曲线也可改变树的造型（感觉当初标题就应该取得大胆点，几秒一个！）

![](/img/in-post/stylized-tree/tree.gif)

这是一次渲染知识与程序化知识结合的小测试。HDA中某些处理是为了渲染服务，理解整个制作过程，模型和材质都正确才能达到好的效果。另外，个人感受是不用过度追求程序化，完全随机可能不是美术想要的结果。合理拆分模块，比如树干和大型也可以直接用DCC软件制作，导入Houdini进行后续处理，这样得到的树造型更可控。
