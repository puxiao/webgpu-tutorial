# 13 WebGPU基础之计算通道编码器(GPUComputePassEncoder)与渲染通道编码器(GPURenderPassEncoder)

**计算通道编码器(GPUComputePassEncoder)与渲染通道编码器(GPURendePassEncoder)都属于可编程通道编码器(GPUProgrammablePassEncoder)的子类。**



<br>

> WebGPU 中各个类的名字真的是够长的了。



<br>

重复一遍，本文要学习的 3 个类：

可编程通道编码器(GPUProgrammablePassEncoder)

以及它的 2 个子类：

1. 计算通道编码器(GPUComputePassEncoder)
2. 渲染通道编码器(GPURenderPassEncoder)



<br>

先介绍几个名词。

> 这几个名词来源于 `-牧野-` 的一篇文章：OpenGL图形渲染管线、VBO、VAO、EBO概念及用例
>
> https://blog.csdn.net/dcrmg/article/details/53556664



<br>

**VBO(Vertex Buffer Objects)：顶点缓冲对象**

显存中开辟一块缓存区用来存储顶点的各类属性信息，例如 顶点坐标、顶点法向量、顶点颜色等。

在渲染时可直接从 VBO 中取出顶点的各类属性数据，且由于这些信息是存储在显存中，所以不需要通过 CPU 传输(读取)数据，因此处理效率更高。



<br>

**VAO(Vertex Array Object)：顶点数组对象**

VAO 用于保存模型所有顶点数据属性的状态结合，它存储了顶点数据的格式以及顶点数据所需要的 VBO 对象的引用。



<br>

换句话说：VBO 是对顶点属性信息的绑定，而 VAO 是对很多个 VBO 的绑定。



<br>

**EBO(Element Buffer Object)：索引缓冲对象**

EBO 相当于 “顶点数组” 的概念，假设同一个顶点被多次重复调用，那么为了减少内存空间浪费，提高执行效率，可以通过该顶点的位置索引来调用该顶点，而无需对重复的顶点信息重复记录多次。



<br>

> 在一些图形学文章中 VBO 可能最频繁被提及。



<br>

回到本文正题中。



<br>

> 由于之前的文章中，我们几乎都在侧重讲解每个类的背景知识、基本概念和属性方法，缺少实际的代码编写，所以本文也不会讲解太过详细，只需对通道编码器有个大致了解即可。
>
> 让我们快速的过完一遍类的基础概念后，等到实际编写示例时，很多地方自然才能真正理解其用法。现在讲再多也很难理解。



<br>

#### GPUProgrammablePassEncoder：可编程通道编码器

作为计算通道编码器和渲染通道编码器的父类，可编程通道编码器(GPUProgrammablePassEncoder) 目前只有一个方法：setBindGroup()

**setBindGroup()：设置绑定组**

在 `@webgpu/types` 中对 setBindGroup() 方法进行了 2 种形式的定义。

```
setBindGroup(
    index: GPUIndex32,
    bindGroup: GPUBindGroup,
    dynamicOffsets?: Iterable<GPUBufferDynamicOffset>
): undefined;
```

和

```
setBindGroup(
    index: GPUIndex32,
    bindGroup: GPUBindGroup,
    dynamicOffsetsData: Uint32Array,
    dynamicOffsetsDataStart: GPUSize64,
    dynamicOffsetsDataLength: GPUSize32
): undefined;
```

可以看出前两个参数是相同的，却别在于剩余参数，即以哪种形式来定义绑定组中数据的偏移量。



<br>

#### GPUComputePassEncoder：计算通道编码器

计算通道编码器 顾名思义就是用来对 计算通道管线进行编码的。

它一共有 4 个方法，我们依次来说一下。



<br>

**setPipeline(pipeline:GPUComputePipeline)：设置计算管线**

```
setPipeline(
    pipeline: GPUComputePipeline
): undefined;
```



<br>

**dispatch()：使用当前 GPUComputePipeline 执行的调度工作**

```
dispatch(
    workgroupCountX: GPUSize32,
    workgroupCountY?: GPUSize32,
    workgroupCountZ?: GPUSize32
): undefined;
```

它的 3 个参数分别对应的含义为：

1. workgroupCountX：要调度的工作组网格的 x 维度
2. workgroupCountY：要调度的工作组网格的 y 维度
3. workgroupCountZ：要调度的工作组网格的 z 维度



<br>

**dispatchIndirect()：使用从参数 GPUBuffer 中得到的数据与当前计算管线(GPUComputePipeline) 一起执行**

> indirect 单词的本意为：间接的、迂回的、不直截了当的

