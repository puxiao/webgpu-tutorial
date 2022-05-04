# 17 WebGPU示例之绘制一个三角形

**我们通过绘制一个简单的三角形来了解 WebGPU 的渲染流程。**



<br>

与本系列教程对应的示例代码仓库：https://github.com/puxiao/react-webgpu-samples



<br>

**说在前面的话：**

首先关于绘制三角形，网上已经有一些相关教程(当然这些教程中的代码大同小异)。

* 半官方性质的示例：https://austin-eng.com/webgpu-samples/samples/helloTriangle

  > 该示例作者是 WebGPU 核心开发者之一，所以这里称呼这个示例为 “半官方性质”，我相信网上其他 WebGPU 示例代码都是参考了该教程中的示例代码。

* 玄玄子 “Hello，WebGPU —— 绘制第一个三角形”：https://juejin.cn/post/7055155834589806600

* Orillusion 社区课程 WebGPU小白入门 认识渲染管线 和 绘制简单三角形：https://mp.weixin.qq.com/s/JZiwahBvv3PqokRNGaNCNw

  > 这里我强烈推荐 Orillusion 这套正在更新中的 WebGPU 入门视频课程



<br>

那既然都已经有人写过类似的、入门的 绘制三角形 课程了，那本文还有必要再写吗？

答：有必要



<br>

**第1**：别人会不代表你自己会。

> 我会继续 “通过写文章教会别人的方式来进行自己的学习提高”，继续自己的 WebGPU 的学习之路。

**第2**：我会补充别人教程中没有提及的知识点

> 看某一个教程一定需要经历的几个过程：总结、怀疑、验证、补充

**第3**：我的示例代码是基于 React + TypeScript

> 我会尝试用自定义 hooks 封装一些 WebGPU 的异步操作，当然这种究竟对不对，有多少意义暂且不去追究



<br>

**特别说明：**

上述那些别人写的 WebGPU 教程，假设的前提是你从未接触过 WebGPU，而本系列教程前面已经花费了相当多的时间用来解读 WebGPU 中绝大多数相关模块、类 的用法，所以在本文以及后续示例中，在涉及某些类的属性和方法时，只是会简单回顾一下，但不会做过多讲述，若想了解相关类的构造参数、属性和方法，可以翻阅以往文章。

<br>

在这个过程中，可能会纠正之前对于某些类的属性和方法的错误理解之处。

请不要完全相信本系列教程，因为我也是在摸索中。



<br>

> 我特别建议你先去看 Orillusion 出的那套 WebGPU 入门课程，之后再来阅读本文。
>
> 尤其是你使用 Vue 不使用 React 的，本文可能不太适合你。



<br>

好了，开始吧。



<br>

#### 创建基础项目

首先需要按照上一篇文章 “WebGPU示例之基础框架搭建(React18+Craco+TypeScript)” ，搭建一个基础项目。

> 相对于第一次发布，我更新了 2 处地方：
>
> 1、目前最新版 create-react-app 5.0.1 已默认适配了 react 18，所以无需我们再手工修改 index.tsx
>
> 2、在配置 alias 时，将存放 .wgsl 文件目录由 /wgsl 改为 shaders/，同时删除关于 @/src 的这条配置项

> 本系列教程地址：https://github.com/puxiao/webgpu-tutorial
>
> 本系列教程示例地址：https://github.com/puxiao/react-webgpu-samples



<br>

#### 自定义钩子函数：useDevice()

考虑到每一个 WebGPU 项目最初都需要获取 显卡设备(GPUDevice)，所以我们先自定义一个 钩子函数(hooks)，用于获取当前浏览器程序中的 GPUDevice 实例。

