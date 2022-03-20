# 12 WebGPU基础之命令缓冲区(GPUCommandBuffer)与命令编码器(GPUCommandEncoder)

**命令缓冲区(GPUCommandBuffer)用于预先记录 GPU 所需要执行的 N 条命令，命令缓冲区是由命令编码器(GPUCommandEncoder)通过“编码”得到的。**



<br>

> 关于计算机中缓冲区(Buffer) 的概念和作用，在之前学习 GPUBuffer 时已经充分讲解过了，本文就不再重复。
>
> 下面直接进入本文正题。



<br>

> 关于 command 这个单词，其表达的含义为 命令 或 指令，但在本文中一概使用 “命令” 这个词语。
>
> 无论把它翻译为 命令 还是 指令都是可以的，出于个人习惯，我将其称呼为 命令。

> 因为 command 字面意思就是 命令，而 指令 的英文单词为 instruction



<br>

> 关于 指令 百度百科的解释为：
>
> 指令是指示计算机执行某种操作的命令。它由一串二进制数码(0或1)组成。  
> 一条指令通常由两个部分组成：操作码+地址码。  
> 操作码：指明该指令要完成的操作的类型或性质，如取数、做加法或输出数据等。  
> 地址码：指明操作对象的内容或所在的存储单元地址。



<br>

#### GPUCommandBuffer：GPU命令缓冲区

当我们需要让 GPU 执行某些任务，例如：设置状态、绘图、复制资源(纹理、缓冲区...) 等等，那么此时就需要通过 GPUCommandEncoder(命令编码器) 将上述那些任务的操作命令转化为一个 GPUCommandBuffer(命令缓冲区)，然后再将这个 GPUCommandBuffer(命令缓冲区) 通过 GPUQueue(命令队列) 传递给 GPU。

简而言之，他们 3 者的关系为：

1. 命令编码器(GPUCommandEncoder)可将多条需要执行的任务(命令)转化(编码)为一个 命令缓冲区(GPUCommandBuffer)
2. 命令缓冲区(GPUCommandBuffer) 的作用就是用于记录(保存) GPU 有待执行的命令
3. 命令队列(GPUQueue) 的作用是将 命令缓冲区(GPUCommandBuffer) 提交给 GPU，让 GPU 开始执行



<br>

