# 18 WebGPU示例之顶点缓冲区(VertexBuffer)

**本文将学习顶点缓冲区(VertexBuffer)：如何通过创建和修改顶点缓冲区中的顶点坐标信息来动态更改 WebGPU 中顶点管线所用到的顶点数据。**



<br>

往大的来说，就是 “CPU 和 GPU 如何共同操作(读/写)某块缓冲区”。

往中的来说，就是 “JS如何动态更改 WebGPU 顶点管线中的顶点数据”。

往小的来说，就是“JS如何动态更改绘制一个三角形的位置信息”。



<br>

这里我假定你一块跟着我先学习过了 WGSL 的基础知识。

如果你对 WGSL 基础知识一点不了解，那么我建议你先阅读我之前写的 WGSL 基础教程。



<br>

从本文开始，我们将把之前学习过的 WebGPU + WGSL 一块深入学习。



<br>

**先回顾一下之前我们是如何绘制一个三角形的：**

1. 搭建好一个 React + TypeScript + @webgpu/types 的项目

2. 本机安装谷歌金丝雀版本浏览器，并通过 `chrome://flags` 开启 webgpu

3. 开始正式编写代码：获取 画布(canvas)，画布上下文(GPUCanvasContext)，显卡适配器(GPUAdapter)，显卡设备(GPUDevice)，当前设备最佳纹理格式('bgra8unorm')

   > 特别强调 2 点：
   >
   > 1. 对于绝大多数电脑而言，最佳纹理格式都是 'bgra8unorm'
   >
   > 2. WebGPU API 目前依然处于剧烈、破坏性更新中，目前获取最佳纹理格式的方式相对于我们上一篇文章中的示例代码，已经发生了变化：
   >
   >    ```diff
   >    - const context = canvas.getContext('webgpu')
   >    - const preferredFormat = context.getPreferredFormat(adapter)
   >    
   >    + navigator.gpu.getPreferredCanvasFormat()
   >    ```

4. 配置画布上下文，显卡设备(GPUDevice)，最佳纹理格式，画布尺寸(宽，高)，图像合成模式

   > 目前我们示例中选择的是不透明模式'opaque'，而不是颜色预乘模式 'premultiplied'

5. 创建管线(GPURenderPipeline)，并设置该管线中具体的 顶点阶段、片元阶段、顶点原始数据迭代模式

6. 其中顶点和片元阶段中，分别设置对应的着色器代码(通过引入 .wgsl 文件)，明确入口函数名('main')

   > 本教程示例使用的是 React18 + webpack5，已经内置 raw-loader ，所以只需要通过 craco.config.ts 中配置规则即可：`{ test: /\.wgsl$/, use: "raw-loader" }`
   >
   > 如果是 vue 框架，则使用 `import xxx from './shader/xx.wgsl?raw'` 这种形式引入

7. 创建命令编码器(GPUCommandEncoder)、画布纹理视图(GPUTextureView)

8. 创建渲染通道编码器(GPURenderPassEncoder)，并将前面创建好的管线写入其中

9. 调用通道编码器的 .draw(3) 函数来进行数据录制，并执行 .end() 方法来结束录制

10. 命令编码器执行 .finish() 函数结束编码，得到命令缓冲区，然后将其通过命令队列(GPUQueue)的 .submit() 方法将 命令缓冲区 传递给显卡，至此 JS 中的工作结束，接下来由 显卡开始绘制



<br>

**顶点着色器代码解读**

在上面的操作步骤中，顶点着色器 vert.wgsl 文件代码内容为：

```
@stage(vertex)
fn main(@builtin(vertex_index) VertexIndex: u32) -> @builtin(position) vec4<f32> {
    var pos = array<vec2<f32>, 3>(
        vec2<f32>(0.0, 0.5),
        vec2<f32>(-0.5, -0.5),
        vec2<f32>(0.5, -0.5)
    );

    return vec4<f32>();
}
```



<br>

我们已学习了 WGSL 基础知识，所以此时时刻相对容易读懂上面的代码了。

1. `@stage(vertex)`：明确本着色器代码脚本应用于 顶点着色阶段

2. `fn main( ... )`：声明一个名为 main 的函数，相当于 JS 中的 `function main()`