```
dispatchIndirect(
    indirectBuffer: GPUBuffer,
    indirectOffset: GPUSize64
): undefined;
```



<br>

**endPass(): 结束当前通道编码**

```
endPass(): undefined
```

一旦 .endPass() 方法被执行后，当前的 计算通道编码器就不能再使用了。

> 这个方法的作用和命令编码器(GPUCommandEncoder) 的 .finish() 非常相似。



<br>

#### GPURenderPassEncoder：渲染通道编码器

渲染通道编码器是由 命令编码器(GPUCommandEncoder) 的 .beginRenderPass() 方法创建的。

> 渲染通道编码器 相对而言比较复杂。

接下来，我们看一下 .beginRenderPass() 方法所需要的参数：

```
beginRenderPass(
    descriptor: GPURenderPassDescriptor
): GPURenderPassEncoder;

interface GPURenderPassDescriptor
  extends GPUObjectDescriptorBase {
  colorAttachments: Iterable<GPURenderPassColorAttachment>;
  depthStencilAttachment?: GPURenderPassDepthStencilAttachment;
  occlusionQuerySet?: GPUQuerySet;
  timestampWrites?: GPURenderPassTimestampWrites;
}
```

> Attachment 单词的翻译为 “附件”



<br>

**colorAttachments(必填项)：颜色附件**

```
colorAttachments: Iterable<GPURenderPassColorAttachment>;

...

interface GPURenderPassColorAttachment {
  view: GPUTextureView;
  resolveTarget?: GPUTextureView;
  clearValue?: GPUColor;
  loadOp: GPULoadOp;
  loadValue?: GPULoadOp | GPUColor;
  storeOp: GPUStoreOp;
}
```

1. view：颜色附件所要输出到的纹理视图(GPUTextView)
2. resolveTarget：描述将为此颜色附件输入到纹理子资源的 纹理视图(GPUTextureView)
3. celarValue：执行渲染通道之前要清除的颜色值，默认为(r:0, g:0, b:0, a:0)
4. loadValue：执行渲染通道之前在 视图 上执行的加载操作
5. storeOp：执行渲染通道后对 视图 执行的存储操作



<br>

**depthStencilAttachment(可选)：深度/模板附件**

> 注：深度对应的英文单词为 depth、模板对应的英文单词为 stencil

```
depthStencilAttachment?: GPURenderPassDepthStencilAttachment;

...

interface GPURenderPassDepthStencilAttachment {
  view: GPUTextureView;
  depthClearValue?: number;
  depthLoadOp?: GPULoadOp;
  depthLoadValue?: GPULoadOp | number;
  depthStoreOp?: GPUStoreOp;
  depthReadOnly?: boolean;
  stencilClearValue?: GPUStencilValue;
  stencilLoadOp?: GPULoadOp;
  stencilLoadValue?: GPULoadOp | GPUStencilValue;
  stencilStoreOp?: GPUStoreOp;
  stencilReadOnly?: boolean;
}
```

1. view：深度模板附件所要输出到的纹理视图(GPUTextView)，也可以从此深度模板附件中读取该纹理视图
2. depthClearValue：执行渲染过程之前清除的 view 的深度组件的值，取值范围为 0.0 ~ 1.0
3. depthLoadOp：在渲染过程之前要在 view 的深度组件上执行的加载操作
4. depthStoreOp：执行渲染通道后对 view 深度组件执行的存储操作
5. depthReadOnly：指示 view 的深度组件仅为只读模式
6. stencilClearValue：执行渲染通道之前清除 view 的模板组件的值
7. stencilLoadOp：执行渲染通道前要在 view 的模板组件上执行的加载操作
8. stencilStoreOp：执行渲染通道后在 view 的模板组件上执行的存储操作
9. stencilReadOnly：指示 view 的模板组件仅为只读模式



<br>

我们再看一下 GPULoadOp 和 GPUStoreOp 的类型定义：

```
type GPULoadOp = "load" | "clear";
type GPUStoreOp = "store" | "discard";
```

GPULoadOp：

1. "load"：将此附件的现有值加载到渲染通道中
2. "clear"：将此附件的明确值加载到渲染通道中

> 注：通常在手机设备的 GPU 中 "clear" 的成本要比 "load" 小很多，在 PC 设备的 GPU 中两者相差不大，建议使用 "clear"。

GPUStoreOp：

1. "store"：存储此附件的渲染通道的结果值
2. "discard"：丢弃此附件的渲染通道的结果值



<br>

**补充：GPURenderPassLayout(渲染通道布局)**

```
interface GPURenderPassLayout
  extends GPUObjectDescriptorBase {
  colorFormats: Iterable<GPUTextureFormat>;
  depthStencilFormat?: GPUTextureFormat;
  sampleCount?: GPUSize32;
}
```

