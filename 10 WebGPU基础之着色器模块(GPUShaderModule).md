# 10 WebGPU基础之着色器模块(GPUShaderModule)

**着色器模块(GPUShaderModule)是对 WebGPU 内部编译的 WGSL 代码模块的引用，可以让我们得到它们的编译信息，例如消息、警告和错误。**



<br>

在学习 GPUShaderModule 之前，我们先简单了解一下 WGSL。



<br>

#### WGSL：WebGPU对应的着色器语言

WGSL 是 WebGPU Shading Language 的简写。

WGSL 的官方文档：

1. 官方地址1：https://www.w3.org/TR/WGSL/
2. 官方地址2：https://gpuweb.github.io/gpuweb/wgsl/
3. 非官方中文翻译版：https://www.orillusion.com/zh/wgsl.html

> 注：目前 WGSL 也是处于草案阶段，正式版并未发布。



<br>

WGSL 是 WebGPU 的着色器语言，是运行在 GPU 中的程序。

WebGPU 以 GPU 命令的形式向 GPU 发出一个工作单元，WGSL 关注两种 GPU 命令：

1. draw command(绘制命令)：在输入(inputs)、输出(outputs)和附加资源(resources)的上下文中执行 渲染管线。
2. dispatch command(调度命令)：在输入(inputs)和附加资源(resources)的上下文中执行 计算管线。

换句话说在 WebGPU 中上述两种管线都使用 WGSL 来编写着色器。



<br>

WGSL 算是一门 "编程"语言，它的代码组织形态和其他编程语言类似：

1. WGSL 也有 内置函数、关键字、数值、变量、常量、表达式、类型等。
2. WGSL 包含 布尔值、数组、向量、矩阵等类型。
3. 与 JS 不同的是，在 WGSL 中没有数字或布尔值类型之间的隐式转换或提升。
4. WGSL 也包含 纹理和采样器类型，用于图形渲染功能。
5. WGSL 除代码外，也可以添加注释，注释的形式和 JS 相同(使用 `//` 或 `/* */`)。



<br>

**JS 文件中添加 WGSL 代码的 2 种方式：**

1. 直接在 JS 中通过 反引号 包裹着文本类型的 WGSL 代码片段。

   ```
   const shaderSource = `
       var<private> pos : array<vec2<f32>, 3> = array<vec2<f32>, 3>(
           vec2(-1.0, -1.0), vec2(-1.0, 3.0), vec2(3.0, -1.0));
   
       @stage(vertex)
       fn vertexMain(@builtin(vertex_index) vertexIndex : u32) -> @builtin(position) vec4<f32> {
           return vec4(pos[input.vertexIndex], 1.0, 1.0);
       }
   
       @stage(fragment)
       fn fragmentMain() -> @location(0) vec4<f32> {
           return vec4(1.0, 0.0, 0.0, 1.0);
       }
   `;
   ```

   

2. 将 WGSL 代码保存在 `.wgsl` 后缀的文件中，然后在 JS 前端框架(例如React或Vue)中引入该文件。

   ```
   import xxxWGSL from './xxx/xx.wgsl'
   ```

   

<br>

关于 WGSL 本文不做过多讲解，我计划是学习完 WebGPU 后再系统学习一下 WGSL。

> 今年我给自己设定的学习计划是：WebGPU、WGSL、Rust



<br>

回到本文主题。



<br>

#### GPUShaderModule：着色器模块

我们可以通过 GPUDevice 的 .createShaderModule() 来创建一个 GPUShaderModule 实例。

```
createShaderModule(
  descriptor: GPUShaderModuleDescriptor
): GPUShaderModule;

interface GPUShaderModule extends GPUObjectBase {
  readonly __brand: "GPUShaderModule";
  compilationInfo(): Promise<GPUCompilationInfo>;
}

type GPUShaderModuleDescriptor = GPUShaderModuleDescriptorWGSL | GPUShaderModuleDescriptorSPIRV;

interface GPUShaderModuleDescriptorWGSL
  extends GPUObjectDescriptorBase {
  code: string;
  sourceMap?: object;
}

/** @deprecated */
interface GPUShaderModuleDescriptorSPIRV
  extends GPUObjectDescriptorBase {
  /** @deprecated */ code: Uint32Array;
}
```

> 注意：deprecated 单词的意思为 “不赞成的”。



<br>

从上面的 TS 类型定义可以看出，GPUShaderModule 实例只有一个名为 compilationInfo() 的方法。

**compilationInfo()**：通过异步的方式获取当前着色器模块(GPUShaderModule) 中 WGSL 代码在编译时所产生的编译信息。

> 请注意，这是一个异步函数。



<br>

接下来我们讲解一下创建着色器模块的 createShaderModule() 所需要传递的参数。



<br>

**createShaderModule() 的参数**

在 `@webgpu/types` 中，createShaderModule() 方法的必填参数类型为 GPUShaderModuleDescriptorWGSL 或 GPUShaderModuleDescriptorSPIRV。

我们先说一下被标记为 `@deprecated(不赞成)` 的 GPUShaderModuleDescriptorSPIRV。



<br>

#### 什么是SPIRV？

SPIR-V 是 Vulkan 使用的着色器语言。

> Vulkan 是 科纳斯组织(Khronos Group) 推出的一个跨平台 2D 和 3D 绘图应用程序接口(API)。
>
> 你可以简单得把 Vulkan 看作是用来取代 OpenGL 的。
>
> Vulkan 是 WebGPU 所支持的 3 大底层架构(DirectX 12、Metal、Vulkan)之一。



