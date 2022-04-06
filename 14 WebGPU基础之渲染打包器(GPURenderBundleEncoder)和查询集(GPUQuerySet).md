# 14 WebGPU基础之渲染打包器(GPURenderBundleEncoder)和查询集(GPUQuerySet)

**渲染打包器(GPURenderBundleEncoder)可以记录多条渲染命令，从而得到一个渲染捆绑包(GPURenderBundle)，而渲染捆绑包可以被反复多次使用，其作用是为了提高渲染性能。**

**查询集(GPUQuerySet)可分为深度查询和时间戳查询。**



<br>

> 本文是 WebGPU 基础系列的最后(倒数) 第 2 篇文章，终于快要对 基础 部分学习完成了。



<br>

在学习 渲染打包器(GPURenderBundleEncoder) 之前，我们先看一下这个所谓的 “包” 是指什么？

**这个 “包” 就是 “捆绑包(bundles)”，“不良商家”进店捆绑消费的那个捆绑。**



<br>

**如果你把 GPURenderBundleEncoder 翻译成 渲染捆绑编码器 实际上也是可以的**。

> 出于个人偏好，本文将其翻译为 渲染打包器，而不是 渲染捆绑编码器。



<br>

> bundle 单词本身翻译为：一捆、一包、一扎、一批，在本文中我们将其翻译为 “包”，将它的复数形式 bundles 翻译为 捆绑包。
>
> 再重复一遍，本文将 bundleEncoder 翻译为 打包器，而不是 捆绑编码器。
>
> 但无论怎么翻译，其含义相同。



<br>

> 本文关于 捆绑包(Bundles) 的概念解释，来源于 `GamebabyRockSun` 的一篇文章：DirectX12（D3D12）基础教程（五）——理解和使用捆绑包，加载并使用DDS Cube Map
>
> https://blog.csdn.net/u014038143/article/details/84261509



<br>

WebGPU 中 捆绑包(Bundles) 的概念就是来源于 D3D12。



<br>

#### 关于“捆绑消费”的举例

我们试想一下这个场景：

1. 假设有一家面馆，他提供不同的面食实际上就是 煮好的面条 + 不同口味的卤
2. 去吃面的绝大多数顾客除了要一份面，还会要一瓶饮料(可乐、雪碧...)
3. 同时，很多人还会再要一根烤肠

现在每天中午去吃面的人数发生了变化：

1. 之前每天中午大约有 100 人去吃面
2. 现在每天中午大约有 1000 人去吃面

为了提高点餐效率，你觉得该怎么做？

答：实行套餐制，就是不再单点，而是改为 固定套餐。

这个套餐的内容为：白面条 + 卤(鸡丁、牛肉...) +  1 瓶饮料(可乐或雪碧) + 1 根烤肠



<br>

在允许单点的时候，饭店与顾客的对话是这样的：

服务员：请问你要什么口味的卤？

顾客：我要 鸡丁

服务员：请问你要不要饮料？

顾客：我要

服务员：请问你要可乐还是雪碧？

顾客：可乐

服务员：请问你要不要 1 根烤肠？

顾客：我要



<br>

在实行固定套餐制后，上述对话可能简化为：

服务员：哪种卤？哪种饮料？

顾客：鸡丁、可乐



<br>

> 服务员：哪种卤？哪种饮料？
>
> 顾客：鸡丁，但我不要可乐，也不要烤肠
>
> 服务员：你 滚。



<br>

这个 套餐制 现实中有个不好听的名字：**捆绑消费**

但 套餐制 在面对客流量足够大、且点餐内容相似时，确实可以提高点餐效率。



<br>

不同顾客所点的套餐变化点在于 哪种卤 和 哪种饮料，对于剩下相同的配置则进行了省略询问。

<br>

> 你是否觉得 套餐制 特别像一个函数，而 哪种卤 和 哪种饮料 是这个函数的参数。
>
> 没错，当理解到这层意思时，也就大概理解 捆绑包 的含义了。



<br>

回到 WebGPU 中。

