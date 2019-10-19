# Unity Lighting

> 想要在unity中实现好的渲染效果，对引擎光照的了解是必不可少的。下图是官方给出的Lighting pipeline 流程图，针对图中各点进行总结，但没有太过详细，具体可深入查阅相关资料。

![\img\in-post\unity-lighting\BestPracticeLightingPipeline15](..\img\in-post\unity-lighting\BestPracticeLightingPipeline15.svg)

## Render Pipelines 渲染管线

1. Build-in 内置管线，又可分为：

   - Forward 前向渲染

   - Deferred 延迟渲染

   在Unity推出SRP之前，一般项目多使用(多pass)前向渲染模式，场景中的所有对象都是一个接一个依次渲染的。因此，当对象被多个灯光照亮时，渲染成本会急剧增加。前向渲染可以轻松处理透明度，大多数unity商店资源都支持这种渲染模式。

2. Lightweight Render Pipeline(LWRP) 轻量级渲染管线，2019.3版本中改名为Universal Render Pipeline(URP)

   快速单通道正向渲染，是专为移动平台、平板电脑和XR等实时照明要求较低的设备设计的。

3. HDRP

   综合了Deferred/Forward、Tile/Cluster 渲染方式，提供高品质渲染效果，适合PC和主机平台。

如何选择需要结合项目实际需求，可参考下图：

![\img\in-post\unity-lighting\BestPracticeLightingPipeline16](..\img\in-post\unity-lighting\BestPracticeLightingPipeline16.svg)

## Global Illumination 全局光照

全局照明(GI)是一个模拟光如何从表面反弹到其他表面的系统(间接光)，而不仅仅局限于光源(直接光)直接照射到表面的光。精确模拟全局光照，计算消耗昂贵，因此游戏一般采用预计算方式实现。在Unity中有两个全局照明系统，可以在Window > Rendering > Lighting Settings中设置。

1. Precomputed Realtime GI Lighting  预计算实时全局光照

   交互式更新场景光照信息，也就是说能实时响应光照条件的变化。为了保证游戏运行时的帧率，需要将这些冗长的数据处理从一个实时过程转变为一个“预先计算”的过程。

   GI系统希望存储的是场景中的间接光，这种颜色通常比较柔和，不会有高频变化。unity的预计算实时GI利用了这种漫反射特性。假设不捕捉光照细节如清晰的阴影，大大降低全局光照解决方案的分辨率，有效减少了游戏运行期间更新GI照明所需的计算次数。

   该系统完全依赖于第三方照明中间件Enlighten。在Unity的预计算过程中，Enlighten先后经过两个阶段，包括：集群化和光传输。第一阶段将场景分解简化为以“集群(clusters)”为单位进行组织的集合，在第二阶段计算集群与集群之间的可见性。预计算后的数据在运行时用于交互性地生成场景的间接照明。通过将世界简化成一个关系网络，无需在游戏运行中进行昂贵的光线追踪计算。 Enlighten的优势在于能够实时改变间接照明效果，因为预计算的数据依赖于集群之间的关系。但是，与其他光照贴图技术一样，改变场景中的静态几何体将触发新的预计算。