3. `@builtin(vertex_index)`：通过 `@builtin` 关键字来强调使用内置值 vertex_index

   > builtin 单词本意就是 内置

   > 在 WGSL 中不同着色器阶段有很多内置的值，具体的可查阅 我写的 WGSL 基础教程中的 “WGSL基础之内置值和内置函数.md” 那篇文章

4. `VertexIndex: u32`：将内置值 vertex_index 关联到一个名为 VertexIndex 的自定义参数，且明确它的类型为 u32 (32位正整数)

   > 由于内置值 vertex_index 的类型为 u32，所以这里将 VertexIndex 的内置值类型也定义为 u32
   >
   > 在顶点渲染阶段，每一次读取一个顶点信息，内置值 vertex_index 都会自动 + 1

   > 关于内置值 vertex_index，它特别像 JS 数组循环方法中的索引下标(index)，例如 `['a','b','c'].map((item, index) => { ... })` 中的 index

5. `-> @builtin(position) vec4<f32>`：表明入口函数 main 每次调度后都会返回内置值 position，且它的类型为  `vec4<f32>`

   > `vec4<f32>`：由 4 个分量组成的一个齐次坐标，每一个分量的类型都是 f32 (32位浮点数)

6. `var pos = array<vec2<f32>,3>`：声明一个名为 pos 的数组，强调该数组长度为 3，且每个元素类型为 `vec2<f32>`

   > 一个 3维空间坐标应该是 x, y, z，而此处只是明确了 x, y 的值，因为后面我们会统一设定 z 的值都是 0.0

7. `vec2<f32>(0.0, 0.5),...`：依次设置 pos 的 3 个元素值

8. `return vec4<f32>( ... )`：每次返回一个齐次坐标

9. `pos[VertexIndex], 0.0, 1.0`：每次返回的齐次坐标值为 "pos[VertexIndex], 0.0, 1.0"



<br>

**关于 WGSL 空间坐标的知识点补充：**

1. WGSL 中 3D 空间坐标你可以把它想象成是一个 “平面的纹理贴图”，想象一下此时此刻你打开了一张壁纸图片
2. 在这个 “平面的纹理贴图”中，左右方向为 x 轴，最左侧为 -1，最右侧为 1
3. 上下方向为 y 轴，最下面是 -1，最上面是 1
4. 与该 “平面的纹理贴图” 垂直，假设你此时打开了一张壁纸图片，从显示器屏幕向里发射，产生一个垂直于屏幕的轴，这个轴为 Z 轴，屏幕的位置 z 轴值为 0，向里面延伸，最里面的值为 1



<br>

按照上面的讲述，这张 “平面的纹理贴图”：

1. 左上角坐标为 (-1.0, 1.0, 0.0)
2. 左下角坐标为 (-1.0, -1.0, 0.0)
3. 右上角坐标为 (1.0, 1.0, 0.0)
4. 右下角坐标为 (1.0, -1.0, 0.0)
5. 本文我们先不讨论 z 轴的值，示例中 z 的值我们姑且都先设置为 0.0



<br>

以上是我们对之前学习过的，最基础的绘制一个三角形的回顾。

回到本文的学习中，

今天我们要学习的是：解如何动态传入顶点坐标数据，而不是每次读取写死到 .wgsl 中坐标。



<br>

**想象一下，如果不是 WGSL，而是 JS 函数，我们怎么实现？**

很简单嘛，既然是作为参数传入，并最终返回出去，那 JS 代码可能长这个样子：

```
function main(vec2){
    return [...vec2, 0.0, 1.0]
}
```

额~，由于 WGSL 属于静态类型语言，所以使用 TS 会更接近 WGSL 的代码风格：

```
type vec2 = [f32, f32]
type vec4 = [f32, f32, f32, f32]

//@stage(vertex)
function main(vec: vec2): vec4 {
    return [...vec, 0.0, 1.0]
}
```



<br>

说了这么多，那么在 WGSL 语法中该如何编写 main 的代码呢？

**在 WGSL 语法中使用 @location() 关键词来定义入口函数的参数。**

> 与其说是 定义 不如说是 承接 更为贴切

1. 如果是函数的第 1 个参数，则使用 `@location(0)`
2. 如果是函数的第 2 个参数，则使用 `@location(1)`
3. 如果是...



