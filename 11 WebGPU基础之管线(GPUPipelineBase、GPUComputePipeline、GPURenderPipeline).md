# 11 WebGPU基础之管线(GPUPipelineBase、GPUComputePipeline、GPURenderPipeline)

**WebGPU 中的管线是指在 GPU 中可进行编程的管线的具体统称，主要分为计算管线(GPUComputePipeline)和渲染管线(GPURenderPipeline)。**



<br>

在学习 WebGPU 中的管线之前，我们先学习(回顾)一下图形学中的管线渲染。



<br>

> 以下内容来源于 闫令琪 老师 现代计算机图形学入门 第 8 课 第 32 分钟后开始讲解的内容。
>
> https://www.bilibili.com/video/BV1X7411F744?p=8



<br>

#### 图形管线/实时渲染管线(Real-time Rendering)

**所谓 图形学 是指：将数学模型转化为图像的过程**

> 所谓 计算机视觉 是指：将图像转化为数学模型的过程
>
> 图形学与计算机视觉属于互逆的关系。
>
> VR、XR 这些属于 图形学与计算机视觉 互相结合的应用场景。



<br>

图形管线 是一种比较传统的称呼，实际上 实时渲染管线 这个词更加贴合含义。

> 在本文中 渲染管线 和 管线渲染 是同一个意思。



<br>

**所谓 渲染管线 就是指将一个 3D 场景(也就是数学模型)转化成一张图片(屏幕上我们能够看到的图像)所经历的整个过程。**



<br>

> 下面部分是我自己编的，并不是 闫令琪 老师讲的。
>
> 假设工厂要生产一个商品，那么工厂会为此搭建一条生产流水线，第 1 道工序完成后将半成品交给第 2 道工序，第 2 道工序完成后再转给下一道工序，直至经过 N 道工序后商品最终被生产完成。
>
> 假设要将 A 点的石油通过石油管道输送到 E 点，在输送过程中石油需要先经过 B点、然后 C点 和 D点。
>
> 以上 2 个场景中的 流水线 和 运输管道 结合一下就是我们要学习的 图形学中的 管线。
>
> 管线(pipeline) 是一个比喻，指在达到某个目标结果的过程中所需要经历的若干阶段。



<br>

> 额外补充：pipeline(管道、管线) 这个词并不是图形学中特有的，例如 MongoDB 数据库中的查询使用的也是 pipeline，暗含 “逐层过滤或处理” 的意思。在 MongoDB 中通常将 pipeline 翻译为 “管道查询” 或 “聚合查询”。



<br>

**渲染管线流程图：**

