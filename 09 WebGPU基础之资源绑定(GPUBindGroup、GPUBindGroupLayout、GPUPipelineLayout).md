# 09 WebGPU基础之资源绑定(GPUBindGroup、GPUBindGroupLayout、GPUPipelineLayout)

**资源绑定包含 3 个内容：资源绑定组(GPUBindGroup)、资源绑定组布局(GPUBindGroupLayout)、管线布局(GPUPipelineLayout)**



<br>

> 有一些文章会把 pipeline 翻译为 管道，在本文中统一使用 管线。



<br>

**资源(Resource)？**

这里的资源是指以下 4 种类型的资源：

1. GPUBuffer(缓冲区)

2. GPUTextureView(纹理视图)

   > 我们无法直接读取 纹理(GPUTexture) 中的数据，只能通过 纹理视图(GPUTextureView) 来访问纹理数据。

3. GPUExternalTexture(外部视频纹理)

4. GPUSampler(纹理采样器)



<br>

**绑定(Bind)？**

这里的绑定是指将上述 4 种类型的资源(实例对象) 组合在一起，同时设定如何在着色阶段使用这些资源。

在绑定(组合)这些资源时：

1. 会给每一个资源(实例) 添加一个索引编号
2. 明确指出该资源的具体类型



<br>

**布局(Layout)？**

这里的布局分别暗含以下 2 个意思：

1. 对于 GPUBindGroupLayout(资源绑定组布局) 中的 布局(Layout) 是指 `资源的分布情况与数据格式`。
2. 对于 GPUPipelineLayout(管线布局) 中的 布局(Layout) 是指 `分布在哪个阶段使用`。



<br>

**所谓 “资源绑定” 就是指将若干个资源以某种形式组合在一起，交给着色器，让着色器可以在某些特定的渲染阶段使用这些资源。**



<br>

下面开始具体的学习。

<br>

#### GPUBindGroup：资源绑定组

可以通过 GPUDevice 的 .createBindGroup() 方法来创建一个 GPUBindGroup 实例。

```
createBindGroup(
  descriptor: GPUBindGroupDescriptor
): GPUBindGroup;

// createBindGroup() 方法中需要的配置参数对象
interface GPUBindGroupDescriptor
  extends GPUObjectDescriptorBase {
  layout: GPUBindGroupLayout;
  entries: Iterable<GPUBindGroupEntry>;
}

// entries 值中可枚举的对象的类型定义
interface GPUBindGroupEntry {
  binding: GPUIndex32;
  resource: GPUSampler | GPUTextureView | GPUBufferBinding | GPUExternalTexture;
}

//GPUBufferBinding的类型定义
interface GPUBufferBinding {
  buffer: GPUBuffer;
  offset?: GPUSize64;
  size?: GPUSize64;
}
```

> 以上类型定义来源于 `@webgpu/types`  0.1.13 。



<br>

创建一个 GPUBindGroup 实例的伪代码如下：

```
const bindGroup = device.createBindGroup({
  layout: xxxx,
  entries: xxxx
})
```



<br>

**layout**：值类型为 GPUBindGroupLayout，用于定义以下 3 个方面

1. 索引排序值
2. 在着色器哪个阶段使用
3. 资源的数据格式

> 我建议你先去看本文下面关于 GPUBindGroupLayout 的讲解，看完后再回到这里。



<br>

**entries**：值类型为 可枚举的对象，且被枚举的每一项值类型都为 `{ binding: GPUIndex32, resource: GPUSampler | GPUTextureView | GPUBufferBinding | GPUExternalTexture }`

entries 用于 “绑定” 那些真实的资源。



<br>

> 补充：像 GPUIndex32、GPUSize32、GPUSize64 这些都是 @webgpu/types 中定义的一些 number 类型，从他们的名字上可以看出来他们要表达的信息。
>
> 1. index：索引值，暗含的信息为 该值应该是相对唯一的。
> 2. size：大小 或 单数数量，暗含的信息为 该值一定是个无符号整数(非负整数)。
> 3. `32`：32 位浮点数
> 4. `64`：64 位浮点数
>
> 和这个套路类似的还有其他类型，例如：GPUShaderStageFlags、GPUSignedOffset32、GPUSampleMask 等，名字背后都暗含某些特定的 number 取值。



<br>