<br>

> 作为前端开发者，习惯了 JS/TS 语法，可能会对上述这种定义函数参数的语法形式很不适应。
>
> 不要惊讶，也不要怀疑，多写几次就会习惯了。



<br>

**最终，改造过后，可承接参数的 main 的代码：**

```
@stage(vertex)
fn main(@location(0) pos:vec2<f32>) -> @builtin(position) vec4<f32>{
    return vec4<f32>(pos, 0.0, 1.0)
}
```

1. `@location(0)`：通过 `@location()` 来明确我们即将定义函数的第1个参数
2. `pos:vec2<f32>`：这个参数名定义为 pos，它的类型为 `vec2<f32>`
3. `return vec4<f32>(pos, 0.0, 1.0)`：将参数 pos 与 另外 2 个固定值组成一个 vec4 并返回出去



<br>

好了，至此关于可动态接收位置信息的顶点着色器我们已经改造完成了。

接下来我们思考一下：

1. 在 JS 中该如何定义这些从 顶点着色器中移出的 顶点坐标信息
2. JS 又是通过什么样的 WebGPU 代码，将这些 顶点坐标信息 传递给 GPU 的 



<br>

**在 JS 中定义顶点坐标数据一共分为 3 步：**

**第1步：在 JS 中使用 TypeArray 来保存原始的顶点坐标位置信息**

1. 必须使用连续内存的 TypeArray 对象，不可以是普通的数组 Array
2. 关于 TypeArray 的介绍，可查看 https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/TypedArray
3. TypeArray 有很多具体的数据类型，在本示例以及绝大多数情况下，顶点坐标我们都会选择 Float32Array 这种类型

具体的代码如下：

```
const vertexArray = new Float32Array([
    0.0, 0.5,
    -0.5, -0.5,
    0.5, -0.5
])
```



<br>

**第2步：通过调用 device.createBuffer() 方法申请到一块连续的 GPU 缓冲区**

1. size：在申请(创建) 缓冲区时，需要明确所需要申请的缓冲区的字节数(size)总大小，请记住每一个 32位浮点数 占 4 个字节
2. usage：同时明确告知 GPU 我们申请的缓冲区的用途标记

具体的代码如下：

```
const vertexBuffer = device.createBuffer({
    size: vertexArray.byteLength, // vertex.length * 4
    usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST
})
```

> 1. GPUBufferUsage.VERTEX：表明该缓冲区是应用于 顶点阶段
> 2. GPUBufferUsage.COPY_DST：表明该缓冲区是可以作为拷贝目的地(destination)，也就是说可以被 写入



<br>

**第3步：通过 queue.writeBuffer() 将 TypeArray 中的数据写入到 申请的 缓冲区中**

这一步没什么好讲的。

具体代码如下：

```
device.queue.writeBuffer(vertexBuffer, 0, vertexArray)
```

> 第 2 个参数为写入的偏差数值，设置为 0 即表示从 vertexArray 的第 0 个字节开始写入到 vertexBuffer 中



<br>

至此，我们已经将 JS 中定义的顶点坐标数据写入到 缓冲区 中了。

接下来我们该配置管线，让管线知道并使用这个顶点缓冲区。



<br>

**将渲染管线与 顶点缓冲区 进行关联一共需要 2 步：**

**第1步：创建渲染管线时，在它的 顶点 阶段添加 buffers 属性值**

具体代码如下：

```diff
const pipeline = device.createRenderPipeline({
     layout: 'auto',
     vertex: {
         module: device.createShaderModule({
             code: vert
         }),
         entryPoint: 'main',
+        buffers: [{
+            arrayStride: 2 * 4,
+            attributes: [{
+                shaderLocation: 0,
+                offset: 0,
+                format: 'float32x2'
+            }]
+        }]
     },
     fragment: { ... },
     primitive: {
         topology: 'triangle-strip'
     }
}
```

1. `layout: 'auto'`：目前最新版的 WGSL API 中，创建管线时 layout(布局) 变成了必填项，之前是可以不填的，但现在必须填写。这里我们就使用 'auto' 值就可以了

   所谓的 layout(布局) 可以简单把它想象成这是 PhotoShop 图像处理软件中的 “图层叠加的顺序值”