**渲染捆绑包就是用来记录某些相对固定的渲染操作(步骤)，且每次操作仅可修改某些特定的渲染变化因素，该渲染捆绑包可以被反复使用。**

假设场景中存在 1000 个相似的立方体，且每个立方体仅仅在某些特定的属性上存在区别，那么就可以通过 渲染捆绑包 定义好 N 个渲染操作步骤，然后逐个(多次)使用该 渲染捆绑包 来对这 1000 个立方体进行渲染。



<br>

请注意：

1. **渲染捆绑包 记录的是 N 个渲染操作(渲染命令)，而不是渲染数据。**
2. **不是任何你以为的渲染操作都可以的，渲染捆绑包所支持的某些渲染操作是有限的。**

> 具体渲染捆绑包 都可记录(支持)哪些渲染操作，会在稍后讲解。



<br>

在 WebGPU 中和 捆绑包(bundles) 相关的 2 个类分别是：

1. GPURenderBundleEncoder：渲染打包器
2. GPURenderBundle：渲染捆绑包



<br>

#### GPURenderBundleEncoder：渲染打包器

渲染打包器(GPURenderBundleEncoder) 用于记录某些渲染操作命令，最终将这些操作命令通过 .finish() 函数 转化为一个 渲染捆绑包(GPURenderBundle)。



<br>

**创建渲染打包器实例**

渲染打包器是由 WebDevice 实例的 .createRenderBundleEncoder() 函数创建的。

```
createRenderBundleEncoder(
    descriptor: GPURenderBundleEncoderDescriptor
): GPURenderBundleEncoder;

interface GPURenderBundleEncoderDescriptor extends GPURenderPassLayout {
    depthReadOnly?: boolean;
    stencilReadOnly?: boolean;
}

interface GPURenderPassLayout
    extends GPUObjectDescriptorBase {
    colorFormats: Iterable<GPUTextureFormat>;
    depthStencilFormat?: GPUTextureFormat;
    sampleCount?: GPUSize32;
}
```

从上面可以看出，创建渲染打包器的参数继承于渲染通道布局(GPURenderPassLayout)，且需要额外提供 2 个参数：

1. depthReadOnly：深度组件 是否为只读 (若为 ture 即表示不会通过此渲染捆绑打包器进行修改)
2. stencilReadOnly：模板组件 是否为只读 (若为 ture 即表示不会通过此渲染捆绑打包器进行修改)

由于 GPURenderPassLayout 有一个必填参数 colorFormats，所以创建一个 渲染打包器 的示例为：

```
const renderBundler = device.createRenderBundleEncoder({
    colorFormats: ['bgra8unorm'],
    depthReadOnly: true,
    stencilReadOnly: true
})
```



<br>

**渲染打包器的定义**

再看一下 渲染打包器(GPURenderBundleEncoder) 的定义：

```
interface GPURenderBundleEncoder
  extends GPUObjectBase,
    GPUCommandsMixin,
    GPUDebugCommandsMixin,
    GPUProgrammablePassEncoder,
    GPURenderEncoderBase {

  readonly __brand: "GPURenderBundleEncoder";
  finish(
    descriptor?: GPURenderBundleDescriptor
  ): GPURenderBundle;
}
```

可以看到 渲染打包器 分别继承于：

1. 命令混合器(GPUCommandsMixin)
2. 命令调试混合器(GPUDebugCommandsMixin)
3. 可编程通道编码器(GPUProgrammablePassEncoder)
4. 渲染编码器基类(GPURenderEncoderBase)

那么自然 渲染打包器 会拥有上述这些类他们的属性和方法。



<br>

> 请注意继承的是 渲染编码器基类(GPURenderEncoderBase)，而不是渲染编码器(GPURenderEncoder)
>
> 由于 渲染编码器 也是继承于 GPURenderEncoderBase，从这个角度来讲，可以把 渲染打包器看作是 渲染编码器 的另外一种特殊存在形式。



<br>

而所谓 “记录 N 个渲染操作命令” 实际上就是去执行 上述 4 个父类中的某些方法。



