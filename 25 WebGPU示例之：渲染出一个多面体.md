# 25 WebGPU示例之：渲染出一个多面体

**多面体，顾名思义就是由多个三角形拼凑而成的、拥有多个面的物体，比如立方体、圆柱体、甚至是球体。**

**在之前示例中渲染单个三角形的基础上，本文使用 WebGPU 渲染一个多面体涉及的新知识点有：**

1. 一个三角形的正反面判定

2. N 个三角形顶点信息的存储先后顺序
3. 渲染通道中的深度附件，用于解决 N 个三角形之间的遮挡关系



<br>

**知识点1：一个三角形的正反面判定：**

> 这是一个 图形学 中的知识点，WebGL、WebGPU 都遵循这个原则。

这里的三角形 “正反面” 即 “正面和反面“，当然你可以理解为 “正面和背面”、“三角形的面是否朝向相机”。

特别强调：

1. 这里说的 “正反面” 是针对 单个三角形(面)，并不是指 多面体(例如立方体盒子)的 “外面和里面”。
2. 所谓 “正反面” 实际上是一个由观察者所处的位置决定的，就好像现实世界中的 “左和右”，它都是一个 “相对结论”。



![triangle_abc](https://raw.githubusercontent.com/puxiao/webgpu-tutorial/main/imgs/triangle_abc.jpg)

假设一个三角形的 3 个顶点坐标分别为 a, b, c，假定此刻观察者就是屏幕前的你，也就是说相机处于 z 轴正方向，那么：

1. 若这 3 个顶点的连接顺序为**逆时针**，例如 a > b > c，则判定观察者此时看到的是**正面**
2. 若这 3 个顶点的连接顺序为**顺时针**，例如 a > c > b，则判定观察者此时看到的是**反面**

<br>

> 以上是基于右手坐标系而言的。



<br>

上面提到的 顺时针 或 逆时针 是“处于某个观察位置的观察者通过大脑思考” 得出了，对于计算机而言则是通过 **叉乘** 来判断的。

> 叉乘 也被称为 叉积、外积、向量积
>
> 两个向量进行叉积运算得到同时垂直于这两个向量的另外一条向量，叉积常用来构建 xyz 坐标系

假定三角形的 3 个顶点连接顺序为 a > b > c，那么：

1. 由 a 到 b 可以得到一个向量 v1
2. 由 b 到 c 可以得到一个向量 v2
3. 将 v1 与 v2 进行叉乘，得到 v3

此时通过判断 v3 的 z 轴值：

1. 若 z 轴值大于 0 则为正面(逆时针)
2. 若 z 轴值小于 0 则为负面(顺时针)



<br>

代码示例：

```
import { Vector3 } from "three";

const a = new Vector3(0.0, 0.5, 0.0)
const b = new Vector3(-0.5, -0.5, 0.0)
const c = new Vector3(0.5, -0.5, 0.0)

//三角形连接顺序 abc，形成的 2 个向量为 ab, bc
const ab = new Vector3().subVectors(b,a)
const bc = new Vector3().subVectors(c,b)
const ab_bc = ab.cross(bc)
console.log(ab_bc.z) 
//输出值为 1，大于 0 即判定结果为正面(逆时针)


//三角形连接顺序 acb，形成的 2 个向量为 ac，cb
const ac = new Vector3().subVectors(c,a)
const cb = new Vector3().subVectors(b,c)
const ac_cb = ac.cross(cb)
console.log(ac_cb.z)
//输出值为 -1，小于 0 即判定结果为负面(顺时针)
```



<br>

> 上面讲解的三角形正反面判断是一个图形学中比较基础的知识点。实际上和本文后面要写的示例关系并不大，但是我觉得比较重要，所以就写出来了。



<br>

**知识点2：N 个三角形顶点信息的存储先后顺序**

我们知道下面几个事情：

1. 一个多面体由  N 个三角形拼凑而成
2. 每个三角形包含 3 个顶点坐标信息
3. 这 N 个三角形的全部顶点数据都依次存储在 顶点缓冲区 中

那么问题来了：以 三角形 3 个顶点为一个单位，以不同的顺序将这 N 个三角形存入顶点缓冲区中，会对渲染结果造成影响吗？

> 换而言之，我们保证每个三角形的 3 个顶点顺序是固定的，但是这 N 个三角形的存入顺序是不同的。



<br>

> 换一种问法：假设一个多面体由 10 个三角形组成，那这 10 个三角形存入到顶点缓冲区的先后顺序不同，会影响渲染结果吗？



<br>

答案是：会，但不是 100%。



<br>

再问：那会造成什么影响？

答：如果三角形没有按照相邻的顺序依次存入，而是 “跳跃式” 的存入，则会有一些三角形 “莫名其妙” 地没有被渲染出来。

> 这是我在编写本文对应示例时，遇到了这样的问题。
>
> 因为最开始我以为只要是三角形的顶点坐标是固定的，顺序无所谓，但实际运行发现并不是这样。



<br>

这个现象背后的原理，或者是根源是什么？

我暂时也不知道。

我只是提醒你：

1. 当创建多面体时，三角形的存入顺序原则应当是 “按照相邻原则依次存入”
2. 当某些三角形 “意外”、“没有按照预期” 被渲染出来时，可以去检查一下三角形的存入顺序



<br>

> 准确来说这不是一个知识点，而是我个人的一个经验。
>
> 如有有人知道原因，请告诉我。



<br>

**知识点3：渲染通道中的深度附件，用于解决 N 个三角形之间的遮挡关系**

假设需要渲染 N 个三角形，对于 WebGPU 而言它默认采用的是 “画家算法”，简单来说就是：把需要绘制的三角形依次绘制出来，后绘制的永远在最上面。

由于绘制的先后顺序仅仅由 三角形 在顶点缓冲区中的存储顺序来决定的，只是机械地一个又一个绘制，一层又一层的叠加，并没有考虑这些三角形在不同角度情况下他们在空间中的 远近(深度) 因素，这会导致有些远离相机的面反而出现在最前面。

为了解决这个问题，需要在渲染通道中引入 深度附件，通过对 深度附件 的配置来明确告知管线在绘制时考虑进去 面(三角形) 的空间远近，也就是 深度，确保最终渲染的结果符合正常视觉。



<br>

不同配置的深度附件可以决定出不同的最终渲染结果。

> 假定某些场景下，希望渲染出不符合自然视觉的，比如 印象或灵魂 画派，就是可以设置深度附件，让远的物体在前，近的物体再后，再或者随机空间错乱。

> 但这种特殊的场景暂时不在我们考虑范围内，我们只需要配置常规的深度附件即可。



<br>

管线渲染中配置常规的深度附件很简单，只需要 3 步：

1. 创建一个纹理，作为深度附件的纹理
2. 创建渲染管线时，配置 `depthStencil` 属性
3. 在渲染通道中，添加并配置 `depthStencilAttachment` 属性



<br>

```
const depthTexture = device.createTexture({
    size: {
        width: canvas.width,
        height: canvas.height
    },
    format: 'depth24plus',
    usage: GPUTextureUsage.RENDER_ATTACHMENT
})
```

> 该纹理的用途(usage)为：GPUTextureUsage.RENDER_ATTACHMENT
>
> attachment 这个单词的翻译为：附件、附属物、附加装置，所以 RENDER_ATTACHMENT 可以翻译为 “渲染附件”



<br>

> 快速复习一下：
>
> 颜色附件：用来配置对上一次渲染结果(颜色)的处理方式，通常会配置为 不保存且清除
>
> 深度附件：用来配置三角形的深度处理方式，即决定三角形的前后顺序



<br>

```
const pipeline = device.createRenderPipeline({
    ...
    depthStencil: {
        depthWriteEnabled: true,
        depthCompare: 'less',
        format: 'depth24plus'
    }
})
```

我们在创建渲染管线的配置项中，增加 `depthStencil(深度模板)`。

这里强调一下 "stencil" 单词，这个单词本意确实就是 “模板”。

提到 “模板” 你可能第一时间想到的是单词 "templete"，但在 WebGL/WebGPU 中 模板 所用单词就是 stencil。



<br>

> templete 和 sentcil 都可以被翻译为模板，那它们的细节差异是什么呢，我特意查了一下百度翻译：
>
> sentcil：(印文字或图案用的)模板、(用模板印的)文字或图案
>
> templete：模版模式、样板
>
> 可以看出 "templete" 用在一些比较通用领域，而 "sentcil" 强调图案印刷方面，更加接近图像渲染。



<br>

下面介绍一下 `depthStencil` 的属性配置。

1. depthWriteEnabled：是否开启深度写入(对比)

2. depthCompare：深度比较的方式(内置的深度比较函数名)，"less" 是"较小"的意思

   > 注意：此时使用的是 标准化设备坐标系(NDC)，z 轴取值范围为 0 - 1，越靠近屏幕其值越接近于 0
   >
   > 而上面配置的 "less(较小的)" 就是说：标准化设备坐标系 z 值越小，则三角形越靠前
   >
   > 与之对应的还有其他可选值：equal(相同)、greater(较大的)、less-equal(小于等于)、greater-equal(大于等于)、not-equal(不等于)、never(从不)、always(总是)

   > 你会发现上面那些值实际上就是数学比较符号中的 =、>、<、>=、<=、!= ...
   >
   > 很多文章中会把 depthCompare 称呼为 "深度测试"，当然本文中我把它称呼为 "深度比较"，不过都是同一个意思。

3. format：深度纹理的格式

> 除了上述 3 个属性外，还有其他可选配置属性：stencilFront(正面操作配置项)、stencilBack(背面操作配置项)、stencilReadMask(默认值 0xFFFFFF)、stencilWriteMask(默认值 0xFFFFFF)、depthBias(深度基数，默认为 0)、depthBiasSlopeScale(深度偏移坡度缩放比例，默认为 0)、depthBiasClamp(深度偏移收窄，默认值为 0)

> 深度附件的配置项非常多，目前可以先不用关心上面每一项的具体用法，只记住最基础的那 3 个即可。



<br>

```
const renderPassDescriptor: GPURenderPassDescriptor = {
    colorAttachments: [{...}],
    depthStencilAttachment: {
        view: depthTexture.createView(),
        depthClearValue: 1.0,
        depthLoadOp: 'clear',
        depthStoreOp: 'store'
    }
}
const passEncoder = commandEncoder.beginRenderPass(renderPassDescriptor)
```

> 我们给渲染通道新增属性 `depthStencilAttachment(深度模板附件)`



<br>

> 在上述配置项中 "depthLoadOp"、"depthStoreOp" 都出现了 "Op"，这个 "Op" 是单词 "operate" 的缩写，而 "operate" 单词翻译为 “操作/操纵/运营/经营”

> 当你知道 "Op" 是操作，那么很自然 "depthLoadOp" 翻译为 "深度加载操作"、"depthStoreOp" 为 "深度存储操作"。



<br>

学习 WebGPU 很重要一项内容就是遇到陌生单词要去查看它的翻译、记忆背诵、大声朗读出这个单词。

遇到一些缩写也要尽量去搞明白它是什么单词的缩写。



<br>

> 再次表达我个人写作观点：我十分讨厌一些技术文章中出现大量单词、缩写。
>
> 这些单词或缩写对新手来说非常不友好，单词还可以通过翻译工具查询含义，但缩写只会让人懵逼，只能硬着头皮学习。这些单词或缩写就不能使用中文词汇表达吗？
>
> 例如之前我看技术大佬 四季留歌 写的一篇文章：WebGPU 中消失的 FBO 和 RBO
>
> 当时文章看了一半，我都不知道它说的 FBO/RBO 究竟是什么。
>
> 后来查阅了一些其他资料，才知道：
>
> FBO：frame buffer object 的缩写，就是 "帧缓冲对象"
>
> RBO：render buffer object 的缩写，就是 "渲染缓冲对象"
>
> 当然还有 VBO：vertex buffer object，顶点缓冲对象

> 技术大佬们，请不要在中文教程中用英文缩写来劝退新手们。



<br>

前面铺垫了这么多，终于该讲本文的主题：渲染出一个多面体。



<br>

先看一下本文示例最终结果：

![simple_diamond_01.jpg](https://raw.githubusercontent.com/puxiao/webgpu-tutorial/main/imgs/simple_diamond_01.jpg)



<br>

这是个什么玩意？？？



<br>

我说它是一颗钻石，你相信吗？



<br>

看一下它的草图：

![simple_diamond_03.jpg](https://raw.githubusercontent.com/puxiao/webgpu-tutorial/main/imgs/simple_diamond_03.jpg)



<br>

信不信由你。

本示例在线演示地址：https://react-webgpu-samples.vercel.app/simple-diamond

> 我把之前本系列教程配套的示例代码仓库由 React 改为 Next.js。



<br>

**简易版钻石(多面体)实现思路：**

1. 根据它的 长、宽、一侧有几个面，可以通过数学公式算出来每一个顶点的坐标

2. 然后给每一个顶点随机生成一个颜色

3. 接下来按照 连续相邻的三角形、且每个三角形按照逆时针方向 这 2 个原则，得到 N 组三角形的顶点坐标和顶点颜色值

4. 将 顶点坐标数组 和 顶点颜色数组 转换成 顶点缓冲区 和 颜色缓冲区

5. 接下来的操作流程和之前渲染 1 个三角形的流程几乎没啥区别了，除了要加上 深度模板缓冲

   > 也就是本文上面讲的第 3 个知识点

<br>

详细的代码，请查阅：

几何顶点：https://github.com/puxiao/react-webgpu-samples/blob/main/src/geometrys/diamond-geometry.ts

页面渲染：https://github.com/puxiao/react-webgpu-samples/blob/main/src/pages/simple-diamond/index.tsx



<br>

**思考一下：为什么最开始那张图看着不像钻石？**

答案很简单：因为没有看到钻石(多面体)的棱角，所以没有直观感受出来切面。



<br>

如果在渲染结果中加入 边框(描边)，立马就看出来它是什么了。

![simple_diamond_02.jpg](https://raw.githubusercontent.com/puxiao/webgpu-tutorial/main/imgs/simple_diamond_02.jpg)



<br>

这次看着像钻石 或 宝石了吧。



<br>

这刚好引申出来，接下来我们要学习的内容：如何绘制多个独立的模型(多面体)。

> 上图中 多面体的 面 和 边框 是 2 个独立渲染对象，他们使用相同的顶点缓冲区，一个渲染的是实心三角形，一个渲染的是线条。



<br>

本文到此结束，接下来学习 多物体资源绑定，也就是如何绘制多个模型。