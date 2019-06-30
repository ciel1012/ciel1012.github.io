---
layout:     post
title:      "Color Space"
subtitle:   "\"伽马和线性颜色空间\""
date:       2019-06-12
author:     "Ciel"
header-img: "img/post-bg-rtr.jpg"
catalog: true
mathjax: true
tags:
    - 基础概念
---

> PBR有一个很重要的要求，就是要在线性空间中进行计算。

# 伽马空间和线性空间

#### 为什么要用伽马颜色空间

看了许多谈论伽马空间与线性空间的文章，大致分为两种原因：

- 历史原因

在早期，CRT几乎是唯一的显示设备。但是CRT输入的电压和显示出来的亮度不是线性关系。显示器电压和亮度的关系大约如下图黄色曲线，所以典型的CRT显示器大致自带一个有伽马值的幂曲线，这类伽马称为display gamma。由于这个问题的存在，那么图像捕捉设备就需要进行一个伽马校正，它们使用的伽马叫做encoding gamma，大约是下图蓝色曲线。而encoding gamma和display gamma的乘积就是整个图像系统的end-to-end gamma。
![伽马曲线](/img/in-post/color-space/gamaquxian.jpg)

虽然现在CRT设备很少见了，但为了保证这种感知一致性（这是它一直沿用至今的很重要的一点），同时也为了对已有图像的兼容性（之前很多图像使用了encoding gamma对图像进行了编码），所以仍在使用这种伽马编码。而且，虽然现在的LCD并不存在这个特点，但是为了保证兼容性，也做了和CRT一样的非线性特性。
再来看一张图：
![伽马空间](/img/in-post/color-space/gamakongjian.jpg)
右边的图像就是原始图片显示在显示器上后我们看到的，因为显示器自带的伽马变换（通常伽马值为2.2），图片会比它本身的亮度要暗。而左边的图片在图片输出前，通常是在保存图片的时候做了伽马值为0.45的伽马校正。现实中绝大部分的图片都是被伽马值为0.45校正过的，也就是说图片文件存储的都是伽马校正过的颜色值。做伽马校正几乎是每台相机、每个图像编辑软件在存储图像时默认要做的事。2.2为0.45的倒数，互相抵消得到了与原始一致的图像。如果图片没有做伽马纠正，显示的图像将是像右图一样，颜色被压黑。

- 人眼特性原因

