# 20 WebGPU示例之资源绑定

**本文将通过一个简单示例来学习和使用资源绑定。**



<br>

**何为 “资源(resource)” ？**

1、采样器(GPUSampler)

2、纹理视图(GPUTextureView)

3、缓冲区(GPUBufferBinding)

4、外部纹理资源(GPUExternalTexture)



<br>

> 你会发现上面提到的几类资源实际上都是跟 片元着色器 关系紧密，和 顶点着色器 关联不大。
>
> 尽管资源绑定中包含缓冲区，但一般情况下资源绑定中的 缓冲区 不会是 顶点缓冲区。



<br>

**何为“绑定(bind)” ？**

就是将 资源 有序地进行 排列组织在一起，并编上序号。



<br>

**本文学习目标：**

本文我们只学习 资源绑定 4 种资源中的 数据缓冲区，至于 采样器、纹理视图、外部纹理资源 等以后再学习。



<br>

**本文示例：**

之前我们通过创建 顶点缓冲区 将顶点坐标数据传递到 顶点着色器中。

本文，我们将通过 资源绑定 来将一个颜色值传递到 片元着色器中。



<br>

> 简单来说就是 之前我们是将一个三角形 3 个顶点坐标信息传递到 WGSL 着色器中，而今天我们将三角形的颜色值传递到 WGSL 着色器中。



<br>

**先创建一个颜色缓冲区**

按照创建 顶点缓冲区 的套路，先来创建一个颜色缓冲区。

```
const colorArr = new Float32Array([
    1.0, 0.0, 0.0, 1.0
])

const colorBuffer = device.createBuffer({
    size: colorArr.byteLength,
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST
})

device.queue.writeBuffer(colorBuffer, 0, colorArr)
```



<br>

**GPUBufferUsage.UNIFORM**

请注意，之前创建顶点缓冲区时，缓冲区的使用标记是：

```
usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST
```

而刚才创建颜色缓冲区时，缓冲区的使用标记是：

```
usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST
```



<br>

> 你可能会以为既然有 GPUBufferUsage.VERTEX，那是不是也会有 GPUBufferUsage.FRAG 或 GPUBufferUsage.FRAGMENT 呢？ 答案是：没有



<br>

**缓冲区名称：**

1. 使用 GPUBufferUsage.VERTEX 标记的缓冲区，我们称之为 顶点缓冲区

2. 使用 GPUBufferUsage.UNIFORM 标记的缓冲区，我们称之为 统一缓冲区

   > uniform 这个单词的本意为 “统一的、一致的”
   >
   > 在很多 OpenGL 或 Vulkan 文章中都是直接使用 uniform 这个单词，没有对其进行中文翻译，我个人不习惯中文教程中出现大量英文单词。



<br>

**可用范围：**

1. 顶点缓冲区 顾名思义，其缓冲区数据只应作用于 顶点着色器中
2. 而 统一缓冲区 中的 “统一” 你可以理解为 “各个阶段的着色器统统都可以访问” ，也就是说 统一缓冲区中的数据在 顶点着色阶段 和 片元着色阶段都可以访问。
3. 但绝大多数时候，统一缓冲区 都是应用在 片元着色器中。



**与 UNIFORM 相近的是 STORAGE：**

1. 统一缓冲区(uniform buffer) 最大可用上限为 64K
2. 存储缓冲区(storage buffer) 最大可用上线为 2G



<br>

> 我们今天只学习 统一缓冲区，GPUBufferUsage.STORAGE 先不做讲解。



<br>

三角形的颜色缓冲区刚才已经创建好了，那究竟该怎么传递到 WGSL 呢？

绑定组(GPUBindGroup) 要登场了。



<br>

**资源绑定组(GPUBindGroup)：**

资源绑定组是通过 GPUDevice 的 .createBindGroup() 来创建的。

它的作用就是将一系列资源 组合、排序、编号。



<br>

> 本文后面，我们将 资源绑定组 简化称呼为 绑定组。



<br>

**绑定组与渲染管线(GPURenderPipeline)关联的时机：**

