# 24 WebGPU示例之：颜色插值

**所谓颜色插值就是指着色器根据某种插值算法将两个像素之间进行临近颜色计算，得到一个处于它们过渡状态的颜色。**



<br>

但是由于颜色插值实际上是由 WebGPU 自动完成的，所以对于我们而言，**颜色插值的重点在于：我们如何将不同的顶点设置成不同的颜色。**

> 这样顶点构成的三角形的中间区域颜色，则由 WebGPU 通过插值计算自动得出。



<br>

先看一下今天示例的最终结果：

![color_interpolation](https://raw.githubusercontent.com/puxiao/webgpu-tutorial/main/imgs/color_interpolation.jpg)

我们将三角形的三个顶点分别设置成 红、绿、蓝 3 个颜色，然后 WebGPU 就会自动进行颜色插值计算，将三角形内容区域填充过渡颜色。

> 在之前的示例中我们始终绘制的都是一个单色的三角形，而今天，我们终于可以绘制一个五彩斑斓的三角形了。



<br>

先说一下本示例所涉及的 3 个知识点：

1. 颜色值转换：我们需要在 JS 中将十六进制的颜色值，例如红色 "#ff0000" 转换成 WGSL 支持的归一化后的 RGB 颜色值(取值范围 0 - 1)

2. 顶点缓冲区：将每个顶点的位置、颜色 传递到顶点着色器中

3. 自定义结构体：顶点着色器不再是仅仅把一个坐标信息对外返回，而是返回一个包含位置和颜色的自定义结构体供渲染管线的下一个阶段使用。

   > 在 WGSL 中通过 `struct` 关键字可以定义结构体



<br>

接下来我们将通过代码片段，分别展示一下上面提到的 3 个知识点。



<br>

**知识点 1：十六进制颜色值转换为RGB：**

这部分属于 JS 的一个知识点，和 WebGPU 关联不大，所以我们直接看代码。

> 我使用 TypeScript 来编写代码

> src/utils/colorToRGB.ts

```
interface RGB {
    r: number
    g: number
    b: number
}

export type colorStr = `#${string}`

export const colorToRGB = (color: colorStr): RGB => {
    const r: number = parseInt(color.substring(1, 3), 16)
    const g: number = parseInt(color.substring(3, 5), 16)
    const b: number = parseInt(color.substring(5, 7), 16)

    return { r, g, b }
}

export const colorToNormalizeRGB = (color: colorStr): RGB => {
    const res: RGB = colorToRGB(color)
    res.r = res.r / 255
    res.g = res.g / 255
    res.b = res.b / 255
    return res
}
```



<br>

我们将颜色转换这部分独立出来，colorToRGB.ts 对外导出 2 个函数：

1. colorToRGB：将类似 "#ff0000" 的颜色值转换为 RGB 颜色值

   > 此时 RGB 每个分量的取值范围为 0 - 255

2. colorToNormalizeRGB：将类似 "#ff0000" 的颜色值转换为归一化后的 RGB 颜色值

   > 此时 RGB 每个分量的取值范围为 0 - 1



<br>

**知识点2：将顶点位置、颜色通过顶点缓冲区传递给顶点着色器**

关于顶点缓冲区如何传递多个参数，我们之前已经学习过了。

简单复习一下：

1. 第 1 种方式：将顶点的位置和颜色分别存放到同一个 缓冲区中，然后再通过设置偏移量来读取(区分)位置和颜色。
2. 第 2 种方式：将顶点的位置和颜色分别存放到不同的 2 个缓冲区中
3. 无论采用以上哪种方式，对于顶点着色器 vert.wgsl 而言，接收这 2 个参数的代码是完全相同的



<br>

在本示例中，我们希望位置和颜色相互独立，所以我们采用第 2 种方式。



<br>

在实际示例中，还通过 JS 可以动态修改三个顶点的颜色值，但此刻为了讲解方便，我们假定颜色颜色值是固定的，这样便于代码演示。

> 接下来，就当是对 顶点缓冲区的一个复习。



<br>

创建顶点坐标缓冲区：

```
const vertexArray = new Float32Array([
    0.0, 0.5, 0.0,
    -0.5, -0.5, 0.0,
    0.5, -0.5, 0.0,
])

const vertexBuffer = device.createBuffer({
    size: vertexArray.byteLength,
    usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST
})

device.queue.writeBuffer(vertexBuffer, 0, vertexArray)
```



<br>

创建顶点颜色缓冲区：

> 请注意这里使用的 rgb  颜色值是归一化后的，取值范围为 0 -1 
>
> 例如此时 r 值为 1.0 相当于日常颜色值中 r = 255

```
const colorArr = new Float32Array([
    1.0, 0.0, 0.0, //红色
    0.0, 1.0, 0.0, //绿色
    0.0, 0.0, 1.0 //蓝色
])

const colorBuffer = device.createBuffer({
    size: colorArr.byteLength,
    usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST
})

device.queue.writeBuffer(colorBuffer, 0, colorArr)
```



<br>

> 不要嫌我啰嗦：无论上面的坐标缓冲区，还是颜色缓冲区，他们的使用标识都有 GPUBufferUsage.VERTEX，这有添加了这个标识才表明该缓冲区是要应用到顶点缓冲区中的。



<br>

在管线中配置上述 坐标缓冲区、顶点缓冲区

```
const pipeline = device.createRenderPipeline({
    layout: 'auto',
    vertex: {
        ...
        buffers: [
            {
                arrayStride: 3 * 4,
                attributes: [
                    {
                        shaderLocation: 0,
                        offset: 0,
                        format: 'float32x3'
                    }
                ]
            },
            {
                arrayStride: 3 * 4,
                attributes: [
                    {
                        shaderLocation: 1,
                        offset: 0,
                        format: 'float32x3'
                    }
                ]
            }
        ]
    },
    ...
})
```

> 我们采用给 buffers 添加 2 个元素的形式来配置 2 个缓冲区(参数)
>
> 另外一种设置顶点缓冲的方式是 给 buffers 只添加 1 个元素，但是在这个元素的 attributes 中添加 2 个配置项，通过设置第 2 个配置项的 offset 来配置读取偏移。
>
> 注意：无论采用那种方式，都记得要将他们的 shaderLocation 设置成不同的值，该值决定他们在顶点着色器入口函数中是第几个参数。



<br>

最后将 坐标缓冲区、颜色缓冲区分别提交到 GPU 中

```
passEncoder.setVertexBuffer(0, vertexBuffer)
passEncoder.setVertexBuffer(1, colorBuffer)
```



<br>

好了，接下来终于到本文的重点了。



<br>

**结构体(structure)：**

在 WGSL 中内置了一些数据类型，例如：

1. 标量：例如 bool、f32、i32、u32、f16

2. 向量：例如 vec3、vec4

3. 矩阵：例如 mat3x3、mat4x4

4. 原子类型：atomic

   > 目前原子类型我们还没有使用过，简单来说 原子类型是 GPU 多线程中最小不可分割的单位

在某些情况下，单独某一个以上的类型无法满足我们的数据类型时，那么就需要自定义类型。

在 WGSL 中将这种自定义数据结构类型的对象称呼为：结构体

在 WGSL 中是通过 `struct` 这个关键词来定义 结构体的。

> 说直白点，struct 就相当于 TypeScript 中的 interface 这个关键词。



<br>

**顶点着色器 vert.wgsl 的代码：**

> 直接看代码，就知道 结构体 是怎么一回事了。

```
struct VertexOutput {
    @builtin(position) position : vec4<f32>,
    @location(0) color: vec4<f32>
}

@group(0) @binding(0) var<uniform> mvp:mat4x4<f32>;

@vertex
fn main(@location(0) xyz: vec3<f32>, @location(1) color: vec3<f32>) -> VertexOutput {
    var output: VertexOutput;
    output.position = mvp * vec4<f32>(xyz, 1.0);
    output.color = vec4<f32>(color, 1.0);
    return output;
}
```



<br>

我们通过 `struct VertexOutput { ... }` 定义了一个名为 VertexOutput 的结构体。

这个结构体包含 2 个元素：坐标 和 颜色



<br>

**第1个元素：坐标**

我们通过 `@builtin(position)` 定义了一个 内置的 顶点坐标变量 position，该变量 position 可供下一个管线节点使用。

> 请注意，你只能将其命名为 position 后，在下一个管线阶段才可以正确读取。

问：下一个管线节点是什么？



<br>

> 复习一下渲染管线的几个节点：
>
> 数据输入 > 顶点处理 > 网格三角形处理 > 光栅化 > 片元处理 > 帧缓冲区操作 > 显示结果(输出)



<br>

答：网格三角形处理，当然也可以称呼其为 “图元装配”。



<br>

**第2个元素：颜色**

我们通过 `@location(0) color ...` 定义了一个名为 color 的变量。



<br>

**实例化结构体：**

```
var output: VertexOutput;
output.position = mvp * vec4<f32>(xyz, 1.0);
output.color = vec4<f32>(color, 1.0);
```

在上述代码中，我们通过 var 实例化了结构体 VertexOutput，并且将顶点缓冲区中得到的数据赋值给它的 2 个属性中。



<br>

我们再看一下 片元着色器的代码。

**片元着色器的相应代码：**

```
struct VertexOutput {
    @builtin(position) position : vec4<f32>,
    @location(0) color: vec4<f32>
}

@fragment
fn main(output: VertexOutput) -> @location(0) vec4<f32> {
    return output.color;
}
```

> 请留意 main 中参数 output 前面是没有加任何修饰符的。



<br>

<br>

在片元着色器中的入口函数中，它接收到参数 output，同样它的类型也是 VertexOutput。

片元着色器需要做的事情就是将 output.color 的值返回出去，供下一个管线节点使用。



<br>

**特别强调：**

没错 ，再次说一遍。

在顶点着色器中，结构体 VertexOutput 实例 output 包含了 2 个属性：position、color，其中 output.position 是供下一步 网络三角形处理(图元装配) 使用的，而片元阶段只关心颜色，不关心位置(因为此时三角形已经生成过了)，所以在 片元着色器中并未出现 output.position，只用到了 output.color 。



<br>

至此，关于颜色插值示例的核心代码已经讲解完毕。

想查看实际完整的代码，请访问：

https://github.com/puxiao/react-webgpu-samples/blob/main/src/components/color-interpolation/index.tsx



<br>

截至目前，以绘制一个三角形为例，我们已经完成了以下几个示例：

1. 顶点缓冲区
2. 资源绑定(缓冲区)
3. MVP 变换矩阵
4. 颜色插值



<br>

本文到此结束，接下来，我们将实现 立体多面 的 三角体 或 立方体。