2. `arrayStride: 2 * 4`：stride 单词是 步幅 的意思，所谓 arrayStride 就是指每次读取的字节数应该是多少。由于我们本示例中一个顶点坐标仅仅声明了 x,  y 2个分量信息，且每个分量信息字节数为 4，所以我们才会将 arrayStride 的值设置为 2 * 4

3. `shaderLocation: 0`：明确每一次读取实际对应的是 WGSL 顶点着色器入口函数的第 0 个参数。

   这里与 vert.wgsl 中的 `@location(0)` 是呼应的

4. `offset: 0`：读取时的起始索引偏差，我们设置为 0，即若每次读取步幅 arrayStride 为 8，则从这 8 个字节的第 0 字节开始读取

5. `format: 'float32x2'`：每一次读取得到的结果类型，"float32x2" 表示为 2 个 32 位浮点数

6. 我们注意到上面配置代码中 buffers 实际是一个数组，也就意味着 我们可以传递多个 缓冲区 到入口函数中，也意味着 main 可以接收多个参数。

   但是本文中我们就只传递进去一个 缓冲区 即可。

   > 换句话说就是：我们的入口函数只需要传入 1 个参数



<br>

上面创建管线时关于配置顶点缓冲区的操作，在《Vulkan 编程指南》中被称为 “**顶点输入描述**”。

> 输入 就是 input，也就是 I/O 中的 I



<br>

**第2步：当通道编码器设置(添加)过管线后，我们把顶点缓冲区也添加其中。**

这样 通道编码器 拥有了 管线，同时也拥有了我们定义好的 顶点缓冲区，就可以提交到显卡中进行绘制了。

具体代码如下：

```diff
  passEncoder.setPipeline(pipeline)
+ passEncoder.setVertexBuffer(0,vertexBuffer)
```

> 上述代码中 setVertexBuffer() 中的第 1 个参数值 0 是指插槽的索引值，第 2 个参数 vertexBuffer 就是这个插槽中对应的 缓冲区数据。
>
> 特别强调：一个插槽可以包含多个参数值，上述代码中的 插槽索引值 为 0 并不意味着第 2 个参数 vertexBuffer 一定 100% 只对应入口函数的第 1 个参数，vertexBuffer 中的数据还有可能包含入口函数的第 2 个，第 3 个 ... 第 n 个参数。



<br>

回顾一下我们上面的几个操作步骤：

1. 在 JS 中通过 TypeArray 定义顶点坐标信息

   > 此时 顶点数据是在 CPU 中保存着

2. 申请到一块固定大小的 顶点缓冲区

3. 将 TypeArray 中的数据写入到这个 GPU 缓冲区中

   > 此时 顶点数据是在 GPU 中保存着

4. 创建渲染管线时，在顶点阶段中关联到我们申请到的 顶点缓冲区

5. 在通道编码器中添加 管线 和 顶点缓冲区

6. 最后执行 通道编码器的 end() 方法，将这个通道编码器 提交到真正的 GPU 渲染中

   > 此操作完成后，GPU 就可以读取该 顶点缓冲区了



<br>

从顶点数据保管和控制权的视角来看：

1. 上述第 1 - 2 步时，控制权归 CPU，( JS 是运行在 CPU 中的一个进程)

   > 此时 CPU (也就是 JS ) 可以操作这块 显存  (缓冲区)

2. 上述第 3 - 6 步时，控制权转到了一个 CPU和GPU 控制的过渡区域

   > 此时 CPU 和 GPU 都不可以操作这块 显存 (缓冲区)

3. 上述第 6 步完成后，控制权发生转移，此时完全由 GPU 来控制

   > 此后 GPU (也就是 WebGPU) 可以操作这块 显存 (缓冲区)

<br>

> 这就是我们本文开头的那句话：`往大了说，就是  “CPU 和 GPU 如何共同操作(读/写)某块缓冲区”`



<br>

> 以上言论观点，仅为个人目前的认知，如果讲错了，还请理解和告知。



<br>

至此，已经讲解了一遍 如何在 JS 中定义顶点坐标，并将坐标传递给 WGSL 顶点着色器入口函数中。



<br>