![GPUCommandEncoder](https://puxiao.com/webgpu_tutorial/imgs/gpu_command_encoder.jpg)



<br>

请注意：GPUCommandBuffer 本身无法通过 new 或 GPUDevice.createXxxx() 这种形式进行创建实例化，它的实例化是由 GPUCommandEncoder 实例的 .finish() 创建得到的。



<br>

在学习 命令编码器(GPUCommandEncoder)之前，我们先看一下 GPUCommandsMixin。



<br>

#### GPUCommandsMixin：命令混合器

命令混合器(GPUCommandsMixin) 定义了所有编码命令的接口的通用的状态。

> “所有编码命令” 是指 GPUCommandEncoder、GPUComputePassEncoder、GPURenderBundleEncoder、GPURenderPassEncoder

> 请注意：GPUCommandsMixin 中间的单词 commands 是有 s 的。



<br>

> 将 GPUCommandsMixin 翻译为 命令混合器 纯粹是我个人的理解，实际上是否应该加上 “器” 这个字我也不太确定。



<br>

**所有编码命令的接口的通用的状态？**

按照 WebGPU 目前官方文档上的介绍，GPUCommandsMixin 内部定义的状态([[stage]]插槽)有：

1. open：编码器可用于对新命令进行编码
2. locked：被子编码器锁定，当前无法使用该编码器
3. ended：编码器已结束，无法再对新命令进行编码



<br>

如果查看 .d.ts 中的定义，会发现：

```
interface GPUCommandsMixin { }
```

???

我们可以这样去理解 GPUCommandsMixin：

1. GPUCommandsMixin 目前是所有编码器的父类
2. GPUCommandsMixin 目前仅为 WebGPU 内部使用，对于普通前端开发者而言暂时无任何有用的属性或方法
3. 不排除未来会扩展 GPUCommandsMixin 的属性或方法供我们使用



<br>

#### GPUDebugCommandsMixin：调试命令标记混合器

GPUDebugCommandsMixin 主要是用来添加调试命令标记的。

它与 GPUCommandMixin 名字相似，但二者的作用并不太相同，

他们两个并不是继承关系，GPUDebugCommandsMixin 并非继承于 GPUCommandsMixin。



<br>

> 实际上在 WebGPU 官方文档中，GPUCommandsMixin 位于第 12 章，而 GPUDebugCommandsMixin 位于第 14 章。

> 请注意：我们本文我们马上要学习的 GPUCommandEncoder 同时继承于 GPUCommandsMixin 和 GPUDebugCommandsMixin。



<br>

**GPUDebugCommandsMixin拥有的方法：**

它的实例一共有 3 个方法：

**pushDebugGroup()**：开始(添加)一个命令组的调试标记(标签)

```
pushDebugGroup(
    groupLabel: string
): undefined;
```



<br>

**popDebugGroup()**：结束(删除)最近一个由 pushDebugGroup() 开始的命令组的调试标记(标签)

```
popDebugGroup(): undefined;
```



<br>

**insertDebugMarker**：用标签标记 命令流 中的一个点

```
insertDebugMarker(
    markerLabel: string
): undefined;
```



<br>

#### GPUCommandEncoder：命令编码器

命令编码器(GPUCommandEncoder) 实例是由 GPUDevice 的 .createCommandEncoder() 方法创建的。

```
createCommandEncoder({
    label?: string;
}): GPUCommandEncoder;
```

从创建方法的 TS 定义可以看出，我们可以选择添加一个 label 属性字段，或者 不传任何参数也是可以的。

```
//不添加任何参数
const commandEncoder = device.createCommandEncoder()

//添加一个 label，方便我们后续调试时识别出究竟是哪个 命令编码器
const commandEncoder = device.createCommandEncoder({ label: 'myCommandEncoder' })
```



<br>

**命令编码器(GPUCommandEncoder)实例拥有的方法：**

拥有以下方法：

> 通过方法的名字，我们几乎都能猜出来每个方法的作用。



<br>

> pass 这个单词本意为 通过、通行证、关口 等，在本文中我们将其翻译为 通道



<br>

**beginRenderPass()**：开始对渲染通道进行编码

```
beginRenderPass(
    descriptor: GPURenderPassDescriptor
): GPURenderPassEncoder;
```

> 请注意：该方法返回值为一个 渲染通道编码器(GPURenderPassEncoder)



<br>

**beginComputePass()**：开始对计算通道进行编码

```
beginComputePass(
    descriptor?: GPUComputePassDescriptor
): GPUComputePassEncoder;
```

> 请注意：该方法返回一个 计算通道编码器(GPUComutePassEncoder)



<br>

**copyBufferToBuffer()**：两个缓冲区(Buffer)之间拷贝数据，换句话说就是创建了一个缓冲区副本

```
copyBufferToBuffer(
    source: GPUBuffer,
    sourceOffset: GPUSize64,
    destination: GPUBuffer,
    destinationOffset: GPUSize64,
    size: GPUSize64
): undefined;
```



<br>

**copyBufferToTexture()**：从纹理(Texture)拷贝数据到缓冲区(Buffer)

```
copyBufferToTexture(
    source: GPUImageCopyBuffer,
    destination: GPUImageCopyTexture,
    copySize: GPUExtent3DStrict
): undefined;
```



<br>

**copyTextureToBuffer()**：从缓冲区(Buffer)拷贝数据到纹理(Texture)

```
copyTextureToBuffer(
    source: GPUImageCopyTexture,
    destination: GPUImageCopyBuffer,
    copySize: GPUExtent3DStrict
): undefined;
```



<br>

**copyTextureToTexture()**：两个纹理(Texture)之间拷贝数据，换句话说就是创建了一个纹理副本

```
copyTextureToTexture(
    source: GPUImageCopyTexture,
    destination: GPUImageCopyTexture,
    copySize: GPUExtent3DStrict
): undefined;
```



<br>

**clearBuffer()**：清除缓冲区(Buffer)，这里所谓的 “清除” 实际上是使用 0 填充该命令缓冲区

```
clearBuffer(
    buffer: GPUBuffer,
    offset?: GPUSize64,
    size?: GPUSize64
): undefined;
```

> 本文开头提到了 百度百科中 关于 指令 的解释，其中说到 指令 是由二进制数据(0或1)构成的，那么所谓的 清空 就是指将所有位数上的值都修改成 0。



<br>

**writeTimestamp()**：在当前(之前)所有的命令都被执行完成时，向查询集(GPUQuerySet)中写入一个时间戳

> 注意是 “当前”，并不是 “全部”
>
> 关于 查询集(GPUQuerySet) 我们会在以后具体学习，目前你大概知道它是用于查询命令执行情况的即可。

```
writeTimestamp(
    querySet: GPUQuerySet,
    queryIndex: GPUSize32
): undefined;
```



<br>

**resolveQuerySet()**：将来自 查询集(GPUQuerySet) 的查询结果解析为 缓冲区(GPUBuffer) 的范围。

```
resolveQuerySet(
    querySet: GPUQuerySet,
    firstQuery: GPUSize32,
    queryCount: GPUSize32,
    destination: GPUBuffer,
    destinationOffset: GPUSize64
): undefined;
```



<br>

**finish()**：结束当前命令编码器的录入状态，换句话说就是 告知当前命令编码器 “全部的待编码命令已录入完成”，该方法会返回一个 命令缓冲区(GPUCommandBuffer)。

当调用过 finish() 方法后，当前的 GPUCommandEncoder 实例已完成它的使命，此后该命令编码器就不能再使用了。

```
finish(
    descriptor?: GPUCommandBufferDescriptor
): GPUCommandBuffer;
```



<br>

**特别强调：**

由于 GPUCommandEncoder 同时继承于 GPUCommandsMixin 和 GPUDebugCommandsMixin，所以它除了自身的这些方法外，也拥有继承于 GPUDebugCommandsMixin 的 3 个方法：pushDebugGroup()、popDebugGropu()、insertDebugMarker()



<br>

**简单示例(伪代码)：**

```
const commandEncoder = device.createCommandEncoder({ label: 'myCommandEncoder' })

...
commandEncoder.beginComputePass()
...
commandEncoder.pushDebugGroup('myGroup')
...

const commandBuffer = commandEncoder.finish()
const queue = device.queue.submit([commandBuffer])
```



<br>

**划重点：**

命令编码器(GPUCommandEncoder) 的 .beginRenderPass() 和 .beginComputePass() 会分别返回一个 计算通道编码器(GPURenderPassEncoder)、计算通道编码器(GPUComputePassEncoder)



<br>

关于 WebGPU 中命令相关的编码器，我们先讲到这里，本文到此结束。

接下来将具体学习 计算通道编码器(GPURenderPassEncoder) 和 计算通道编码器(GPUComputePassEncoder)。