![pipeline](https://puxiao.com/webgpu_tutorial/imgs/pipeline.jpg)



<br>

**渲染管线的几点补充：**

1. 在 顶点处理 阶段，发生的是 顶点着色

2. 在 片元处理 阶段，发生的是 像素着色

3. 无论是 顶点着色 还是 像素着色，对于 着色器 中的代码而言，它们都是统一的、通用的。

   换句话说就是：你只需要针对一个顶点或像素来编写代码，而这些代码会应用到全部的顶点或像素中。

4. 你可以把 纹理 看作是一种特殊的着色器，因为纹理本质上就是更改(决定)了 像素着色 的结果。

> 所谓 图形高手 比拼的就是看谁写的着色器更好、更酷炫。



<br>

以上文字中出现的 “像素” 实际上是指 fragment(片元)，我们可以将一个片元理解为一个像素。

片元也可以称呼为 片段，因此 片元着色器、片段着色器、像素着色器 本质上是同一个事情，只是每个人习惯称呼不同而已。



<br>

关于图形学中的渲染管线讲解完毕，接下来开始学习 WebGPU 中的管线。



<br>

**特别强调**：把 pipeline 翻译为 管道  或 管线 都是可以的，我个人习惯使用 管线 这个词。



<br>

#### WebGPU 中的管线(pipeline)

**所谓 WebGPU 中的管线更加精准的称呼应该是：管线配置。**

是指允许你进行 GPU 着色编程的几个关键阶段中所添加的着色器与固定模式状态的总称。



<br>

从 JS 类的定义角度来看，在 WebGPU 中一共有 3 种管线相关的类：

1. GPUPipelineBase：基础管线
2. GPUComputePipeline：继承于 基础管线的 计算管线
3. GPURenderPipeline：继承于 基础管线的 渲染管线

但是从实际使用的角度来看，GPUPipelineBase 仅仅是作为 GPUComputePipeline、GPURenderPipeline 的父类，且无法被单独创建。

所以，我们可以这样说：在 WebGPU 中存在 2 种管线

1. GPUComputePipeline：计算管线
2. GPURenderPipeline：渲染管线



<br>

在内部，WebGPU 会悄悄地将这些 固定模式(例如 混合模式) 转换成对应目标平台所支持的着色器代码。

> 混合对应的英文单词为 blend。

最终你自己编写的着色器 加上 固定模式状态对应的着色器 共同决定了该阶段的渲染结果。



<br>

#### GPUPipelineBase：基础管线

GPUPipelineBase 是 GPUComputePipeline 和 GPURenderPipeline 的父类。

它只有一个方法：getBindGroupLayout()

```
interface GPUPipelineBase {
  getBindGroupLayout(
    index: number
  ): GPUBindGroupLayout;
}
```



<br>

**getBindGroupLayout()**：

获取当前管线中指定索引对应的绑定组布局(GPUBindGroupLayout)



<br>

> GPUBindGroupLayout(绑定组布局)：作为创建 GPUBindGroup 配置项 layout 的值，其自身为一个可迭代对象(例如 一个数组)，每一个元素包含：
>
> 1. binding：所绑定资源的索引排序值
> 2. visibility：在哪个管线中可见(可用)

> 特别强调一下 getBindGroupLayout() 中的参数 index 是指 GPUBindGroupLayout 对应的索引值，和上面提到的 "binding" 并没有任何关系。



<br>

#### GPUComputePipeline：计算管线

GPUComputePipeline 更加准确的解释应该是：负责控制计算着色器的阶段

GPUComputePipeline 可以在 GPUComputePassEncoder 中使用。

> 准确来说是作为 GPUComputePassEncoder 的 .setPipeline() 方法中的参数使用。



<br>

> 注：目前我们还没有学习过 GPUComputePassEncoder，这里先简单介绍一下。
>
> GPUComputePassEncoder：计算通道编码器，用来将我们所创建计算管线进行转码，并提交给 GPU 来进行相关执行。
>
> 当然你可能会猜到，与之对应的还有一个 GPURenderPassEncoder(渲染通道编码器)。



<br>

**创建一个GPUComputePipeline实例**

通过 GPUDevice 实例的 .createComputePipeline() 方法可以创建一个 GPUComputePipeline 实例。

```
createComputePipeline(
  descriptor: GPUComputePipelineDescriptor
): GPUComputePipeline;

interface GPUComputePipelineDescriptor
  extends GPUPipelineDescriptorBase {
  compute: GPUProgrammableStage;
}

interface GPUPipelineDescriptorBase
  extends GPUObjectDescriptorBase {
  layout?: GPUPipelineLayout;
}

interface GPUProgrammableStage {
  module: GPUShaderModule;
  entryPoint: string;
  constants?: Record<
    string,
    GPUPipelineConstantValue
  >;
}
```



<br>

与此同时，也可以通过 GPUDevice 实例的 .createComputePipelineAsync() 以异步的形式创建一个 GPUComputePipeline 实例。

```
createComputePipelineAsync(
  descriptor: GPUComputePipelineDescriptor
): Promise<GPUComputePipeline>;
```



<br>

在 WebGPU 官方文档中提到：建议使用异步的方式 即通过 .createComputePipelineAsync() 来创建 GPUComputePipeline，因为这种方式可以避免阻塞管道编译中的相关操作。

> 准确来说是避免阻塞 队列时间线(queue timeline)单元上的相关操作。



<br>

无论是同步还是异步，他们所需要的参数是完全相同的，都是：{ compute: xxx, layout?: xxx } 。

接下来我们讲述一下 必填参数 compute 与可选参数 layout。



<br>

**layout?: GPUPipelineLayout**

用于指定该 GPUComputePipeline 的 管线布局(GPUPipelineLayout)。

**compute: GPUProgrammableStage**

用于指定该 GPUComputePipeline 用户提供的 GPUSaderModule 的入口点。



<br>

**GPUProgrammableStage：可编程阶段**

```
type GPUPipelineConstantValue = number;

interface GPUProgrammableStage {
  module: GPUShaderModule;
  entryPoint: string;
  constants?: Record<
    string,
    GPUPipelineConstantValue
  >;
}
```

1. module：所对应的 GPUShaderModule(着色器调试模块)
2. entryPoint：入口点标记字符，其值是由我们来设定的任意字符串
3. constants：用于定义 常量值的集



<br>

> 关于 TypeScript 中 Record 的补充介绍：
>
> Record 是 TypeScript 内置的映射类型之一，其作用是将选定的属性名对应的值全部转化为指定类型。
>
> 换句话说 Record 是用于改变和约束属性名和属性值的。
>
> 在上面代码中的 Record< string, GPUPipelineConstantValue> 的意思是说：对象的属性名必须为字符串，其值必须是 GPUPipelineConstantValue(number) 类型。



<br>

**创建一个 GPUComputePipeline 的简单示例：**

```
const computePipeline = device.createComputePipeline({
    layout: pipelineLayout,
    compute: {
        entryPoint: 'computeMain',
        module: shaderModule,
        constants: {
            value: 0
        }
    }
})
```

> 请注意 entryPoint 的值只要是字符串即可，但最好让它具有某些字面意义。



<br>

#### GPURenderPipeline：渲染管线

GPURenderPipeline 更加准确的解释应该是：负责控制顶点和片元着色器的阶段状态

GPURenderPipeline 可以在 GPURenderPassEncoder、GPURenderBundleEncoder 中使用。

> 准确来说是作为 GPURenderPassEncoder 的 .setPipeline() 方法中的参数使用。
>
> GPURenderBundleEncoder 第一次出现，暂时我们把它理解为：渲染打包编码器



<br>

**渲染管线(GPURenderPipeline)的 8 个细分渲染阶段(状态)：**

1. 获取顶点：由 GPUVertexState.buffers 控制
2. 顶点着色器：由 GPUVertexState 控制
3. 原始组装：由 GPUPrimitiveState 控制
4. 光栅化：由 GPUPrimitiveState、GPUDepthStencilState、GPUMultisampleState 控制
5. 片元着色器：由 GPUFragmentState 控制
6. 深度模板(stencil)附件测试与操作：由 GPUDepthStencilState 控制
7. 深度测试与写入：由 GPUDepthStencilState 控制
8. 输出合并：由 GPUFragmentState.targets 控制



<br>

> 按照道理 state 应该翻译为 “状态” ，但是我将其称呼为 “阶段状态”，因为我认为这样更便于我们理解。



<br>

我们将上面 8 个阶段(状态)反向整理，得到下面这个表格。

| 阶段(状态类)         | 对应渲染阶段(render states)                    |
| -------------------- | ---------------------------------------------- |
| GPUVertexState       | 获取顶点、顶点着色器                           |
| GPUPrimitiveState    | 原始组装、光栅化                               |
| GPUDepthStencilState | 光栅化、深度模板附件测试与操作、深度测试与写入 |
| GPUMultisampleState  | 光栅化、输出合并                               |
| GPUFragmentState     | 片元着色器、输出合并                           |

> 也就是说 8 个细分阶段一共对应 5 个状态类。



<br>

> stencil 通常被翻译为：模板附件
>
> multisample 被翻译为：多重采样



<br>

**名词解释：**

1. 原始组装：简单来说就是决定空间中哪 3 个点来构成一个三角形，最终无数个三角形构成模型网格。

2. 深度模板附件：假设有 A、B 两个物体，从相机(人眼)视角来看 A 在 B 前面，如果按照默认的 深度测试 那么 A 一定会遮挡住 B。但是假设我们添加有 深度模板附件，且我们给 A 和 B 分别设置相应的深度模板附件值，那么我们就可以进行相关逻辑设定：我们可以规定 A 在 B 前面但是要求 不渲染 A 反而渲染 B。

   换句话说 所谓 深度模板附件 可以改变默认的 深度测试结果。

   > 管线渲染中的 深度测试 默认是不可以进行编程的。



<br>

#### 创建一个渲染管线实例

可以通过 GPUDevice 的 .createRenderPipeline() 来创建一个 渲染管线(GPURenderPipeline)实例。

也可以通过 .createRenderPipelineAsync() 来异步创建。

```
createRenderPipeline(
  descriptor: GPURenderPipelineDescriptor
): GPURenderPipeline;

createComputePipelineAsync(
  descriptor: GPUComputePipelineDescriptor
): Promise<GPUComputePipeline>;

interface GPURenderPipelineDescriptor
  extends GPUPipelineDescriptorBase {
  vertex: GPUVertexState;
  primitive?: GPUPrimitiveState;
  depthStencil?: GPUDepthStencilState;
  multisample?: GPUMultisampleState;
  fragment?: GPUFragmentState;
}
```

从上面可以看出，初始化一个渲染管线的配置项中 vertex (GPUVertexState) 是必填项，其他 4 项均为选填项。



<br>

假设我们把上述 5 个状态看作是 “输入(input)” 的话，那么还会对应有 “输出(output)”。

输出中会有一个新的状态：GPUColorTragetState(颜色目标状态)。

颜色目标状态(GPUColorTragetStage) 还包含 混合状态(GPUBlendState)。



<br>

#### 特别说明：

原本我们应该认真、详细讲解一下些状态：

1. GPUVertexState：顶点状态
2. GPUPrimitiveState：原始状态
3. GPUDepthStencilState：深度/模板状态
4. GPUMultisampleState：多重采样状态
5. GPUFragmentState：片元状态
6. GPUColorTragetState：颜色目标状态
7. GPUBlendState：混合状态



<br>

> 准确来说 是不是写着色器的高手，就看是不是对上面这些状态运用的熟练程度了。



<br>

但是考虑到实际上我们还未真正去编写过渲染示例。

准确来说我们目前已学习的知识点还不足以支撑走完整个 WebGPU 流程，所以我们暂且放下这些状态的学习。

**就当是我们此刻先挖一个坑，等到学习过后面几个知识点后，我们再来填这个坑。**



<br>

关于 WebGPU 管线(pipeline) 我们就姑且先学习到这里，对于管线(pipeline)以及渲染管线中的各个状态(state)，我们有一个初步印象即可。

接下来我们将学习 命令缓冲区(GPUCommandBuffer)。