#### GPUBindGroupLayout：绑定组布局

GPUBindGroupLayout 定义了 GPUBindGroup.entries 中所绑定的资源在着色器阶段的可访问性。

> 补充提醒：GPUBindGropu.entries 这个属性属于 "内部插槽"，也就是内部定义的私有属性，我们无法直接访问。

GPUBindGroupLayout 实例是通过 GPUDevice 实例的 .createBindGroupLayout() 创建的。

```
createBindGroupLayout(
  descriptor: GPUBindGroupLayoutDescriptor
): GPUBindGroupLayout;

interface GPUBindGroupLayoutDescriptor
  extends GPUObjectDescriptorBase {
  entries: Iterable<GPUBindGroupLayoutEntry>;
}

interface GPUBindGroupLayoutEntry {
  binding: GPUIndex32;
  visibility: GPUShaderStageFlags;
  buffer?: GPUBufferBindingLayout;
  sampler?: GPUSamplerBindingLayout;
  texture?: GPUTextureBindingLayout;
  storageTexture?: GPUStorageTextureBindingLayout;
  externalTexture?: GPUExternalTextureBindingLayout;
}
```



<br>

从类型定义上我们就可知道，创建一个 GPUBingGroupLayout 的伪代码如下：

```
const groupLayout = device.createBindGroupLayout({
  extries: [ { binding: xx, visibility: xx, ...}, ... ]
})
```



<br>

**extries：可枚举对象，需要和 GPUBindGroup.entries 中的资源完全呼应**

> 只要提到可枚举对象，一般情况下我们都会使用 数组(array)，但 JS 中除了 array，可枚举对象还包括 Map、Set。请注意 “可枚举” 除了需要可以遍历外，还需要拥有 .next() 方法，本质上是一个 “链” 的数据结构。



<br>

**GPUBindGroupLayoutEntry：被枚举项的属性和含义**

**binding**：指定所绑定资源的索引(排序)值，一定要和 GPUBindGroup.entries 中对应的资源的 binding  值相同。

**visibility**：指定所绑定资源在着色器哪个阶段可见(可用)。它的取值只能是 GPUShaderStage 中定义好的 3 种情况：

1. GPUShaderStage.VERTEX：实际值为 0x1，表示 顶点着色阶段
2. GPUShaderStage.FRAGMENT：实际值为 0x2，表示 片元着色阶段
3. GPUShaderStage.COMPUTE：实际值为 0x4，表示 管线的计算阶段

visibility 可以同时设置为 多个阶段可见，例如：

```
{ visibility: GPUShaderStage.VERTEX | GPUShaderStage.FRAGMENT, ... }
```



<br>

以上 2 个属性 binding、visibility 都是必填项，剩下的 5 个可选项分别是 buffer、sampler、texture、storageTexture、externalTexture：

1. 它们分别对应 5 种资源类型的布局(layout)：GPUBufferBindingLayout、GPUSamplerBindingLayout、GPUTextureBindingLayout、GPUStorageTextureBindingLayout、GPUExternalTextureBindingLayout

2. 它们需要和 GPUBindGroup.entries 中对应的资源逐个呼应

3. 它们是互斥的，你只能从它们当中选择设置其中 1 个

   > 这似乎是一句废话



<br>

**资源种类补充说明：4 种？5种？**

本文在介绍 GPUBindGroup 时说它可以将 4 种类型的资源组合绑定在一起。

刚才又说 GPUBindGroupLayout 有 5 种资源类型设定。

这是因为 GPUBindGroupLayout 中的 texture(GPUTexture) 和 stroageTextrue(存储型纹理) 他们都是通过 GPUTextureView(纹理视图) 访问数据的。

<br>

 所谓 存储型纹理 可以将其理解为 GPUTexture 的一种特殊形式。

> 更加深入点的我也不清楚，暂时先这样理解吧。



<br>

**5种类型的TS值类型：**

**buffer：**

```
interface GPUBufferBindingLayout {
  type?: "uniform" | "storage" | "read-only-storage";
  hasDynamicOffset?: boolean;
  minBindingSize?: GPUSize64;
}
```

**sampler：**

```
interface GPUSamplerBindingLayout {
  type?: "filtering" | "non-filtering" | "comparison";
}
```

**texture：**