尽管实现步骤和细节讲得很清楚，但是如果你不去实际敲几遍代码，可能还是没有掌握。

一定多敲几遍，记下这些套路，回过头看你会发现，实际上并没有多少代码。



<br>

> 后续学习的其他很多操作，例如 资源绑定，添加纹理等等，都几乎是相同的套路。



<br>

**非常有用的思维补充：插槽(slot)**

本文讲解的是如何通过 JS 来更新 WGSL 顶点着色阶段中 顶点的坐标信息。

我们换一个思维场景：回到我们最擅长的 Vue 或 React 中

1. 假设我们把 WGSL 顶点着色器 想象成是一个 Vue/React 组件
2. 那么所谓 JS 更新 WGSL 顶点着色器中的顶点坐标数据，就可以看作是 对 Vue/React 组件中传入和更新 props 值，从而达到更新这个组件显示内容的目的。



<br>

举个最简单的例子：假设在 Vue/React 中有一个组件，这个组件用于显示一个人的姓名和年龄

```
const PersonComponent = ({ name, age }) => {
    return (
        <div>
            <h2>{ name }</h2>
            <span>{ age }</span>
        </div>
    )
}
```

在上面组件中，我们实际上相当于定义了 2 个插槽：name 、age

当传入不同的 name 和 age 值时，组件会自动更新显示内容。



<br>

```
@stage(vertex)
fn main(@location(0) pos:vec2<f32>) -> @builtin(position) vec4<f32>{
    return vec4<f32>(pos, 0.0, 1.0)
}
```

**`@location(0) pos:vec2<f32>`、pos 就是我们 WGSL 顶点着色器中的那个插槽。**

> 当然本文前半截我们都是把 pos 当成 main 的参数来讲解的。



<br>

所谓更新 WGSL 顶点坐标 整个实现思维也就通顺了，没有什么玄幻的。



<br>

本文示例对应的完整代码：https://github.com/puxiao/react-webgpu-samples/blob/main/src/components/vertex-buffer-slot/index.tsx



<br>

**最后，特别感谢：**

1. Orillusion 的 WebGPU 教程：顶点插槽 & 资源绑定

   https://www.bilibili.com/video/BV13u411v7nN/

2. fangcun010 翻译的《Vulkan 编程指南》第22章 “顶点输入描述”、第 23 章 “创建顶点缓冲”：

   https://github.com/fangcun010/VulkanTutorialCN/blob/master/Vulkan%E7%BC%96%E7%A8%8B%E6%8C%87%E5%8D%97.pdf

> 你甚至可以先不看我写的教程，直接去读以上 2 个教程都可以。



<br>

**多说两句：**

1. WebGPU 的 API 大量借鉴了 Vulkan，所以想学习 WebGPU 可以去看 Vulkan 的文档资料，因为二者的理论知识、概念和用法 几乎都是一样的。

2. 未来和前端竞争 WebGPU 开发工作岗位的人或许就是搞 Vulkan 或 Unity 的人，他们会对我们进行降维打击。在我们看来 WebGPU 中一些非常难以理解的概念和知识，在他们看来那都是基础中的基础。

3. 尽管 “顶点插槽” 这个词很形象得表述了 动态更新顶点坐标位置 这个操作，但是我个人还是觉得应该慎用这个词。这个词我感觉是 Orillusion 自己造出来的，我查了一些 Vulkan 文章，他们并没有使用过这个词。

   我们姑且还是老老实实使用 “顶点缓冲区” 这个词语吧。
   
4. 经过大半年的运行，本人的微信公众号 “WebGPU” 截止到今天(2022.08.07)，已关注人数 104 了。

   ![](https://raw.githubusercontent.com/puxiao/webgpu-tutorial/main/imgs/me_qrcode02.jpg)

   我没有在一些前端平台，例如 掘金、思否 上发布过本系列教程，从来没有在微信交流群和微信朋友圈提及过我这个微信公众号。

   我把学习 WebGPU 这个过程看作是自己一次沉默的修行，苦练内功，提高自己，然后在未来某个机会中实现弯道超车。

   前端小伙伴们，与你共勉！



<br>

本文到此结束，下一节我们将讲解一下 顶点缓冲区 如何传递多个参数 。

可以把它当做是对本文的一个补充。

