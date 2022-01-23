# 05 WebGPU基础之命令队列(GPUQueue)

**当我们得到一个 显卡设备(GPUDevice) 实例后，可以通过它的属性 `.queue` 来发送一些 GPU 执行命令，而 `.queue` 对应的类型就是 GPUQueue。**



<br>

![GPUQueue](https://puxiao.com/webgpu_tutorial/imgs/gpuqueue.jpg)



<br>

> 特别强调：由于本人也是处于学习 WebGPU 阶段，所以以下内容仅仅为我个人的理解。难免有错误、偏差，还请理解。



<br>

#### 队列(queue)

GPUQueue 中的 Queue 单词本意就是队列，并且它就是 数据结构 中的 队列。

队列(queue) 是一种操作受限制的线性表，其的特点是：先进先出。

队列的 2 个操作行为：

1. 向 队首(前端) 删除并返回一条数据

   > 相当于 JS 中数组的 array.shift() 操作

2. 向 队尾(后端) 添加一条数据

   > 相当于 JS 中数组的 array.push() 操作

<br>

所谓 `队列的特点：先进先出`，就是 **GPUQueue 作为命令队列，会严格按照命令的添加顺序逐条执行**。



<br>

> 与 队列(queue) 对应的是另外一种数据结构：栈(stack)，也被称为 堆栈。
>
> 堆栈的特点是：后进先出
>
> 堆栈的 2 个操作行为：
>
> 1. 向 队尾 添加一条数据，相当于 JS 中数组的 array.push() 操作
> 2. 向 队尾 删除并返回一条数据，相当于 JS 中数组的 array.pop() 操作
>
> 在上一篇文章讲解 GPUDevice 时，简单提到了它有 2 个方法：`pushErrorScope()`、`popErrorScope()`，这两个方法操作的就是 堆栈。



<br>

当我们明白 GPUQueue 的作用和含义后，那么接下来就希望知道 GPUQueue 究竟可以发送哪些类型的执行命令，也就是 GPUQueue 这个类提供哪些方法。



<br>

#### GPUQueue的属性和方法

GPUQueue 没有任何属性。

GPUQueue 本身不可以通过 new 关键词来实例化。

GPUQueue 只可以通过 GPUDevice 实例的 .queue 属性来获取。

> 这句话暗含的意思是：一个 GPUDevice 实例只对应一个 GPUQueue 实例，同理，一个 GPUQueue 实例 也只能对应一个 GPUDevice 实例。
>
> GPUDevice 实例 和 GPUQueue 实例 是一种互相绑定的关系。



<br>

如果我们简单得把 GPUAdapter 看作是 浏览器与 GPUDevice 的沟通桥梁，那么 GPUQueue 就是 GPUDevice 与 GPUBuffer 的沟通桥梁。

关于 GPUBuffer(显存缓冲区) 我们会在下一篇文章中讲解。



<br>

> 绝大多数人都会将 GPUBuffer 翻译为 “缓冲区”，并不会像我这样翻译成 “显存缓冲区”。我将 GPUBuffer 翻译成 显存缓冲区，多少是有一些歧义的，但是我认为这样翻译会很容易让人理解。

<br>

> 补充一点计算机术语：内存与显存
>
> 内存(memory)：暂时存放 CPU 中的运算数据
>
> 显存：显卡的内存，暂时存放 GPU 中的运算数据
>
> 而 Buffer 这个单词直译就是 “缓冲区”，可以简单将 GPUBuffer 理解为：“临时存放那些即将要存入显存(还未真正存入)中的运算数据”。



<br>

> 特别强调：在解释内存和显存时，使用的词语是 “运算数据”，不是 “运算结果”。
>
> 也就是说 **GPUBuffer 对应的数据可能是供 GPU 运算的一些数据，不是 GPU 运算结果**。
>
> 
>
> 至此，对 GPUBuffer 应该有了一个简单的概念，那么我们可以回到本文关于 GPUQueue 的学习中。



<br>

下面开始讲解 GPUQueue 的各个方法。

> 这些方法、参数的讲解，仅代表我个人目前的理解。



<br>

**writeBuffer()：将提供的数据写入到 GPUBuffer 中**

下面是 `@webgpu/types` 中关于 writeBuffer() 方法的类型定义：

```
writeBuffer(
    buffer: GPUBuffer,
    bufferOffset: GPUSize64,
    data:
      | BufferSource
      | SharedArrayBuffer,
    dataOffset?: GPUSize64,
    size?: GPUSize64
): undefined;
```

| 参数         | 类型                              | 是否可选 | 含义                                                         |
| ------------ | --------------------------------- | -------- | ------------------------------------------------------------ |
| buffer       | GPUBuffer                         | 必填     | 要被写入数据的 GPUBuffer 实例                                |
| bufferOffset | GPUSize64                         | 必填     | 写入位置的偏移量，以字节为单位                               |
| data         | BufferSource \| SharedArrayBuffer | 必填     | 要写入的数据                                                 |
| dataOffset   | GPUSize64                         | 可选     | 要写入数据的开始偏移量<br />不传该参数则默认为 data 的起始位置 |
| size         | GPUSize64                         | 可选     | 要写入数据的长度<br />不传该参数则默认为 data 的结束位置     |

> 对于第 4 个参数 dataOffset 的补充：
>
> 1. 若第 3 个参数 data 类型为 TypedArray，则 dataOffset 的值为数组元素索引值
> 2. 否则 dataOffset 的值为以字节为单位的偏移量，同时该值必须是 4 的倍数。
>
> 同理，第 5 个参数 size 对应的是 数组的长度 或 字节的长度。

<br>

> 请注意 TypedArray 并不是 WebGPU 中的概念，而是 JS 中的一种类型化数组的概念，具体可查阅：
>
> https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/TypedArray

> 类型化数组 在 TypeScript 种通常被称为 元组。



<br>

**writeTexture()：将提供的数据写入到 GPUTexture 中**

> 由于我们还没学习过 GPUTexture，这里你只需要知道它是指 贴图纹理 即可。

关于 writeTexture() 的 TS 类型定义：

```
writeTexture(
    destination: GPUImageCopyTexture,
    data:
      | BufferSource
      | SharedArrayBuffer,
    dataLayout: GPUImageDataLayout,
    size: GPUExtent3DStrict
): undefined;
```

| 参数        | 类型                              | 是否可选 | 含义                                        |
| ----------- | --------------------------------- | -------- | ------------------------------------------- |
| destination | GPUImageCopyTexture               | 必填     | 要写入的纹理资源和原点                      |
| data        | BufferSource \| SharedArrayBuffer | 必填     | 要写入到 destination 中的纹理数据           |
| dataLayout  | GPUImageDataLayout                | 必填     | data 中的内容布局                           |
| size        | GPUExtent3DStrict                 | 必填     | 从 data 写入到 destination 中的内容范围长度 |



<br>

**copyExternalImageToTexture()：将图像或画布内容复制到目标纹理中**

关于 copyExternalImageToTexture() 的 TS 类型定义：

```
copyExternalImageToTexture(
    source: GPUImageCopyExternalImage,
    destination: GPUImageCopyTextureTagged,
    copySize: GPUExtent3DStrict
): undefined;
```

| 参数        | 类型                      | 是否可选 | 含义                                              |
| ----------- | ------------------------- | -------- | ------------------------------------------------- |
| source      | GPUImageCopyExternalImage | 必填     | 要复制到“目标(destination)”的源图像和起始点       |
| destination | GPUImageCopyTextureTagged | 必填     | 要写入的纹理资源和原点，以及编码元数据            |
| copySize    | GPUExtent3DStrict         | 必填     | 要从“源(source)”写入“目标(destination)”的内容范围 |

> 上面第 1 个参数 source 中的起始点(源点)是指要复制的起始点位置，你可以把它简化想象成是一个二维坐标，即(x,y)。
>
> 上面第 3 个参数 copySize 中的内容范围是指复制的宽和高，对应的是 copySize.width，copySize.height。
>
> 请注意 source.z 和 copySize.depthOrArrayLayers 也是有值的，具体这 2 个值的作用以后再讲。



<br>

**submit()：提交在此队列上的命令缓存**

简单直白点，就是 “提交命令”，提交的内容是 GPUCommandBuffer(命令缓存)。

关于 submit() 的 TS 类型定义：

```
submit(
    commandBuffers: Iterable<GPUCommandBuffer>
): undefined;
```

| 参数           | 类型                         | 是否可选 | 含义                 |
| -------------- | ---------------------------- | -------- | -------------------- |
| commandBuffers | `Iterable<GPUCommandBuffer>` | 必填     | 需要被提交的命令缓存 |

> `Iterable<GPUCommandBuffer>` 中的 "Iterable" 是指可以通过 for ... of 循环迭代访问的 数组或集合，例如 Array、Map、Set 等。



> GPUBuffer 中的 Buffer 会被翻译为 “缓冲区”，但是 GPUCommandBuffer 中的 Buffer 我更倾向于翻译成 “缓存”。



<br>

**onSubmittedWorkDone()：当此队列已完成全部的提交后触发 Promise 回执**

```
onSubmittedWorkDone(): Promise<undefined>;
```



<br>

**小总结：**

GPUQueue 的方法可以大致划分成 3 类：

1. 自身就可以提交的一些命令：writeBuffer()、writeTexture()、copyExternalImageToTexture()
2. 提交由 GPUCommandBuffer 定义好的其他命令：submit()
3. 监控所有命令提交完成：onSubmittedWorkDone()



<br>

> 在 https://www.w3.org/TR/webgpu/ 中是在第 18 章才开始讲解 GPUQueue 的，但是我认为如果不提前简单学习一下 GPUQueue，很难去学习 GPUBuffer，所以提前介绍了 GPUQueue。

> 与 GPUQueue 相关的是 GPUQueueSet(队列查询)，以后用到了再详细讲它。



<br>

由于 GPUQueue 每个方法中涉及到了很多我们还未学习过的类型，所以对于本文提到的 GPUQueue 的这些方法你只需有个大致印象即可。

有了这个初步印象后便于我们接下来要学习的 GPUBuffer。

本文到此结束。