<br>

**渲染打包器的 finish() 方法**

渲染打包器除继承父类的方法外，自身 只有一个方法：finish()。

通过执行该方法 完成渲染捆绑包命令序列的记录，得到 渲染捆绑包(GPURenderBundle) 实例。

```
finish(
    descriptor?: GPURenderBundleDescriptor
): GPURenderBundle;

type GPURenderBundleDescriptor = GPUObjectDescriptorBase;

interface GPUObjectDescriptorBase {
  label?: string;
}
```

> 请注意 finish() 的参数目前仅仅为普通的 GPUObjectDescriptorBase，即仅仅可配置一个 label 标记



<br>

当我们得到一个 渲染捆绑包(GPURenderBundle) 实例后，就可以在 渲染通道编码器(GPURenderPassEncoder) 的 executeBundles() 方法中使用到它。



<br>

关于 渲染打包器 和 渲染捆绑包 就先讲解到这里，等到下一阶段 去编写实际示例时，再慢慢使用。



<br>

接下来，我们将学习 WebGPU 基础部分中另外一个模块：查询集(GPUQuerySet)



<br>

> 按照 WebGPU 官方文档的介绍顺序，到了这个阶段才开始介绍 命令队列(GPUQueue) 和 查询集(GPUQuerySet)，但是由于我们在很早时候就提前学习了 命令队列(GPUQueue)，所以接下来只讲一下 查询集(GPUQuerySet)。



<br>

#### GPUQuerySet：查询集

> 这里的 set 实际上就是 JS 中的 Set 对象的概念，即：唯一值的集合

> 补充：queue 翻译为 队列，而 queries 翻译为 查询

查询集(GPUQuerySet)可用于深度查询和时间戳查询。



<br>

**创建查询集**

通过 GPUDevice 的 .createQuerySet() 可以创建一个 查询集(GPUQuerySet) 实例。

```
interface GPUQuerySet extends GPUObjectBase {
  readonly __brand: "GPUQuerySet";
  destroy(): undefined;
}

createQuerySet(
    descriptor: GPUQuerySetDescriptor
): GPUQuerySet;

interface GPUQuerySetDescriptor extends GPUObjectDescriptorBase {
    type: GPUQueryType;
    count: GPUSize32;
}

type GPUQueryType = "occlusion" | "timestamp";
```

创建查询集 参数需要的 2 个配置项：

1. type：查询集所管理的查询类型，该值为 “occlusion(遮挡查询)” 或 "timestamp(时间戳查询)"
2. count：查询集所管理的查询数量



<br>

创建一个包含 32 个遮挡查询结果的查询集 示例代码：

```
const querySet = device.createQuerySet({
    type: 'occlusion',
    count: 32
})
```



<br>

**查询类型之：遮挡查询(occlusion)**

遮挡查询仅在渲染过程中可用，用于查询通过一组绘图命令的所有片段测试的数量，包括裁剪、样本蒙版、覆盖率、模板和深度测试。

若没有任何样本通过测试则返回结果为 0。



<br>

**查询类型之：时间戳查询(timestamp)**

时间戳查询允许应用程序通过 计算通道编码器(GPUComputePassEncoder)、渲染通道编码器(GPURenderPassEncoder)、命令编码器(GPUCommandEncoder) 上调用 writeTimestamp() 方法将时间戳写入到 查询集(GPUQuerySet) 中，然后通过 .resolveQuerySet() 以 纳秒(nanoseconds) 为单位解析时间戳值到 缓冲区(GPUBuffer)。

> 注意：如果物理设备重置时间戳计数器，时间戳可能为 0。



<br>

**销毁查询集实例**

当不再需要 查询集(GPUQuerySet) 实例时，可通过执行其 .destroy() 方法对其进行销毁。



<br>

关于查询集(GPUQuerySet) 有个初步印象即可，以后再深入掌握其用法。



<br>

本文到此结束，接下来，我们再学习最后一个模块：GPU画布上下文(GPUCanvasContext)

