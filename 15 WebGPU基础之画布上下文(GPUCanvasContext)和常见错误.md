# 15 WebGPU基础之画布上下文(GPUCanvasContext)和常见错误

**GPU画布上下文(GPUCanvasContext)用于提供画布(canvas)的 WebGPU 渲染上下文，其本身相当于 WebGPU 中的一个纹理，可以将 GPU 渲染的结果显示到画布(网页)当中。**



<br>

#### 获取 GPU 画布上下文

如果想在 `<Canvas />` 元素上绘制 2D 内容，那么首先需要得到一个 2D 画布的一个渲染上下文，代码如下：

```
const canvas = document.querySelector('#mycanvas')
const ctx = canvas.getContext('2d')
```



<br>

在 WebGL 时代，如果我们使用 Three.js 绘制 3D 场景，对于我们使用者而言是直接将 画布元素(canvas) 作为参数传递给 WebGLRenderer 的。

在 WebRenderer 内部，他会依次尝试使用 "webgl2"、"webgl"、"experimental-webgl" 来获取 画布 WebGL 上下文。



<br>

> “2d” 得到的是一个 CanvasRenderingContext2D 实例
>
>  “webgl” 得到的都是一个 WebGLRenderingContext 实例



<br>

而此刻，获取一个 GPU 画布上下文的方式和上面类似，就是把 .getContext() 的第一个参数写成 "webgpu" 。

```
const canvas = document.querySelector('#mycanvas')
const ctx = canvas.getContext('webgpu')
```

这样我们就得到一个 GPU画布上下文(GPUCanvasContext) 实例。



<br>

> 补充：对于 WebGL 而言，.getContext() 第 2 个参数用于设置 画布上下文的一些属性，例如：
>
> 1. alpha：表明画布是否背景透明
> 2. antialias：表明画布是否开启抗锯齿
> 3. ...
>
> 但这些对于 WebGPU 而言，是不需要设置第 2 个参数的。



<br>

若想获得 离屏画布(OffscreenCanvas) 的 GPU 画布上下文，则代码如下：

```
const offscreenCanvas = new OffscreenCanvas(400,300)
const ctx = offscreenCanvas.getContext('webgpu')
```

> 注：目前 火狐 和 苹果浏览器是不支持 离屏画布 的。
>
> 我不清楚是不是因为这个原因(或许根本不是这个原因)，目前在 TS (@webgpu/types) 环境下使用 OffscreenCanvas 时会提示错误。但是在原生 JS 环境下是可以正常运行的。
>
> ```
> Error: 'OffscreenCanvas' only refers to a type, but is being used as a value here.
> ```



<br>

#### GPUCanvasContext 与 WebGLRenderingContext 的差异之处

> 这段话来源于 `四季留歌` 的这篇文章：https://juejin.cn/post/7010673087586762766

GPUCanvasContext 与 WebGLRenderingContext 的最大不同点在于，GPUCanvasContext 仅仅作为 Canvas 与 WebGPU 沟通的桥梁，**其自身相当于 WebGPU 中的一个纹理**，除此之外再无别的其他作用。

但是 WebGLRenderingContext 除了上述职责外，还包括创建 WebGL 子对象、绑定、编译着色器、触发渲染等等。

> 这实际上也解释了为什么  .getContext("webgpu") 不需要第 2 个参数的原因，因为它(GPUCanvasContext) 根本并不负责 渲染相关操作，自然也不需要进行渲染相关的配置。



<br>

#### GPU画布上下文(GPUCanvasContext)的属性和方法

GPUCanvasContext 只有一个 .canvas 的只读属性，用于获取上下文对应的实际 canvas 元素。

```
readonly canvas: HTMLCanvasElement | OffscreenCanvas;
```



<br>

**getPreferredFormat()：获取当前系统首选的纹理格式**

```
getPreferredFormat(
    adapter: GPUAdapter
): GPUTextureFormat;
```

通常情况下，我们系统首选的纹理格式为 “bgra8unorm”。



<br>

**configure()：配置画布的上下文**

> 注意，配置画布的上下文是本文的学习重点。

```
configure(
    configuration: GPUCanvasConfiguration
): undefined;

interface GPUCanvasConfiguration {
    device: GPUDevice;
    format: GPUTextureFormat;
    usage?: GPUTextureUsageFlags;
    viewFormats?: Iterable<GPUTextureFormat>;
    colorSpace?: GPUPredefinedColorSpace;
    compositingAlphaMode?: GPUCanvasCompositingAlphaMode;
    size?: GPUExtent3D;
}
```