```
interface GPUTextureBindingLayout {
  sampleType?: "float" | "unfilterable-float" | "depth" | "sint" | "uint";
  viewDimension?: GPUTextureViewDimension;
  multisampled?: boolean;
}
```

**storageTexture：**

```
interface GPUStorageTextureBindingLayout {
  access?: "write-only";
  format: GPUTextureFormat;
  viewDimension?: GPUTextureViewDimension;
}
```

> 请注意目前 access 的值只能是 "write-only"

**externalTexture：**

```
interface GPUExternalTextureBindingLayout {}
```

> 请注意 externalTexture 目前的值类型为 { }，意味着它没有任何一种 绑定类型(Binging Type) 标记。



<br>

创建一个 GPUBindGroupLayout 的一个简单示例：

```
const layout = device.createBindGroupLayout({
    entries: [
        { binding:1, visibility: GPUShaderStage.VERTEX, sampler: { type: 'filtering' } }
    ]
})
```



<br>

关于这 5 种资源布局设置还有非常多限制细节，这里我们暂时先不做深入了解。

后期我们真正开始写完整示例代码时再做细致讲解，目前我们只是先过一遍基础的知识和概念。



<br>

#### GPUPipelineLayout：管线布局

> 由于目前还没有学习 WebGPU 管线相关的知识点，所以下面的内容理解起来会有点吃力。

GPUPipelineLayout(管线布局) 定义了在 setBindGroup 中命令编码期间设置的所有 GPUBindGroup 对象的资源与 GPURenderEncoderBase.setPipeline 或 GPUComputePassEncoder.setPipeline 设置的管线着色器之间的映射 。

> 补充：setBindGroup 是我们后面才会学到的一个知识点。



<br>

> 下面这段话来自 四季留歌 的文章：https://segmentfault.com/a/1190000040719295

一种资源(GPUBuffer、GPUTextureView、GPUExternalTexture、GPUSampler)的完整绑定地址可以通过以下 3 个特性共同定位：

1. 着色器的阶段：顶点、片元或计算阶段来指定资源在哪个阶段的可见性
2. 绑定组 id：即 setBindGroup 方法的参数
3. 绑定 id：即 entry 的 bingding 属性

以上 3 者共同构建成一种资源在着色器中的地址，这个地址又可以称作管线的 绑定空间(binging space)。



<br>

对多个渲染管线(GPURenderPipeline) 或 计算管线(GPUComputePipeline) 使用相同的管线布局(GPUPipelineLayout)就能保证在切换管线时不需要在内部重新绑定任何资源。



<br>

GPUPipelineLayout 的预期用途是将最常见和最不频繁更改的绑定组放置在布局的底部，这意味着它们应拥有较低的绑定组索引编号，例如 0 或 1。

绑定越频繁的绑定组(GPUBindGroup)应放置在布局的顶部，它们的索引值应该越高。

通过这个策略来降低 CPU 开销。



<br>

#### 创建 GPUPipelineLayout 实例

一个 GPUPipelineLayout 实例是通过 GPUDevice 的 .createPipelineLayout() 来创建的。

```
createPipelineLayout(
  descriptor: GPUPipelineLayoutDescriptor
): GPUPipelineLayout;
  
interface GPUPipelineLayoutDescriptor
  extends GPUObjectDescriptorBase {
  bindGroupLayouts: Iterable<GPUBindGroupLayout>;
}
```

> 一个 GPUPipelineLayout 中的 bindGroupLayouts 可以包含 N 个 GPUBindGroupLayout。



<br>

**一个极其简单的示例**：

```
const layout = device.createBindGroupLayout({
    entries: [
        { binding:1, visibility: GPUShaderStage.VERTEX, sampler: { type: 'filtering' } }
    ]
})

const bindGroup = device.createBindGroup({
    layout,
    entries: [ { binding: 1, resource: sampler } ]
})

const pipelineLayout = device.createPipelineLayout({
    bindGroupLayouts: [layout]
})
```

> 从上面这个简单示例可以看到：我们创建的 GPUBindGroupLayout 实例同时被 GPUBindGroup 和 GPUPipelineLayout 所使用。



<br>

如果对 管线布局(GPUPipelineLayout) 还是一脸迷糊，没有关系，接下来我们就要开始学习 着色器和管线。

等这两个模块学完后再来理解管线布局就会相对容易一些。