渲染通道布局即当前通道渲染目标的布局，它决定了 通道 与 渲染管线的兼容性。

> 我也不是太理解它的具体含义和用法，暂时不做过多介绍。



<br>

上面讲的是渲染通道编码器(GPURenderPassEncoder) 的创建所牵涉的一些参数，接下来讲解它的其他一些方法。



以下是和 **绘制** 相关的方法：

<br>

**setPipeline()：设置当前的渲染管线**

```
setPipeline(
    pipeline: GPURenderPipeline
): undefined;
```



<br>

**setIndexBuffer()：设置当前索引缓冲区**

```
setIndexBuffer(
    buffer: GPUBuffer,
    indexFormat: GPUIndexFormat,
    offset?: GPUSize64,
    size?: GPUSize64
): undefined;
```



<br>

**setVertexBuffer()：设置给定槽(slot)的当前顶点缓冲区**

```
setVertexBuffer(
    slot: GPUIndex32,
    buffer: GPUBuffer,
    offset?: GPUSize64,
    size?: GPUSize64
): undefined;
```



<br>

**draw()：绘制图元**

```
draw(
    vertexCount: GPUSize32,
    instanceCount?: GPUSize32,
    firstVertex?: GPUSize32,
    firstInstance?: GPUSize32
): undefined;
```



<br>

**drawIndexed()：绘制索引图元**

```
drawIndexed(
    indexCount: GPUSize32,
    instanceCount?: GPUSize32,
    firstIndex?: GPUSize32,
    baseVertex?: GPUSignedOffset32,
    firstInstance?: GPUSize32
): undefined;
```



<br>

**drawIndirect()：使用从 GPUBuffer 中读取的参数来绘制图元**

```
drawIndirect(
    indirectBuffer: GPUBuffer,
    indirectOffset: GPUSize64
): undefined;
```



<br>

**drawIndexedIndirect()：使用从 GPUBuffer 读取的参数来绘制索引图元**

```
drawIndexedIndirect(
    indirectBuffer: GPUBuffer,
    indirectOffset: GPUSize64
): undefined;
```



<br>

以下是和 **光栅化状态** 相关的方法：

**setvViewport()：将光栅化阶段使用的视口设置为从标准设备坐标线性映射到视口坐标**

```
setViewport(
    x: number,
    y: number,
    width: number,
    height: number,
    minDepth: number,
    maxDepth: number
): undefined;
```



<br>

**setScissorRect()：设置在光栅化阶段使用的裁剪矩形**

当转换为视口坐标后，任何出在剪裁矩形以外的片元都将被丢弃。

```
setScissorRect(
    x: GPUIntegerCoordinate,
    y: GPUIntegerCoordinate,
    width: GPUIntegerCoordinate,
    height: GPUIntegerCoordinate
): undefined;
```



<br>

**setBlendConstant()：设置与 constant 一起使用的常量混合颜色(含 alpha 值)**

```
setBlendConstant(
    color: GPUColor
): undefined;
```



<br>

**setStencilReference()：设置模板测试期间使用的模板参考值**

```
setStencilReference(
    reference: GPUStencilValue
): undefined;
```



<br>

以下是和 **查询** 相关的方法：

**beginOcclusionQuery()：开始查询**

```
beginOcclusionQuery(
    queryIndex: GPUSize32
): undefined;
```



<br>

**endOcclusionQuery()：结束查询**

```
endOcclusionQuery(): undefined;
```



<br>

以下是和 **绑定** 相关的方法：

**executeBundles()：执行前先记录到给定 渲染绑定束(GPURenderBundle) 集合中的命令，作为此渲染通道的一部分**

```
executeBundles(
    bundles: Iterable<GPURenderBundle>
): undefined;
```



<br>

以下适合 **结束通道** 相关的方法：

**end()：结束渲染通道编码器**

用户完成记录命令后，结束当前渲染通道编码器。一旦执行该方法后，当前渲染通道编码器就不能再使用了。

```
end(): undefined;

endPass(): undefined;
```

注：`endPass()` 和 `end()` 的作用是相同的，目前官方计划将 endPass() 废弃，推荐使用 end()。



<br>

计算通道编码器与渲染通道编码器的众多方法，我们先不必去过分追查细节，因为这些方法需要大量的实际编写示例后才能慢慢掌握具体怎么使用。



<br>

再坚持一下，预计再有 2 个模块讲解完后，我们就该真正进入到实际示例编写中了。

本文到此结束，下一节我们将学习 渲染包(GPURenderBundle) 和 查询集(GPUQuerySet)。