> 关于 显卡适配器(GPUAdapter) 与 显卡设备(GPUDevice)，可回顾 “[WebGPU基础之显卡适配器(GPUAdapter)与显卡设备(GPUDevice)](https://github.com/puxiao/webgpu-tutorial/blob/main/04%20WebGPU%E5%9F%BA%E7%A1%80%E4%B9%8B%E6%98%BE%E5%8D%A1%E9%80%82%E9%85%8D%E5%99%A8(GPUAdapter)%E4%B8%8E%E6%98%BE%E5%8D%A1%E8%AE%BE%E5%A4%87(GPUDevice).md)” 这一篇文章。



**useDevice：**

> src/hooks/useDevice.ts
>
> 在线地址：https://github.com/puxiao/react-webgpu-samples/blob/main/src/hooks/useDevice.ts

```
import { useEffect, useState } from "react"

const useDevice = () => {
    const [adapter, setAdapter] = useState<GPUAdapter>()
    const [device, setDevice] = useState<GPUDevice>()

    useEffect(() => {
        if (navigator.gpu === undefined) return
        const initWebGPU = async () => {
            const adapter = await navigator.gpu.requestAdapter()
            if (adapter === null) return
            const device = await adapter.requestDevice()
            setAdapter(adapter)
            setDevice(device)

            device.lost.then(() => {
                setDevice(undefined)
            })
        }
        initWebGPU()
    }, [])

    return {
        adapter,
        device
    }

}

export default useDevice
```

useDevice 内部代码讲解：

1. 首先通过 `navigator.gpu` 是否有值，来判断当前浏览器是否支持 WebGPU
2. 通过 `navigator.gpu.requestAdapter()` 来异步获取 显卡适配器(GPUAdapter) 实例
3. 通过 `adapter.requestDevice()` 来异步获取 显卡设备(GPUDevice) 实例
4. 通过 `device.lost.then(()=>{ ... })` 来添加显卡设备丢失时的回调函数
5. 该钩子函数对外返回 显卡适配器(adapter) 和 显卡设备实例(device)



<br>

虽然 useDevice 内部并不复杂，但是它可以帮我们简化一些异步逻辑判断。

当我们想获得 显卡适配器 和 显卡设备 时，只需下面这样的代码：

```
const { adapter, device } = useDevice()
```



<br>

#### 自定义钩子函数：useWebGPU

我们需要把 3D 渲染内容输出显示到网页中，就需要用到 画布 `<canvas />`，以及画布的 GPU 上下文(GPUCanvasContext)，所以我们在 useDevice 的基础上，再额外定义一个名为 useWebGPU 的钩子函数，进一步封装。

> 关于 GPUCanvasContext 可回顾 “[WebGPU基础之画布上下文(GPUCanvasContext)和常见错误](https://github.com/puxiao/webgpu-tutorial/blob/main/15%20WebGPU%E5%9F%BA%E7%A1%80%E4%B9%8B%E7%94%BB%E5%B8%83%E4%B8%8A%E4%B8%8B%E6%96%87(GPUCanvasContext)%E5%92%8C%E5%B8%B8%E8%A7%81%E9%94%99%E8%AF%AF.md)” 这一篇文章



<br>

**useWebGPU：**

> src/hooks/useWebGPU.ts
>
> 在线地址：https://github.com/puxiao/react-webgpu-samples/blob/main/src/hooks/useWebGPU.ts

```
import { useEffect, useState } from "react"
import useDevice from "./useDevice"

const useWebGPU = (canvas: HTMLCanvasElement | null | undefined) => {

    const [context, setContext] = useState<GPUCanvasContext>()
    const [format, setFormat] = useState<GPUTextureFormat>('bgra8unorm')
    const { adapter, device } = useDevice()

    useEffect(() => {

        if (!canvas || !adapter) return

        const context = canvas.getContext('webgpu')
        if (context === null) return
        setContext(context)

        const preferredFormat = context.getPreferredFormat(adapter) // RGBA8Unorm
        setFormat(preferredFormat)

    }, [canvas, adapter])

    return { canvas, context, format, adapter, device }
}

export default useWebGPU
```

useWebGPU  内部代码讲解：

1. 该函数接收一个 canvas 的参数，这个 canvas 就是 DOM 中的 画布元素

2. 通过 useDevice 尝试获取 显卡适配器(GPUAdapter)、显卡设备(GPUDevice)

3. 当画布元素(canvas) 和 显卡适配器实例(adapater) 都不为 null 或 undefined 时，执行后续代码

4. 通过 `canvas.getContext('webgpu')` 来获取 GPU 画布上下文

5. 通过 GPU 画布上下文的 .getPreferredFormat() 获取到当前浏览器最适合的颜色格式

   > 实际上绝大多数设备的最佳颜色格式都是 bgra8unorm
   >
   > 平时我们说颜色习惯顺序是 rgb，此处顺序颠倒为 bgr，这个跟颜色值作为字节码存储的顺序有关，bgr 这个顺序更利于读取。网上有另外一种说法：OpenCV 最早为了兼容老的设备采用了 bgr 这个顺序，导致后面很多框架或库也都跟着使用了这个顺序。

6. 该钩子函数对外返回 { canvas, context, format, adapter, device }



<br>

此时此刻，我们已经可以通过 useWebGPU() 获取 几个关键对象了，那么接下来看一下对应页面的代码。



<br>

**主页面代码：SimpleTrangle.tsx**

> src/components/simple-trangle/index.tsx
>
> 在线地址：https://github.com/puxiao/react-webgpu-samples/blob/main/src/components/simple-triangle/index.tsx

```
import { useEffect, useRef } from "react"
import useWebGPU from "@/hooks/useWebGPU"

const SimpleTriangle = () => {

    const canvasRef = useRef<HTMLCanvasElement | null>(null)
    const { adapter, device, canvas, context, format } = useWebGPU(canvasRef.current)

    useEffect(() => {
        if (!canvas || !context || !adapter || !device) return

        //...

    }, [canvas, context, format, adapter, device])

    return (
        <canvas ref={canvasRef} width={document.body.clientWidth} height={document.body.clientHeight} tabIndex={0} />
    )
}

export default SimpleTrangle
```

SimpleTrangle.tsx 内部代码讲解：

1. 定义 canvasRef 用于 勾住 画布元素

2. 通过 useWebGPU() 获取开发 WebGPU 初始的几个关键对象(实例)

3. 设置画布元素的 宽度和高度，特别注意对于 画布(canvas) 而言不要试图通过 CSS 样式来修改其尺寸

4. 由于画布元素默认是不接收键盘事件的，因此我们额外添加了 `tabIndex={0}`，这样画布就可以接收键盘事件了

   > 尽管在本示例中我们并没有使用键盘事件，但是出于习惯，还是加上吧

5. 在 useEffect() 中，先检查确保 几个关键对象 都不为 null 或 undefined，接下来就可以开始编写绘制三角形的具体代码了



<br>

在开始编写具体代码前，我们先简单归纳一下完成一次 WebGPU 渲染所需要大大致流程。



<br>

#### WebGPU代码流程

结合本示例目标 “绘制一个三角形”，我们先归纳一下大体代码的流程：

1. 获取 显卡适配器、显卡设备、画布上下文、最佳颜色格式

   > 这一步上面我们已经完成了

2. 配置 画布上下文

   > 我们需要调用 GPUCanvasContext 的 .configure() 方法，去配置具体的渲染配置项节

3. 创建 一个渲染管线(GPURenderPipeline)，在渲染管线中，我们需要对 顶点(vertex)、片元(fragment)、原始组装(primitive) 这 3 个阶段进行相关配置

   > 渲染管线 8 个细分阶段：获取顶点、顶点着色、原始组装、光栅化、片元着色、深度模板附件测试与操作、深度测试与写入、输出合并

   > 关于管线，请回顾 “[WebGPU基础之管线(GPUPipelineBase、GPUComputePipeline、GPURenderPipeline)](https://github.com/puxiao/webgpu-tutorial/blob/main/11%20WebGPU%E5%9F%BA%E7%A1%80%E4%B9%8B%E7%AE%A1%E7%BA%BF(GPUPipelineBase%E3%80%81GPUComputePipeline%E3%80%81GPURenderPipeline).md)” 一文。

   > 有些文章会把 “原始组装” 称呼为 “图元拼装”

4. 创建命令编码器，再由命令编码器创建 渲染通道编码器

   > 关于编码器，请回顾 “[WebGPU基础之渲染打包器(GPURenderBundleEncoder)和查询集(GPUQuerySet)](https://github.com/puxiao/webgpu-tutorial/blob/main/14%20WebGPU%E5%9F%BA%E7%A1%80%E4%B9%8B%E6%B8%B2%E6%9F%93%E6%89%93%E5%8C%85%E5%99%A8(GPURenderBundleEncoder)%E5%92%8C%E6%9F%A5%E8%AF%A2%E9%9B%86(GPUQuerySet).md)” 一文。

5. 获取 画布上下文中的纹理视图，并交给 渲染通道(GPURenderPass) 供其操作

   > 关于 纹理，请回顾 “[WebGPU基础之纹理(GPUTexture)与纹理视图(GPUTextureView)](https://github.com/puxiao/webgpu-tutorial/blob/main/07%20WebGPU%E5%9F%BA%E7%A1%80%E4%B9%8B%E7%BA%B9%E7%90%86(GPUTexture)%E4%B8%8E%E7%BA%B9%E7%90%86%E8%A7%86%E5%9B%BE(GPUTextureView).md)” 一文。

   > 重要的事情这里再说一遍：WebGPU 与 WebGL 一个非常大的不同点在于，他们各自的画布上下文 GPUCanvasContext 和 WebGLRenderingContext 所肩负的功能责任不同。
   >
   > GPUCanvasContext 仅仅作为 Canvas 与 WebGPU 沟通的桥梁，**其自身相当于 WebGPU 中的一个纹理**，除此之外再无别的其他作用。但是 WebGLRenderingContext 除了上述职责外，还包括创建 WebGL 子对象、绑定、编译着色器、触发渲染等等。

6. 最终 渲染通道编码器 根据各项配置和渲染内容，完成打包工作，得到命令缓冲对象，并通过 显卡设备的命令队列提交给系统的真正显卡(GPU)

   > 关于 缓冲区，请回顾 “[WebGPU基础之缓冲区(GPUBuffer)](https://github.com/puxiao/webgpu-tutorial/blob/main/06%20WebGPU%E5%9F%BA%E7%A1%80%E4%B9%8B%E7%BC%93%E5%86%B2%E5%8C%BA(GPUBuffer).md)” 一文。
   >
   > 关于 命令队列，请回顾 “[WebGPU基础之命令队列(GPUQueue)](https://github.com/puxiao/webgpu-tutorial/blob/main/05%20WebGPU%E5%9F%BA%E7%A1%80%E4%B9%8B%E5%91%BD%E4%BB%A4%E9%98%9F%E5%88%97(GPUQueue).md)” 一文
   >
   > 关于 命令编码器、命令缓冲，请回顾 “[WebGPU基础之命令缓冲区(GPUCommandBuffer)与命令编码器(GPUCommandEncoder)](https://github.com/puxiao/webgpu-tutorial/blob/main/12%20WebGPU%E5%9F%BA%E7%A1%80%E4%B9%8B%E5%91%BD%E4%BB%A4%E7%BC%93%E5%86%B2%E5%8C%BA(GPUCommandBuffer)%E4%B8%8E%E5%91%BD%E4%BB%A4%E7%BC%96%E7%A0%81%E5%99%A8(GPUCommandEncoder).md)” 一文

7. 真正的显卡(图形)处理器处理完成后，直接将结果反馈到画布中

   > 这一步操作是无需我们介入的。
   >
   > 换句话说对于 WebGPU 渲染而言，我们要做的就是将 渲染命令(内容) 提交到缓冲区，至于显卡(图形)处理器什么时候处理好，我们无需等待，也无需处理。



<br>

除了 采样器、资源绑定 和 打包器 外，上面步骤所涉及的知识点，已经涵盖了在前一阶段我们所有的学习篇章。



<br>

> 如果你是第一次看我的文章，第一次接触 WebGPU，那么我强烈建议你先去看一下本教程前面十几篇 WebGPU 基础文章。

> 本文是假定你已经看过前面的 WebGPU 基础知识，所以对于很多 名词 没有做过多讲解和解释。



<br>

回到具体的代码中，直接贴出 绘制一个三角形 完整的代码。



<br>

#### 完整的代码

```diff
import { useEffect, useRef } from "react"
import useWebGPU from "@/hooks/useWebGPU"

+ import vert from '@/shaders/triangle.vert.wgsl'
+ import frag from '@/shaders/red.frag.wgsl'

const SimpleTriangle = () => {

    const canvasRef = useRef<HTMLCanvasElement | null>(null)
    const { adapter, device, canvas, context, format } = useWebGPU(canvasRef.current)

    useEffect(() => {
        if (!canvas || !context || !adapter || !device) return

+        const canvsConfig: GPUCanvasConfiguration = {
+            device,
+            format,
+            size: [canvas.clientWidth, canvas.clientHeight],
+            compositingAlphaMode: 'opaque'
+        }
+        context.configure(canvsConfig)

+        const pipeline = device.createRenderPipeline({
+            vertex: {
+                module: device.createShaderModule({
+                    code: vert
+                }),
+                entryPoint: 'main'
+            },
+            fragment: {
+                module: device.createShaderModule({
+                    code: frag
+                }),
+                entryPoint: 'main',
+                targets: [{ format }]
+            },
+            primitive: {
+                topology: 'triangle-list'
+            }
+        })

+        const draw = () => {
+            const commandEncoder = device.createCommandEncoder()
+            const textureView = context.getCurrentTexture().createView()
+            const renderPassDescriptor: GPURenderPassDescriptor = {
+                colorAttachments: [{
+                    view: textureView,
+                    clearValue: { r: 0, g: 0, b: 0, a: 1 },
+                    loadOp: 'clear',
+                    storeOp: 'store'
+                }]
+            }
+            const passEncoder = commandEncoder.beginRenderPass(renderPassDescriptor)
+            passEncoder.setPipeline(pipeline)
+            passEncoder.draw(3, 1, 0, 0)
+            passEncoder.end()

+            device.queue.submit([commandEncoder.finish()])
+        }
+        draw()

    }, [canvas, context, format, adapter, device])

    return (
        <canvas ref={canvasRef} width={document.body.clientWidth} height={document.body.clientHeight} tabIndex={0} />
    )
}

export default SimpleTrangle
```



<br>

从上往下，我们对新增的代码进行逐一讲解。



<br>

```
import vert from '@/shaders/triangle.vert.wgsl'
import frag from '@/shaders/red.frag.wgsl'
```

**引入 .wgsl 文件**

尽管我们还没有系统学习 WGSL 语法，但看一下这 2 个 .wgsl 文件的内容，大致能猜出其含义。

> 特别强调：**在 WGSL 中的数字是需要统一小数位的，如果你将下面代码中的 1.0 改为 1 ，那么程序是无法正常运行的。**

> src/shaders/triangle.vert.wgsl

```
@stage(vertex)
fn main(@builtin(vertex_index) VertexIndex: u32) -> @builtin(position) vec4<f32> {
    var pos = array<vec2<f32>, 3>(
        vec2<f32>(0.0, 0.5),
        vec2<f32>(-0.5, -0.5),
        vec2<f32>(0.5, -0.5)
    );

    return vec4<f32>(pos[VertexIndex], 0.0, 1.0);
}
```

我也没系统学习过 WGSL，所以下面的一些解读一部分是我查资料，一部分是我瞎猜的。

1. `@stage(vertex)`：用于表明接下来的代码是在 vertex(顶点) 阶段中使用的

2. `fn main(...)`：fn 是 function 的简写，相当于声明一个名为 main 的函数

3. `@builtin` 是 “内置” 的意思

4. ` -> @builtin(position) vec4<f32>`：表示 main 函数的返回结果

5. `var pos = array<vec2<f32>, 3>`：基于二维坐标，定义一个数组，同时明确这个数组中的元素总个数 为 3

   > 这里的 pos 是 position(位置) 的简写

   > 为什么是 二维坐标？因为在后面 `return vec4...` 时会固定增加后 2 个分量(0.0,1.0) ，最终组成了一个 vec4。
   >
   
6. `vec2<f32>(0.0, 0.5)...`：定义了 3 个二维坐标(x,y)

7. `return vec4<f32>...`：每次执行 main 函数，会返回一个 vec4

   > 该 vec4 的前 2 个分量为 x, y，但是后 2 个分量 0.0, 1.0 究竟是什么含义，我们本文先不做解释。你可以尝试将其修改成别的值试一下看渲染出的结果发生什么变化。



<br>

> src/shaders/red.frag.wgsl

```
@stage(fragment)
fn main() -> @location(0) vec4<f32> {
    return vec4<f32>(1.0, 0.0, 0.0, 1.0);
}
```

1. `@stage(fragment)`：表明下面的代码是在 fragment(片元) 阶段中使用的

2. `fn main()`：声明一个 main 的函数

3. `-> @location(0) vec4<f32>`：表明函数会返回一个 vec4

4. `return vec4<f32>(1.0, 0.0, 0.0, 1.0)`：返回一个具体的 vec4，这里的 vec4 并不是指 “四维坐标”，而是表明 “包含 4 个分量”的一个颜色值，即 rgba 的 4 个值。

   而 `(1.0, 0.0, 0.0, 1.0)` 即 r 为 1.0 (即255)，b、g 为 0.0 (即 0)，a 为 1.0，这个颜色对应的是红色。

   > 为什么这里颜色使用的是 rgba 而不是 bgra ？额~，反正就是这样规定的。



<br>

```
const canvsConfig: GPUCanvasConfiguration = { ... }
context.configure(canvsConfig)
```

**配置画布上下文**

这点我们在之前的文章中讲解过，不再过多细说。



<br>

```
import vert from '@/shaders/triangle.vert.wgsl'
import frag from '@/shaders/red.frag.wgsl'

const pipeline = device.createRenderPipeline({
    vertex: {
        module: device.createShaderModule({
            code: vert
        }),
        entryPoint: 'main'
    },
    fragment: {
        module: device.createShaderModule({
            code: frag
        }),
        entryPoint: 'main',
        targets: [{ format }]
    },
    primitive: {
        topology: 'triangle-list'
    }
})
```

**创建渲染管线**

1. `module: device.createShaderModule({ code: xxx})`：将引入(import) 的 vert 和 frag 的值作为 顶点和片元的 “代码内容”

2. `entryPoint:'mian'` ：指出函数名为 "mian"

3. `targets: [{ format }]`：指出片元光栅化阶段 目标输出 颜色格式

   > 这里的 format 就是我们从 GPU 画布上下文中得到当前设备最佳的颜色格式

4. `topology: 'triangle-list'`：指出 “原始组装” 点的方式为 triangle-list(每 3 个点组合出 1 个三角形)

   > topology 一共有以下几种可选值：
   >
   > 1. "point-list"：每 1 个点都代表 1 个单独的点
   >
   > 2. "line-list"：每 2 个点组成 1 根线
   >
   > 3. "line-strip"：后 1 根线沿用 上一根线的 最后一个节点，构成一根新的线
   >
   >    举例：假设有 a,b,c 3 个点，那么 a 和 b 先组成一根线，之后 b 和 c 再组成另外一根线
   >
   >    > 简单来说就是 复用 上一根线的最后一个点
   >
   > 4. "triangle-list"：每 3 个点组成 1 个三角形
   >
   >    > 这意味着 点 的总个数必须是 3 的整倍数
   >
   > 5. "triangle-strip"：后 1 个三角形沿用上一个三角形的最后 2 个点，构成一个新的三角形
   >
   >    举例：假设有 a,b,c,d 4 个点，那么先由 a,b,c 组成一个三角形，然后再由 b,c,d 组成另外一个三角形



<br>

**延展变化：其他多边形**

1. 假设我们把上面示例中的 `src/shaders/triangle.vert.wgsl` 再增加 1 个点坐标，也就是一共有 4 个点坐标

   > 同时记得修改此处代码：
   >
   > ```diff
   > - var pos = array<vec2<f32>, 3>
   > + var pos = array<vec2<f32>, 4>
   > ```

2. 同时将 `topology` 的值设置为  “triangle-strip”，那么最终我们可能得到一个 四边形 或者其他图形(这取决于你这 4 个点的前后顺序)

   > 同时记得修改此处代码：
   >
   > ```diff
   > - passEncoder.draw(3, 1, 0, 0)
   > + passEncoder.draw(4, 1, 0, 0)
   > ```



<br>

```
const draw = () => { ... }
draw()
```

**开始绘制**

1. `device.createCommandEncoder()`：用同步的方法，创建一个命令编码器

   > 与之对应的是 异步创建 的方法：device.createComputePipelineAsync()，这里我们暂时先选择使用同步的方法，若日后需要同时创建大量的 命令编码器时，我们再选择使用异步。

2. `context.getCurrentTexture().createView()`：得到画布上下文中的纹理视图

3. `const renderPassDescriptor: GPURenderPassDescriptor = { ... }`：创建初始化渲染通道编码器的配置项

4. `clearValue: { r: 0, g: 0, b: 0, a: 1 }`：设置默认的背景色 (黑色)

   > 这里的 "clearValue" 表达的意思是 “当清空上一帧后，在绘制新的一帧之前 给整个场景所填充的颜色”
   >
   > 如果修改为 `clearValue: { r: 1, g: 1, b: 1, a: 1 }`，那么背景色将会为 纯白色

5. `commandEncoder.beginRenderPass(renderPassDescriptor)`：创建一个渲染通道编码器

6. `passEncoder.setPipeline(pipeline)`：将之前创建的管线交给该渲染通道编码器

7. `passEncoder.draw(3, 1, 0, 0)`：开始绘制，其中第 1 个参数表明调用 "main" 这个函数的次数，后  3 个参数为可选参数，这里使用的是它们的默认值 “1,0,0”，实际上这行代码可以简写为 `passEncoder.draw(3)`

   > 这一步发生在 CPU 中，这里的绘制不是指 图形绘制，而是指针对 绘制命令的编码(转录)

8. `passEncoder.end()`：结束绘制

9. `device.queue.submit([commandEncoder.finish()])`：调用 命令编码器的 .finish() 方法完成编码，并将其返回值 命令缓冲区 交给 命令队列(GPUQueue)，并通过 .submit() 函数进行提交

10. 接下来会在 显卡(GPU) 中开始真的 GPU 绘制，并将结果反映在 画布元素中



<br>

执行 `yarn start`，就可以在支持 WebGPU 的浏览器中，看到下面这个渲染结果：

![red_triangle.jpg](https://raw.githubusercontent.com/puxiao/webgpu-tutorial/main/imgs/red_triangle.jpg)



<br>

**当画布元素尺寸发生变化时：**

当 画布(canvas) 尺寸发生变化时，我们需要重新配置一下 画布上下文，根据更改后的画布尺寸(width,height) 重新配置一下 size 的值。

重新配置过后，我们再次执行上述代码中的 draw() 函数即可重新绘制。

> 请注意，这里的 “重新绘制” 是指对于 内容的重新渲染(相当于重新生成了一份纹理)，但不会对 画布保持等比的处理。
>
> 对于本示例中的三角形假设你修改浏览器窗口尺寸，三角形并不会保持等比，而是会变形的。
>
> 那究竟该怎么做可以让三角形永远等比(不变形)显示呢？这个问题我们留到以后再解决。



<br>

画布尺寸变化的 2 种监听方式：

1. `window.addEventListener('resize', handleResize)`

2. `resizeObserver.observe(canvas)`

   > resizeObserver 这个新的 API 虽然还未正式成为 W3C 的标准，但是目前除 IE 以外的浏览器都已经支持了该接口，所以是可以放心使用的。
   >
   > https://developer.mozilla.org/zh-CN/docs/Web/API/ResizeObserver



<br>

第1 种对应的代码：

```
const handleResize = () => {
    canvsConfig.size = [canvas.clientWidth, canvas.clientHeight]
    context.configure(canvsConfig)
    draw()
}
window.addEventListener('resize', handleResize)

return () => {
    window.removeEventListener('resize', handleResize)
}
```



<br>

第 2 种对应的代码：

```
const resizeObserver = new ResizeObserver(entries => {
    for (const entry of entries) {
        if (entry.target !== canvas) continue
        canvsConfig.size = [canvas.clientWidth, canvas.clientHeight]
        context.configure(canvsConfig)
        draw()
    }
})
resizeObserver.observe(canvas)

return () =>{
    resizeObserver.unobserve(canvas)
    //resizeObserver.disconnect()
}
```



<br>

虽然此刻我们仅仅绘制了一个三角形，但是这其中的渲染流程中的细节非常多。一定要反复多敲代码几遍，将绘制流程熟记于心。

好了，作为 WebGPU 实际入门的示例，绘制一个三角形，本文讲解到此结束。



<br>

我也是正在学习中，如果有讲错的地方，或者由更优雅的代码书写方式，欢迎直接留言或提交 Issues：

https://github.com/puxiao/react-webgpu-samples/issues



<br>

接下来，我们将要学习如何动态更新管线，达到修改这个三角形  颜色，位置，大小等。