下面逐个介绍一下各个配置项。

1. **device**：与此画布上下文关联的 GPU设备(GPUDevice)

   > 注：我个人把 GPUDevice 翻译为 显卡设备，但实际上我们清除这个 “设备” 并不真的是系统的 GPU 硬件设备，而是 WebGPU 为了适配不同 GPU 而做出来的一个 “虚构” 的设备。

   > 我看某个人的文章，他把 GPUDevice 翻译为 “GPU驱动”... 额，无论翻译成什么，你明白其中含义即可。

2. **format**：设定 纹理(实际)格式，一般情况下我们都会将其设置为系统首选纹理格式

   ```
   const textformat = ctx.getPreferredFormat(adapter) //先得到首选纹理格式
   
   //将首选纹理格式作为 画布上下文 的纹理配置参数值
   ctx.configure({
       ...
       textform
       ...
   })
   ```

3. **usage**：纹理用途(usage)，默认为 0x10，表明是用作颜色附件的

4. **viewFormats**：设定纹理 除了实际格式(format)以外，调用 createView() 时其他被允许的格式

5. **colorSpace**：画布的颜色空间，该值目前只支持 "srgb"

6. **compositingAlphaMode**：合成到画布上的透明选项，默认为 “opaque” 即 不透明

   ```
   type GPUCanvasCompositingAlphaMode = "opaque" | "premultiplied";
   ```

   > "opaque" 表示不透明，会忽略 alpha 原有的值，换句话说相当于把 alpha 的值全部设置为 1
   >
   > "premultiplied" 表示 颜色透明度预乘
   >
   > 假设原有颜色值为 src，最终颜色值为 dst，那么 预乘 需要满足的公式为：
   >
   > `|dst.rgb = src.rgb + dst.rgb*(1-src.a)|`
   >
   > 注：上面颜色的 4 个分量 r g b a 他们的取值范围都是 0 ~ 1

   <br>

   > 以下内容来源于 `LaoYuanPython` 的一篇文章：图像处理术语解释：什么是PRGBA和Alpha预乘（Premultiplied Alpha ）
   >
   > https://blog.csdn.net/LaoYuanPython/article/details/106772363

   <br>

   > **名词解释：alpha预乘**
   >
   > 对于 4 通道 rgba 格式的图像(颜色)在合成时存在 2 个问题：
   >
   > 1. 在图像合成时需要对 r g b 通道应用 alpha 通道 ，这会造成 合成 时处理压力较大
   > 2. 在图像合成时可能需要进行 插值处理，但一般的插值计算都只是在 r g b 这 3 个通道中进行的，而带有 alpha 通道的图像的 r g b 并不是最终的 r g b 颜色
   >
   > 为了解决这个问题，在 图像处理 过程中引入了 预乘(premultiplied) 这个概念，经过预乘处理的图片格式称为 prgba。保存的数据通常称为(ar,ag,ab,a)，这样即保存了真正展现时的像素 rgb 值，又保存了 alpha 通道值。
   >
   > 由于 预乘 计算后保存的值是整数，会丢失小数点后的数值，因此会存在偏差，其中当 r g b 的值很小但 alpha 的值很大时，该误差会比较大。
   >
   > 注：这里说的 “保存的值是整数” 应该是指 0 - 255。

   > 关于 预乘 在 WebGPU 官方文档中有这样一句话：颜色值(rgb) 应该小于或等于他们的 alpha 值。例如 [1.0,0,0,0.5] 是 “超亮”，无法可靠显示。

7. **size**：画布的像素点数量

   ```
size?: GPUExtent3D;
   
   type GPUExtent3D = Iterable<GPUIntegerCoordinate> | GPUExtent3DDict;
   
   type GPUIntegerCoordinate = number;
   
   interface GPUExtent3DDict {
       width: GPUIntegerCoordinate;
       height?: GPUIntegerCoordinate;
       depthOrArrayLayers?: GPUIntegerCoordinate;
   }
   ```
   
   你可以有 2 种设置 size 的方式：

   > 注：这里说的画布宽高是指 canvas 元素实际的宽高，而不是指通过 css 控制画布的宽高

   ```
{
       /** size: [canvas.clientWidth, canvas.clientHeight] */
       size: [400, 300]
   }
   
   //或者
   
   {
       size: {
           width: canvas.clientWidth,
           height: canvas.clientHeight
       }
   }
   ```
   
   对于高清屏，那么可能需要将上述代码中 画布的宽高对应的像素数量 修改为：

   ```diff
   - canvas.clientWidth
   + canvas.clientWidth * window.devicePixelRatio
   
   - canvas.clientHeight
   + canvas.clientHeight * window.devicePiexRatio
   ```