2. Baked GI Lighting 烘焙全局光照

   计算光照对场景中静态物体的影响，照明信息被烘焙到光照贴图和光照探头中。

   这些光照贴图可以既包括直接照射到表面的光，也包括从场景中其他物体或表面反射回来的“间接”光。可以与物体材质的“着色器”中的颜色和法线等表面信息一起使用。

   烘焙光照不能在游戏运行时改变，因此被称为静态。实时光可以被覆盖和叠加到被光照映射的场景中，但是不能交互式改变LightMap本身。

   烘焙GI 系统可以使用以下两种技术之一

   - Enlighten （[Unity 2021.1将完全移除Enlighten](https://connect.unity.com/p/enlightenxi-tong-qi-yong-he-ti-dai-de-jie-jue-fang-an)）
   - Progressive Lightmapper

   Enlighten和Progressive Lightmapper 使用了不同的技术计算光照，所以两者产生的光照效果会有不同。

   **无论使用哪种全局照明系统，Unity 都只会考虑标记为“Lightmap Static”的游戏对象。动态游戏对象需要借助场景中放置的光照探头来接收间接照明**。（[unity2019.2版本开始使用Contribute Global Illumination标志](https://blogs.unity3d.com/2019/08/28/static-lighting-with-light-probes/)）

   由于全局光照计算是一个相对缓慢的过程，因此只有具有明显光照变化的大型复杂资源才需要应标记为“Lightmap Static”。接收均匀光照的较小网格可保持为动态设置，然后通过使用 Light Probes 为其提供近似效果的间接照明效果。较大的动态游戏对象可以使用 Light Probe Proxy Volume（LPPV），以便在局部接收更好的间接照明。限制场景中静态游戏对象的数量对于提高烘焙时间同时保持足够照明品质至关重要。

   在Unity中可以同时使用烘焙和实时GI技术，但是必须注意，同时使用会大大增加烘焙时间和程序运行时的内存消耗，因为这两个系统不使用相同的数据,同时渲染两个系统的性能成本正好是它们的总和。不仅要存储两组光映射，而且还要在着色器中解码处理。此外，间接照明在运行时的交互式更新将给CPU带来额外的压力，并且在视觉上，烘焙和实时GI提供的间接照明效果会有差异，因为它们使用了不同的技术来模拟间接照明，并且通常在完全不同的分辨率下执行。若同时使用这两种技术，建议将使用范围限制在高端平台或具有可预测性能成本且严格把控场景的项目中，同时，建议由对所有照明设置有很好理解的团队成员负责，因为管理这两个系统相对复杂。

   因此，对于大多数项目而言，尽量避免同时使用两种GI技术，选择其一是相对比较稳妥的做法。采取哪种方法的决定必须根据特定项目的性质和期望的目标平台进行评估。在针对一系列不同的硬件时，通常是性能最差的部分决定了需要哪种方法。

   默认情况下，两种方案在Unity的照明面板(Lighting>Scene)中都启用。然后可以由每个灯单独控制(Inspector>Light>Mode)。可以通过取消其中一种方案来覆盖灯光设置。

![\img\in-post\unity-lighting\BestPracticeLightingPipeline4](..\img\in-post\unity-lighting\BestPracticeLightingPipeline4.svg)

## Light Mode

1. Realtime Lighting

   实时光照是对物体进行照明最基本的方式，对于照亮移动物体非常有用。但是它们本身的光线并不反弹。要使用全局照明技术创建更真实的场景，需要启用unity的预计算照明解决方案。directional,spot,point默认状态下是Realtime mode的。

   默认模式是“Realtime”意味着选择的光线仍然会直接作用于你的场景，而间接光线则由Unity的预先计算的实时GI系统处理。

   如果不使用任何GI技术，或者仅使用实时GI，那么所有Baked和Mixed模式的灯光都将被作为Realtime进行处理（此时，mixed和baked不可选）。
   - 优点

     - 可交互，灯光修改即时反馈

     - 可产生动态阴影

     - 物体表面有高光

   - 缺点

     - 开销很大

     - 只使用Baked GI时不产生间接光照

![\img\in-post\unity-lighting\noGI](..\img\in-post\unity-lighting\noGI.png)

2. Baked

   该模式下直接光和间接光信息都将被烘焙成光照贴图，不能在游戏运行过程中改变。烘焙这些信息十分耗时，好处是没有运行时成本来处理这些光照，但光照映射到场景有一个小的成本。

   - 优点

     - 高性能

     - 静态物体生成间接光照和阴影

   - 缺点

     - 不能运行时修改灯光属性，修改后需要重新烘焙

     - 物体表面没有高光

     - 烘焙精度决定烘焙质量，越高越耗时

3. Mixed

   选择Mixed模式，标记为静态的GameObjects会在其烘焙的GI光照图中包含此光照信息。然而与标记为baked的灯光不同，混合灯光仍然可以实时、直接地照亮场景中的非静态游戏对象。可用于在静态环境中使用了光照贴图，但仍希望角色也被这些光线影响，从而将实时阴影投射到光照贴图的几何体上。

   - 优点

     - 可产生动态阴影

     - 物体表面有高光

   - 缺点

     - 开销较大

     - 烘焙耗时

Light Mode 对应 Lighting 设置

![\img\in-post\unity-lighting\3GI](..\img\in-post\unity-lighting\3GI.png)

## Mixed Lighting Mode

1. Baked Indirect 烘焙间接光![\img\in-post\unity-lighting\mix1](..\img\in-post\unity-lighting\mix1.png)

2. Substractive 烘焙直接光照、间接光照和阴影![\img\in-post\unity-lighting\mix2](..\img\in-post\unity-lighting\mix2.png)

3. Shadowmask 烘焙间接光照和阴影![\img\in-post\unity-lighting\mix3](..\img\in-post\unity-lighting\mix3.png)

- Shadowmask Mode可以在Edit > Preferences > Quality设置

1. Shadowmask 使用Lightmap_shadowmask上的烘焙阴影

2. Distance Shadowmask 根据设置的Shadow Distance参数对实时阴影和烘焙阴影进行融合

![\img\in-post\unity-lighting\mix4](..\img\in-post\unity-lighting\mix4.jpg)

如何选择Light Mode

![\img\in-post\unity-lighting\BestPracticeLightingPipeline12](..\img\in-post\unity-lighting\BestPracticeLightingPipeline12.svg)

![\img\in-post\unity-lighting\BestPracticeLightingPipeline5](..\img\in-post\unity-lighting\BestPracticeLightingPipeline5.svg)

## Light  Sources

1. Directional 平行光

2. Spot 聚光灯

   聚光灯目前不支持间接阴影，在使用预计算实时GI时。这意味着由聚光灯产生的光将穿过几何图形并在另一边反弹。因此，需要仔细考虑设置问题。

3. Point 点光

   为点光源启用阴影开销昂贵，因此必须谨慎使用。点光源要求阴影必须在6个世界方向上渲染6次，在较慢的硬件上这可能是一个无法接受的性能代价。

   它也不支持间接反弹光的阴影。这意味着由点光源产生的光将继续穿过物体并在另一侧反弹，除非被范围衰减。这可能导致光线通过墙壁和地板，因此必须小心地放置灯光以避免此类问题。使用烘焙GI时没有这个问题。

4. Area(baked only) 区域光

   区域光只在烘焙GI中可用，在整个表面上均匀发光。区域光范围不能手动控制，但是强度随着距光源距离的平方成反比减小。可以用在希望创建柔和的灯光效果时，常用来作为天花板灯光或背光。

   区域光计算时从世界上的每个lightmap texel向光发射大量射线，以确定是否可以看到光线。这意味着区域灯光在计算上非常昂贵，并且会增加烘焙时间。然而，如果使用得当，它们可以为场景照明增加很大的真实感，这种额外的预计算是合理的，因为他们只是烘焙，游戏运行性能不受影响。

5. Emission 自发光

   虽然区域灯不支持预计算实时GI，想达到类似软照明效果可以使用自发光材质。就像区域灯一样，自发光材质通过其表面区域发射光线。他们能贡献场景中的反射光线，如颜色和强度可以在游戏过程中改变。

   自发光材质没有具体距离，但是发出的光将以平方的速度衰减。只有标记为static或Lightmap static的物体才会接受自发光材质的产生影响。同样，没有标记为静态的自发光材质不会影响场景光照。

   不过自发光值大于0的材质仍然会在屏幕上显示出明亮的光芒，即使它们对于场景照明没有贡献。

   自发光材质只会直接影响场景中的静态几何体。如果需要动态的或者非静态的几何体，例如角色，为了从自发光中获取光，必须使用Light Probes。在游戏中改变发射值将会交互式地更新光照探头，任何当前从这些探针接收到光的物体上可以看到改变后结果。

除了以上的光源外，还需注意**Ambient Lighting**的影响。它影响场景整体显示效果和亮度的环境照明，可以被认为是一个全局光照，从各个方向影响场景中的物体。在Lighting>Scene>Environment Lighting设置

根据选择的艺术风格，环境光可以起到很大作用。比如想要一个明亮的卡通风格，黑暗的阴影可能是不受欢迎的，或者照明可能是手绘成纹理的。如果需要增加一个场景的整体亮度，而不调整个别灯，环境光也能起到调节作用（天空盒不受影响）。

没有使用Unity的预计算照明解决方案，环境光将不会被遮挡，因此不会是物理上准确的。然而，如果在场景中启用了烘焙GI或预计算实时GI，那么这个“天光”将会被你的场景中的对象所阻挡，从而产生更真实的效果。

使用环境光的一个重要优点是它的渲染成本很低，因此对于移动应用程序特别有用，在移动应用程序中，可能需要尽量减少场景中的灯光数量。

请注意，改变环境光源的颜色并不影响可见的天空盒，它只影响场景中照明的颜色。

## Reflection Probes

Unity场景默认使用天空盒来进行反射计算。在许多情况下，物体可能被遮挡。它们可能在室内，也可能在桥梁或隧道等建筑下方。为了创建更精确的反射，我们需要使用反射探针来采样对象所看到的东西。这些探测器从它们在三维空间中的位置绘制世界，并将结果写入cubemap。这可以被附近的物体使用，给人一种反映周围世界的印象。

GameObject>Light>Reflection Probe创建一个反射探针。实时反射探针性能消耗很大，因为会为每个反射探针额外渲染6次场景。因此一般使用性能较好的烘焙反射探针。使用烘焙反射探针需要将想要烘焙的物体勾选static。

## Light Probes

lightmaps存储关于光到达场景表面的照明信息，而light probes存储关于光通过场景中的空白空间的信息。主要有两点用途：

- 为场景中移动的物体提供高质量的照明(包括间接反射光)。
- 在静态场景使用Unity的LOD系统时，为静态场景提供照明信息。

只有静态对象才能使用unity的烘焙或预计算实时GI系统。为了让动态对象，如交互元素、角色能够接收到静态几何体所接收的一些丰富的反弹光，unity将这些信息记录成一种格式，以便在游戏过程中快速读取并用于照明方程。

unity通过在世界上放置样本点，然后从各个方向捕捉光线来做到这一点。这些点记录的颜色信息被编码成一组值(或“系数”)，这些值可以在游戏过程中快速计算。在Unity中，称这些采样点为“光照探针”。

![\img\in-post\unity-lighting\lightprobe](..\img\in-post\unity-lighting\lightprobe.png)

为了得到更好的计算结果，在照明变化的区域（例如阴影或颜色过渡）周围以更大的密度放置这些采样点。

光照探针允许移动的物体响应复杂的反弹照明，无论使用烘焙GI或预计算实时GI。一个对象的网格渲染器将寻找周围的光照探针的位置和混合他们的值。这是通过寻找由光照探针的位置组成的四面体，然后决定物体的支点落在哪个四面体上来实现的。这让我们可以在场景中放置移动的角色，并让它们适当地融合在一起。如果没有光照探针，动态对象就不会接收GI。

默认情况下，场景中没有光照探针，所以需要自己创建(GameObjects>Light>Light Probe Group)。

如果勾选了自动烘焙，每当改变场景照明或静态几何时会自动更新光照探头。否则当单击Build按钮时才被更新。

## Light Probe Proxy Volume

LPPV 可以为大型动态游戏对象(不能使用烘焙的光照贴图的对象)使用更多的照明信息，例如粒子系统或蒙皮网格。为受光照探针影响的游戏对象提供了一个空间梯度。

标准着色器（Standard Shader）支持该特性。如果想将该特性添加到自定义着色器中，需要使用 ShadeSHPerPixel 函数（该函数位于UnityStandardUtils.cginc）

![\img\in-post\unity-lighting\Unity Lighting Modes Reference Card](..\img\in-post\unity-lighting\Unity Lighting Modes Reference Card.jpg)

## 参考资料

https://docs.unity3d.com/Manual/BestPracticeLightingPipelines.html

[https://docs.unity3d.com/Manual/LightingInUnity.html](https://docs.unity3d.com/Manual/LightingInUnity.html)

[https://docs.unity3d.com/Manual/GlobalIllumination.html](https://docs.unity3d.com/Manual/GlobalIllumination.html)

[https://docs.unity3d.com/Manual/LightModes.html](https://docs.unity3d.com/Manual/LightModes.html)