还有看到文章里说到，其实人眼对暗部的变化更加敏感，而对亮部变化其实不是很敏感。也就是说，从0亮度变到0.01亮度，人眼是可以察觉到的，但从0.99变到1.0，人眼可能就根本察觉不出来，觉得它们是一个颜色。如果做编码，亮部位数可以少一些，而暗部需要更多的位数。只看描述还是很难感觉到，可以看下这个[实验](https://www.cnblogs.com/guanzz/p/7416821.html)，有助于理解。

1. 在Photoshop中新建一张 500*150/分辨率75/颜色模式RGB 8位/背景白色 的图，然后从左到右,从黑到白，水平拉一条渐变，如图所示：

   ![8位](/img/in-post/color-space/8bit.jpg)

   图中的明暗分界线从人眼来看大概就在中间位置。用QQ截图工具可以发现大箭头指示，最黑的地方RGB（0,0,0）,最白的地方RGB(255,255,255)，而中间位置正好是RGB（128,128,128）。可以发现你的眼睛和电脑都认为渐变的中点也是中间亮度。

2. 在Photoshop中新建一张 500*150/分辨率75/颜色模式RGB 32位/背景白色 的图，然后从左到右,从黑到白，水平拉一条渐变，如图所示：

   ![32位](/img/in-post/color-space/32bit.jpg)

   我们知道了，从黑到白的渐变里，明暗分界线的色阶应该是RGB(128,128,128),现在我们在32位图里，用QQ截图工具找这个点，会发现RGB(128,128,128)出现在了渐变过程0.2的位置（红线位置）。此时也会发现32位渐变在人眼看来不如8位的渐变自然。

“黑少白多”其实只是人眼看起来是这样，正如前面说的人眼对暗部的变化更为敏感，如果你用Photoshop自带的取色工具会发现，32位图中依然是真实的线性渐变。

由此可以发现，其实在8位图中我们看到的明暗分界线其实也应该处于0.2的位置，之前的8位图其实是欺骗了我们。8位图其实是把原来0.2的位置转换成了0.5，这样暗部就有了更多色阶来存储，而亮部则舍弃了一些人眼可能感受不到的亮部色阶变化。

值为50的颜色不是是值为25的颜色亮度的两倍，大多数的美术人员都是工作在Gamma空间下的。

8位图因为储存空间较少，因此使用的就是Gamma校正过的颜色。在只有8bit的情况下，没有必要在亮部浪费过多，这样就可以表现更多暗部的细节变化，所以实际亮度只有0.2经过gamma校正后被编码成了0.5的像素值。

#### 伽马校正

由上面的原因可以知道，如果直接将捕捉到的颜色保存到图片会有以下情况：

![\img\in-post\pbr\nocorrect](\img\in-post\pbr\nocorrect.jpg)

而伽马校正会把颜色变亮：

![\img\in-post\pbr\gammacorrect](\img\in-post\pbr\gammacorrect.jpg)

因此往往把伽马校正这个阶段提前到图片存储的时候，显示器要让这个画面变暗，一亮一暗抵消了之后就是原来的亮度了，看到所有的图片也就正常了：

![\img\in-post\pbr\gamma](\img\in-post\pbr\gamma.jpg)

当然有时不同的显示器Gamma曲线可能也有不同，例如Mac的显示器的Gamma值为1.8。图片到底应该用哪个伽马值做校正？数据预先校正完之后保存在图片中了，无法对所有的显示器分别做校正。所以人们发明了Color Profile，可以称为颜色空间或者色彩空间：

![\img\in-post\pbr\colorprofile](\img\in-post\pbr\colorprofile.jpg)

Color Profile一般也会定义伽马值，来定义一个伽马变换，这个伽马值是多少，也是包含在这个Color Profile里边的，通过伽马值就知道它的变换曲线是什么样子的。从显示器那一端可以拿到当前的Color Profile，甚至可以指定用哪一种Color Profile，选不同配置的时候，桌面或者图片显示出来的颜色是不一样的。

![\img\in-post\pbr\srgb](\img\in-post\pbr\srgb.jpg)

sRGB是一种最常用的Color Profile，是由微软和惠普联合制定的一个规范，它的使用很广泛以至于我们经常把sRGB等同于了伽马颜色空间。如果显示一个颜色，所有的显示设备都用了sRGB的颜色空间，存储一个图片或者保存一个视频也好或者其它图片工具也使用sRGB的编码，因为都在一个颜色空间下，所以输入和显示是可以免去转换的。也就是RGB值可以直接输出，省略了转换过程。

#### 伽马空间工作流程

以unity中为例，如果贴图没有经过伽马校正，会出现如下情况：

![\img\in-post\pbr\unity_nogamma](\img\in-post\pbr\unity_nogamma.jpg)

贴图是没有经过伽马校正的，因此被显示器进行伽马变换后变暗了。图片参与光照计算后得到了正确结果被保存在Frame Buffer，但显示器进行伽马变换后将结果变暗了，这显然不是我们想要的结果。

如果将贴图事先进行伽马校正则会出现如下情况：

![\img\in-post\pbr\unity_gamma](\img\in-post\pbr\unity_gamma.jpg)

可以看到原始图片经过伽马校正被提亮，传到unity中做光照计算。因此Frame Buffer中看到它的计算结果特别亮，经过显示器伽马变换后，亮度被压了下来。这个亮度似乎让人觉得正常，但它的结果其实是不正确的，后面会详细讲到。

#### 线性颜色空间

拿刚才的渲染流程对比一下，上面伽马空间流程里输出是偏暗的，正常的亮度被显示器压低了，我们没有办法在显示器输出前进行校正。但是sRGB Frame Buffer可以，如果使用sRGB Frame Buffer，它能够在结果输出到显示器这个阶段做sRGB伽玛校正，如图所示，最后的显示是正常的。

![\img\in-post\pbr\sRGBframebuffer](\img\in-post\pbr\sRGBframebuffer.jpg)

sRGB Frame Buffer是由硬件支持的，刚才所做的变换用Shader也可以实现，sRGB Frame Buffer的转换速度要比它快。安卓手机只有OpenGL ES3.0才可以支持它，所以你会看到开启线性空间必须指定Graphics API为OpenGL ES3.0。需要注意的是Alpha值不做任何变换的，它只支持每通道8位的格式。

![\img\in-post\pbr\sRGBframebuffer1](\img\in-post\pbr\sRGBframebuffer1.jpg)

目前能用到的大部分作图软件都是在sRGB的空间下制作的，软件内的预览和图片的保存都经过了伽玛校正。这个在伽玛颜色空间下没问题，但是在线性颜色空间里，贴图的颜色应该是线性的。所以当使用线性空间时，经过sRGB校正的贴图应该被还原。有二种方法，可以把图重做改成线性空间，或者可以使用sRGB Sampler。

![\img\in-post\pbr\sRGBframebuffer2](\img\in-post\pbr\sRGBframebuffer2.jpg)

sRGB Sampler也是硬件支持的一个特性，如果勾选sRGB选项，硬件会认为图是sRGB编码的，在采样颜色的时候会先做一次sRGB反向转换，再把结果返回，所以Shader读到的颜色是校正之前的颜色。

![\img\in-post\pbr\unitysRGB](\img\in-post\pbr\unitysRGB.jpg)

如果贴图是线性存储的，则不需要勾选sRGB选项，采样得到的颜色值就是直接从贴图中取得。

![\img\in-post\pbr\unitysRGB1](\img\in-post\pbr\unitysRGB1.jpg)

使用伽马空间的原因如下：

![\img\in-post\pbr\reson](\img\in-post\pbr\reson.jpg)

然而伽马空间有很多问题，$func(贴图颜色_{r/g/b}^{gamma},光照系数)\not=func(贴图颜色_{r/g/b},光照系数)^{gamma}$。

![\img\in-post\pbr\gamma2](\img\in-post\pbr\gamma2.jpg)

二张图中左边是线性正确值，右边的曝光结果不对，因为这种色温比较温和的颜色，会迅速的曝光，存的这个值在经过sRGB变换了之后已经比原来的值大了。注意右边上眼皮轮廓应用的地方，你会发现有一些黑、有一些蓝色，也是一样的问题，当你做光照计算的时候，冷色调和暖色调的差别已经拉开了，把光照系数加上去，差异变得更大，这就是你发现结果不对的原因。

![\img\in-post\pbr\TosRGB](\img\in-post\pbr\TosRGB.jpg)

#### 选择颜色空间

如果要求的是物理写实的效果，才只能考虑使用线性空间。其实在其他的情况下，二种方式都是可以的。

![\img\in-post\pbr\space](\img\in-post\pbr\space.jpg)

2D游戏推荐使用伽马空间，这不是Unity的考虑，而是美术工作流的考虑，需要花很长时间培训美术，转到线性空间做图，这是成本比较高的，所以推荐伽马空间。3D游戏推荐使用线性空间，尤其使用PBR时线性空间是前提，伽马空间是做不了的。

![\img\in-post\pbr\space1](\img\in-post\pbr\space1.jpg)

#### 常见问题

- Substance Painter的效果和Unity中不一样

![\img\in-post\pbr\substance](\img\in-post\pbr\substance.jpg)

- PhotoShop中导出的图片应该如何设置

如果使用线性空间，一般来说Photoshop可以什么都不改，导出的贴图只要勾上sRGB就可以了。你也可以调整PhotoShop的伽玛值为1，导出的贴图在Unity中也不需要勾选sRGB了。二种流程都可以，因为我们要求它的预览效果和实际效果是一样的，所以只要保证Photoshop Unity两边的变换结果是一致的就可以。

![\img\in-post\pbr\ps](\img\in-post\pbr\ps.jpg)

#### 线性空间制作半透明图片

Photoshop的图层和图层之间做混合的时候，每个上层图层都经过了伽马变换，然后才做了混合，这个方式是设置中的默认值。

![\img\in-post\pbr\ps1](\img\in-post\pbr\ps1.jpg)

需要在设置中更改它，选择“用灰度系数混合RGB颜色”，参数设置为1，这样图层直接就是直接混合的结果。但是如果你修改了这个选项，你可能只有选择上面的Linear流程制作贴图了。

![\img\in-post\pbr\ps2](\img\in-post\pbr\ps2.jpg)

sRGB编码是为了增加8位颜色的精度，如果你是用了32位浮点数的贴图格式，PhotoShop自动使用的是线性空间，没有做任何伽玛变换。

![\img\in-post\pbr\ps3](\img\in-post\pbr\ps3.jpg)

- 线性空间下截图和游戏画面有差异

![\img\in-post\pbr\liner](\img\in-post\pbr\liner.jpg)

- HDR和线性空间同时开启的结果

32位浮点格式的Frame Buffer是没有sRGB扩展的。这一点Unity已经为你做好了处理，浮点格式的Frame Buffer在输出前会先绘制在8位的sRGB Frame Buffer上，然后输出到屏幕，伽玛校正在这个阶段自动完成。

![\img\in-post\pbr\liner2](\img\in-post\pbr\liner2.jpg)

#### 参考链接

[https://connect.unity.com/p/unite-2018-qian-tan-jia-ma-he-xian-xing-yan-se-kong-jian](https://connect.unity.com/p/unite-2018-qian-tan-jia-ma-he-xian-xing-yan-se-kong-jian)

[https://blog.csdn.net/u014370148/article/details/79442001](https://blog.csdn.net/u014370148/article/details/79442001)
