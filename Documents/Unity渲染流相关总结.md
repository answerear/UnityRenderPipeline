<center><font face="微软雅黑" size=36>Unity 渲染相关总结</font></center>

[TOC]

# 1. Unity 渲染流水线

## 1.1 Built-in Render Pipeline (BRP) 

　　内置渲染流水线

### 1.1.1 Forward Rendering Path 

　　Forward Rendering Path —— 前向渲染。前向渲染流水线根据影响物体的光照通过一个或者多个 Pass 来渲染每一个物体。依赖灯光设置和强度，灯光本身在前向渲染会区别对待。

#### 实现细节

　　在前向渲染中，被最明亮的一些灯光影响的物体都是用逐像素光照模式渲染，其次，最多有4个点光源的光照是逐顶点计算的，剩下的光通过性能较好但只能近似表示的球谐光照来计算。而是否逐像素光照计算，取决于以下：

- 灯光渲染模式（Render Mode）为 “Not Important” 时，使用逐顶点光照或者球谐光照
- 最明亮的方向光使用逐像素光照
- 灯光渲染模式（Render Mode）为 “Important” 时，使用逐像素光照
- 如果少于在质量设置中的 “Pixel Light Count" 的逐像素灯光数量，那么更多的灯光会使用逐像素，按顺序的降低亮度

　　每个物体的渲染如下：

- Base Pass 使用一个逐像素方向光和所有的逐顶点/球谐光照
- 其他的逐像素光照在额外的 Passes 中渲染，每个灯光一个 Pass

　　例如：某个物体被一系列的灯光影响（如下图所示，一个圆被灯光A到H影响）：

![ForwardLightsExample](image\ForwardLightsExample.png)

　　假设灯光A到H都是同样颜色和强度，并且它们的渲染模式都是 “Auto"，然后它们会被按照一定顺序排序。最明亮的灯会按照逐像素来渲染（A到D），然后最多有4个灯光是逐顶点渲染（D到G），最后剩下的都是球谐光照（G到H）。如下图：

![ForwardLightsClassify](image\ForwardLightsClassify.png)

　　注意上述的灯光组是有重叠的

#### Base Pass

　　Base pass 通过一个逐像素方向光和所有的逐顶点光照/球谐光照来渲染物体。这个 pass 也可以通过 shader 加任意 lightmaps，环境光和自发光。在这个 pass 中渲染的方向光可以有影子。lightmap的物体无法从球谐光照获得照明。当在 shader 中使用了 “OnlyDirectional" pass flag 时，前向 base pass 只会渲染主的方向光、环境光/光照探头和 lightmap（球谐光照和逐顶点光照没有包含到 pass 数据中）。

#### Additional Passes

　　每一个额外的逐像素光照影响的物体都需要的额外的 passes 来渲染。在这些 passes 里的光照效果默认是没有影子的（因此，前向渲染支持一个方向光有影子），除非 multi_compile_fwdadd_fullshadows 变量快捷方式开启。

#### 性能思考

　　球谐光照渲染非常高效。他们只需要很少的CPU开销，并且没有GPU开销（即：base pass 固定计算球谐光照，但是鉴于球谐光照计算原理，实际消耗跟多少球谐光照的光源没关系）。

球谐光照的一些缺点：

- 它们是物体顶点级的计算，而不是像素级。这意味着它们不支持 light 烘焙或者法线贴图
- 球谐光照是使用度不高。你不能在明亮光照转换上使用球谐算法。它们只能影响漫反射光源（很少在镜面反射使用）
- 球谐光照不是本地的，点光源和聚光灯离某些表面很近的时候，会“看起来错误”

总之，球谐光照对于一些小的动态物体已经足够了。

## 1.2 Universal Render Pipeline (URP) —— 通用渲染流水线

## 1.3 High Definition Render Pipeline (HDRP) —— 高可定义渲染流水线
一种让你可以在高端平台上创建先进的、高保真度图形的SRP

## 1.4 Scriptable Render Pipeline (SRP)