<br>

> 你会发现上述配置项中 format、usage、size 这些都是创建 纹理(GPUTexture) 时需要的配置项。
>
> 从侧面也印证了那个结论：GPUCanvasContext 相当于 WebGPU 中的一个纹理。



<br>

当每次通过 .configure() 修改画布上下文配置后，会立即销毁之前 配置 而产生的纹理。

> 再次提醒：GPUCanvasContext 仅仅相当于 WebGPU 中的一个 纹理。



<br>

**unconfigure()：销毁画布上下文的配置**

```
unconfigure(): undefined;
```

执行该函数后，则会立即销毁此前 画布上下文 产生的任何纹理。



<br>

**getCurrentTexture()：获取当前通过 GPUCanvasContext 即将要合并到文档中的纹理**

```
getCurrentTexture(): GPUTexture;
```

> 在 英文版官方文档中，这句的原话是：Get the GPUTexture that will be composited to the document by the GPUCanvasContext



<br>

以上就是 GPU 画布上下文(GPUCanvasContext) 的全部内容。



<br>

#### 关于WebGPU中可能会发生的一些错误

在 WebGPU  运行的过程中，可能会发生一些意外错误。



<br>

**GPUDevice.lost：GPU丢失错误(不可用)**

```
readonly lost: Promise<GPUDeviceLostInfo>;

interface GPUDeviceLostInfo {
    readonly __brand: "GPUDeviceLostInfo";
    readonly reason: GPUDeviceLostReason | undefined;
    readonly message: string;
}

type GPUDeviceLostReason = "destroyed";
```

我们可以向 GPUDevice 实例的 .lost 添加错误侦听。

```
device.lost.catch((err) => {
    console.log(err)
})
```

> GPU 丢失属于致命性的错误。若系统 GPU 都突然不可用了，那还搞什么 WebGPU。



<br>

**GPUOutOfMemoryError：GPU内存不足错误**

```
interface GPUOutOfMemoryError {
    readonly __brand: "GPUOutOfMemoryError";
}
```



<br>

**GPUValidationError：GPU验证错误**

```
interface GPUValidationError {
    readonly __brand: "GPUValidationError";
    readonly message: string;
}
```



<br>

**错误范围(gpu error scope)**

错误范围用于隔离一组 WebGPU 调用中发生的错误，通常用于调试或使操作更具容错性。



<br>在讲解 GPUDevice 时提到过，GPUDevice 实例 有 3 个和错误相关的方法：

1. pushErrorScope()：在结尾处 添加(push) 一个错误范围

   ```
   pushErrorScope(
       filter: GPUErrorFilter
   ): undefined;
   
   type GPUErrorFilter = "out-of-memory" | "validation";
   ```

2. popErrorScope()：在结尾处 删除并返回 最后一个错误范围

   ```
   popErrorScope(): Promise<GPUError | null>;
   
   type GPUError = GPUOutOfMemoryError | GPUValidationError;
   ```

3. onuncapturederror：当发生错误(GPUError) 但未被任何 错误范围 观察到时

   ```
   onuncapturederror: (( 
       this: GPUDevice,
       ev: GPUUncapturedErrorEvent
   ) => any) | null;
   
   interface GPUUncapturedErrorEvent extends Event {
       readonly __brand: "GPUUncapturedErrorEvent";
       readonly error: GPUError;
   }
   ```



--------- 伟大的分割线 ---------



<br>

到此刻为止，我们终于把 WebGPU 绝大多数模块和类学习一遍了，第一阶段完成了！

尽管第一阶段花费了很多时间，尽管很多类的概念、用法 我们依然不是那么清晰，但是有了这些基础，相信下一阶段会通顺起来的。

**地基打得深，大楼才能盖得高。**



<br>

接下来，我们将开始编写看得见的 WebGPU 示例。

Hello world 我来啦！