1. 顶点缓冲区是在创建渲染管线时，通过配置管线描述的 vertex: { buffers : [ ... ]} 来进行关联的。
2. 绑定组则是在 渲染管线创建完成后，通过 device.createBindGroup( { layout: ... }) 中的描述配置与渲染管线进行关联的。



<br>

**创建资源绑定组的完整代码：**

> 假设 pipeline 为我们已经创建好的渲染管线

```
const bindGroup = device.createBindGroup({
    layout: pipeline.getBindGroupLayout(0),
    entries: [
        {
            binding: 0,
            resource: { buffer: colorBuffer}
        }
    ]
})
```

1. `device.createBindGroup({ ... })`：通过调用 显卡设备实例的 createBindGroup() 来创建一个绑定组

2. `layout`：布局，这里是指当前绑定组对应哪个 绑定组布局(GPUBindGroupLayout)

   > GPU 缓冲区中的内存是连续的，所谓连续可以简单理解为 “不同数据(字节)之间是连续紧挨着的”。为了在显存中快速定位到某一段数据，那么就需要对每一段数据进行标记/分割，这就是所谓 “布局” 。
   >
   > 上面代码中的 绑定组布局索引值 0 可以简单理解为 “数据的分布标记索引值”。
   
3. `pipeline.getBindGroupLayout(0)`：渲染管线通过调用它的 getBindGroupLayout() 函数用来获取一个 绑定组布局 实例，而参数 0 表示索引为 0 的绑定组布局。

4. `entries`：用于设定该绑定组的一些配置，它的值是一个数组，也就意味着可以传递多个 资源 到同一个 绑定组中。

5. `binding: 0`：绑定组中对应的 资源索引值

6. `resource`：绑定组中的资源，对于本文示例而言，我们绑定的资源是 一个 包含颜色值信息的缓冲区

   



<br>

当我们得到了一个 绑定组，接下来就剩最后一步：将这个绑定组 添加到 通道编码器中。

```diff
passEncoder.setPipeline(pipeline)
passEncoder.setVertexBuffer(0, vertexBuffer)
+ passEncoder.setBindGroup( 0, bindGroup )
```



<br>

此刻，JS 中的工作全部完成，那么接下来看一下在 片元着色器 中如何访问 统一缓冲区 中的数据。



<br>

**片元着色器中获取绑定资源的代码：**

```
@group(0) @binding(0) var<uniform> color:vec4<f32>;
@fragment
fn main() -> @location(0) vec4<f32> {
    return color;
}
```

这里面起关键作用的就是第 1 行代码。

1. `@group(0)`：获取绑定组布局索引值为 0 的绑定组
2. `@binding(0)`：获取这个绑定组中 资源索引为 0 的资源，并且表明这个资源将会和后面即将声明的变量进行关联
3. `var<uniform>`：声明一个 uniform 性质的变量
4. `color:vec4<f32>`：这个变量名为 color，变量值类型为 `vec4<f32>`



<br>

**小总结：**

如果不去深究资源绑定这个比较大的概念，我们只是去实现 “如何传递一个颜色值到顶点着色器”。

那么上面的一系列操作总结下来也就以下几个步骤：

1. 创建一个包含颜色值的缓冲区
2. 当渲染管线创建好后，通过它的 .getBindGroupLayout(0) 函数来得到它的一个绑定组布局
3. 将 绑定组布局、绑定索引、资源内容(颜色缓冲区) 作为创建一个绑定组的配置项
4. 通过 device.createBindGroup({ ... }) 方法来创建一个绑定组
5. 通过 通道打包器的 .setBindGroup() 方法对这个绑定组进行打包
6. 最终片元着色器中通过 `@group(0)`、`@binding(0)`、`var<uniform>` 得到这个颜色缓冲区



<br>

至此，我们已经实现了：

1. JS 传递三角形顶点坐标数据到 顶点着色器 中
2. JS 传递三角形填充的颜色值到 片元着色器 中

掌握了这些数据操作，我们已经可以写出一些有交互的 WebGPU 示例了。



<br>

本文到此结束，接下来我们将多写几个示例。