<br>

SPIR 是 Standard Portable Intermediate Representation 的缩写。

> Standard(标准)、Portable(便携式)、Intermediate(中间的)、Representation(表示)

SPIR 是一种用在 GPU 通用计算和图形学上的中间语言。

<br>

> 通用计算：general-purpose computing
>
> 所谓 通用计算 就是指 “啥都能算”，不仅仅是图形学相关计算。



<br>

> OpenGL 也是科纳斯组织(Khronos Group)的，主要用于桌面操作系统。
>
> OpenGL 的一个分支 OpenGL ES 则用于移动或嵌入设备上。
>
> OpenGL ES 的一个分支 WebGL 则用于浏览器。 



<br>

在 WGSL 的官方文档中写道：

1. WGSL 参考了 SPIR-V 的规范
2. WGSL 特性和语法与 SPIR-V 非常相似
3. WGSL 中所有的功能都可以转换为 SPIR-V



<br>

尽管如此，官方并不建议你在 WebGPU 中使用 SPIR-V。

在 `@webgpu/types` 中对于 GPUShaderModuleDescriptorSPIRV 特别添加了 `@deprecated(不赞成)` 的标签，官方更推荐你使用 WGSL。

关于 SPIR-V 我们不再做过多介绍，如果感兴趣你可以去看一下这个讨论帖子。

Possibility of SPIR-V and/or GLSL as a WebGPU extension ?

https://github.com/gpuweb/gpuweb/issues/847



<br>

接下来看一下 GPUShaderModuleDescriptorWGSL。

#### GPUShaderModuleDescriptorWGSL

```
interface GPUShaderModuleDescriptorWGSL
  extends GPUObjectDescriptorBase {
  code: string;
  sourceMap?: object;
}
```



<br>

**Code**：字符串形式的WGSL的代码

> 为了可以换行书写，所以都会采用 JS 中的 模板字符串形式。



<br>

**sourceMap**：可选参数，源码映射

这里的 sourceMap 的含义和平时我们在使用 Webpack 编译项目时的 sourceMap 非常相似。

也就是说通过 sourceMap 可以快速帮我们定位出错误信息(代码)位置。



<br>

补充：在 https://www.w3.org/TR/webgpu/#shader-module-creation 中还提到了另外一个配置参数项 `hints`，但是 `@webgpu/types` 中并未提到，所以我们就忽略它。

> 我们应当对 @webgpu/types 的维护者充满信任，他们没有定义 hints 那就别去理会它了。



<br>

**一个简单的示例：**

```
const shaderSource = `
    var<private> pos : array<vec2<f32>, 3> = array<vec2<f32>, 3>(
        vec2(-1.0, -1.0), vec2(-1.0, 3.0), vec2(3.0, -1.0));

    @stage(vertex)
    fn vertexMain(@builtin(vertex_index) vertexIndex : u32) -> @builtin(position) vec4<f32> {
        return vec4(pos[input.vertexIndex], 1.0, 1.0);
    }

    @stage(fragment)
    fn fragmentMain() -> @location(0) vec4<f32> {
        return vec4(1.0, 0.0, 0.0, 1.0);
    }
`;

const shaderModule = device.createShaderModule({
    code: shaderSource
})
```



<br>

着色器模块(GPUShaderModule) 实例创建好了，让我们去得到(捕获)它编译时它所对应的各种消息吧。

> 所谓 “编译时的各种消息” 就像我们日常写代码时的 打印信息、警告、错误信息。



<br>

#### GPUCompilationInfo：着色器模块编译信息

我们可以通过 GPUShaderModule 实例的 .compilationInfo() 方法来异步得到一个 GPUCompilationInfo 实例。

```
interface GPUCompilationInfo {
  readonly __brand: "GPUCompilationInfo";
  readonly messages: ReadonlyArray<GPUCompilationMessage>;
}
```

GPUCompilationInfo 实例只有一个只读属性 message，以数组形式存放着 N 条编译消息(GPUCompilationMessage)



<br>

> 在本文中将 info 翻译为 “信息"，将 message 翻译为 "消息(通知)"。



<br>

#### GPUCompilationMessage：一条编译消息

```
interface GPUCompilationMessage {
  readonly __brand: "GPUCompilationMessage";
  readonly message: string;
  readonly type: GPUCompilationMessageType;
  readonly lineNum: number;
  readonly linePos: number;
  readonly offset: number;
  readonly length: number;
}
```

**message**：消息的具体文字内容

**type**：消息的类型，一共有 3 种 "error"、"warning"、"info"

**lineNum**：警告或错误信息所对应的代码行号

**linePos**：警告或错误信息所对应的具体代码字符

**offset**：发生消息的着色器代码中从开头到错误的位置偏移数

**length**：发生消息的代码单元数量

> 对于我们而言 lineNum 和 linePos 已足够我们定位到代码的位置了。



<br>

**一个简单示例：**

```
shaderModule.compilationInfo().then(info => {
    info.messages.forEach(message => {
        console.log(message)
    })
})
```



<br>

着色器模块(GPUShaderModule) 并不复杂，比较容易理解，说白了就是用于捕获 WGSL 代码编译时所产生的各种消息。

真正复杂、有难度的是去编写 WGSL 代码。



<br>

本文到此结束，接下来我们要学习一个非常重要、有难度、你无法绕开的知识点：管线(Pipelines)。