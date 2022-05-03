# 07 WebGPU基础之纹理(GPUTexture)与纹理视图(GPUTextureView)

**纹理(GPUTexture)用于存储纹理图像数据。**



<br>

假设你之前有一定的 WebGL 基础 或者你会一些 Three.js，那么对于 `纹理` 这个词肯定不会陌生。

> 特别说明：我并没有在原生 WebGL 中使用过纹理，本文关于纹理的理解都是我基于 Three.js 经验。而这些 “纹理” 实际上都是 Three.js 封装过后的，Three.js 背后帮我们做了很多事情的，因此这个 “纹理” 与本文要学习的 GPUTexture(纹理) 并不是同一个概念。



<br>

**纹理的概念：**

纹理就是图像数据，可应用到 3D 对象的表面作为其外观贴图。

例如 大理石纹理贴图、金属纹理贴图、木头纹理贴图等等，这些看似不同风格的纹理贴图其本质是相同的，他们只是图像数据不同而已。

> 对于我们的眼睛而言，我们能看出哪个像大理石，哪个像金属，但是对于计算机而言它们是没有区别的，它们都是一堆(张)图像数据而已。

给一个立方体模型添加大理石风格的贴图，那么渲染后立方体表面就会出现大理石纹理图案。

> 这是句废话

还有一种特殊用途的纹理：深度纹理，它将自己的明暗信息应用到物体表面渲染中。



<br>

**纹理的图像数据来源：**

1. 加载的图片文件

   > 例如 .jpg、.png 等

2. 某段视频中当前播放帧中的画面内容

   > 这实际上还是一张图片

3. 由程序代码生成的某些图像数据

4. 网页中 `<canvas>` 标签中的内容

5. ...



> 纹理的本质就是图，而 图 对应的英文单词是 map
>
> 在数据结构中 图 的英文也是 map



> 补充几个 闫令琪 现代图形学课程上和纹理相关的基础知识点

<br>

**线性插值：**

线性插值是指插值函数为一次多项式的插值方式，其在插值节点上的插值误差为零。

简单来说就是假设有一个三角形，知道这个三角形三个顶点的颜色，然后根据线性插值得到该三角形内任意位置的颜色，使得整个三角形颜色看上去平滑过渡。



<br>

**双线性插值(Bilinear)：**

双线性插值又被称为 双线性内插，所谓双线性就是在两个方向分别进行一次线性插值。



<br>

**锯齿感：小纹理渲染大面会遇到的问题**

双线性插值在图形学中用来计算贴图点非整数情况下如何得到对应的纹理像素的一种算法，换句说就是如何让一张比较小的纹理去渲染一个比较大的面，希望减少像素颗粒感(锯齿)，让渲染结果更加平滑。

> 与之对应的是：
>
> 1. Bicubic(双三次插值)：更加平滑，计算量也更大，它是取周围 16 个像素，第 1 次先以 4 个像素为一组分别做三次插值(是三次插值而非线性插值)，然后再将 4 个结果再做一次三次插值，最终得到一个像素的结果。
> 2. Nearest(就近取值)：四舍五入取整，计算量最小，但渲染结果不平滑，像素感(锯齿)比较重。



<br>

**摩尔纹：大纹理渲染小面会遇到的问题**

假设纹理比较大，而需要渲染的面比较小，是否得到的渲染结果就会比较清晰？事情并不是想象的那个样子。

如果使用点采样模型(Point sampled)最终结果恰恰是 “走样”：无论近处还是远处都会有很强烈的锯齿感。

通常会采用 mipmap。



<br>

**mipmap：**

mipmap 核心思想为：近似的正方形范围查询。

1. 假设有一个纹理贴图宽高均为 256，我们将这个纹理贴图作为第 1 个层级

2. 我们将第 1 级纹理贴图的宽高都缩小一半，得到一个宽高为 128 的纹理贴图作为第 2 个层级

3. 然后再将第 2 级纹理的宽高再次缩小一半，得到一个宽高为 64 的纹理贴图作为第 3 个层级

4. ...

5.  当第 N 个层级时纹理宽高为 1

6. 至此我们的这个纹理贴图 一共有 N 个层级，尽管每个层级的贴图大小不同，但是他们的宽高比却完全相同

7. 假设有一个 3D 物体需要应用这个纹理，那么渲染器会根据实际情况进行匹配：

   当这个被渲染的 3D 物体距离镜头足够近的时候，使用第 1 个层级的纹理贴图。

   当被渲染的 3D 物体距离镜头稍远时，使用第 2 个层级的纹理贴图

   当被渲染的 3D 物体距离镜头更远时，使用第 3 个层级的纹理贴图

   ...

8. 结论：根据实际渲染需要(先计算出渲染目标结果的大小)，来使用不同层级上、不同尺寸的纹理贴图，减少采样工作量，同时避免出现摩尔纹。

9. 补充：计算并存储 N 个层级上所有的数据会占用一定的存储空间(内存)，但是这种策略却可以带来渲染时的性能提升，实际上相当于用空间换时间。

   N 个层级上所有数据 = 1 + ( 1/4 + 1/8 + 1/16 ... ) ≈ 4/3

   也就是说采用 mipmap 这种策略所需额外多存储的纹理数据仅为之前的 1/3。

在计算机视觉 或 图形学中会将这种策略称为：图像金字塔

<img src="https://puxiao.com/webgpu_tutorial/imgs/mipmap.jpg" alt="mipmap" style="zoom:80%;" />



<br>

上面我们简单讲解了一些 3D 渲染中纹理的概念，**请注意上面所讲的纹理和我们接下来要学习的 GPUTexture(纹理) 并不是一回事**。



<br>

**GPUTexture 是比较原始的、用于存储纹理图像数据的类。**

GPUTexture 是通过 GPUDevice 实例的 .createTexture() 方法来创建的。

```
GPUDevice.createTexture(descriptor: GPUTextureDescriptor): GPUTexture
```



<br>

#### GPUTextureDescriptor的配置项

```
interface GPUObjectDescriptorBase {
  label?: string;
}

interface GPUTextureDescriptor extends GPUObjectDescriptorBase {
  size: GPUExtent3DStrict;
  mipLevelCount?: GPUIntegerCoordinate;
  sampleCount?: GPUSize32;
  dimension?: GPUTextureDimension;
  format: GPUTextureFormat;
  usage: GPUTextureUsageFlags;
}
```

> 以上为 GPUTextureDescriptor 在 `@webgpu/types` 中的定义



<br>

**size：纹理的尺寸**

这个 size 的值有 2 种设置方式：字典类型 或 可迭代的类型

无论哪种形式，其具体值必须是大于 0 的整数。



<br>

> 我把 size 翻译成 尺寸 略显不合适，但是换成 数量、大小 也觉得不妥。



<br>

第1种：字典类型的值

```
interface GPUExtent3DDict {
  width: GPUIntegerCoordinate; //宽度
  height?: GPUIntegerCoordinate; //高度，默认值为 1
  depthOrArrayLayers?: GPUIntegerCoordinate; //深度或阵列层数，默认值为 1
}
```

> 纹理的宽(width) 和 高(height) 很容易理解，但是 depthOrArrayLayers 该怎么理解呢？  
>
> 从字面上可以看出 depthOrArrayLayers 是指 “Depth(深度信息)” 或 "ArrayLayers(阵列层数)"。
>
> 深度信息(Depth) 很容易理解，阵列层数(ArrayLayers) 可以简单理解为 “当前纹理是由多少组纹理数据构成的”。通常情况下我们不需要去设置 depthOrArrayLayers 的值，使用默认的 1 就好了。此时 1 表示 “当前纹理是由 1 组纹理数据构成的”。
>
> 但是假设我们想创建一个 立方体(Cube)纹理，立方体由 6 个面组成，我们希望每个面使用不同的纹理，也就是说这个立方体(Cube)纹理是由 6 组数据构成的，那么此时 depthOrArrayLayers 的值就必须设置为 6。然后就是其他场景下 depthOrArrayLayers 设置的一些其他值。

<br>

具体的示例：

```
{
  size: {
      width: 64,
      height: 64,
      depthOrArrayLayers: 1
  }
}
```



<br>

第2种：可枚举类型的值

> 被迭代枚举的属性值必须是正整数

```
Iterable<GPUIntegerCoordinate>
```

在 JS 中符合上述要求的类型有：Array、Map、Set

具体的示例：

```
{
    size: [64,64,1]
}
```

> `size: [64,64,1]` 和 `size:{ width: 64, height: 64, depthOrArrayLayers: 1}` 这 2 种设置 size 的方式都是合法的。



<br>

特别强调：无论上述那种设置方式，示例中其中 width 的值 64 并不是指 “64个像素”，而是指 “64个纹素”。

**纹素(texel)：构成纹理的最小基本单位元素。**  

> 像素的英文单词是 pixel，而纹素的英文单词是 texel



<br>

**format：纹理的数据格式**

当然这里实际上是指 纹理中每个纹素 的数据格式。

format 的值类型为字符串(string)，且必须是以下值中的一个：

```
"r8unorm"、"r8snorm"、"r8uint"、"r8sint"、"r16uint"、"r16sint"、"r16float"、"rg8unorm"、"rg8snorm"、"rg8uint"、"rg8sint"、"r32uint"、"r32sint"、"r32float"、"rg16uint"、"rg16sint"、"rg16float"、"rgba8unorm"、"rgba8unorm-srgb"、"rgba8snorm"、"rgba8uint"、"rgba8sint"、"bgra8unorm"、"bgra8unorm-srgb"、"rgb9e5ufloat"、"rgb10a2unorm"、"rg11b10ufloat"、"rg32uint"、"rg32sint"、"rg32float"、"rgba16uint"、"rgba16sint"、"rgba16float"、"rgba32uint"、"rgba32sint"、"rgba32float"、"stencil8"、"depth16unorm"、"depth24plus"、"depth24plus-stencil8"、"depth32float"、"depth24unorm-stencil8"、"depth32float-stencil8"、"bc1-rgba-unorm"、"bc1-rgba-unorm-srgb"、"bc2-rgba-unorm"、"bc2-rgba-unorm-srgb"、"bc3-rgba-unorm"、"bc3-rgba-unorm-srgb"、"bc4-r-unorm"、"bc4-r-snorm"、"bc5-rg-unorm"、"bc5-rg-snorm"、"bc6h-rgb-ufloat"、"bc6h-rgb-float"、"bc7-rgba-unorm"、"bc7-rgba-unorm-srgb"、"etc2-rgb8unorm"、"etc2-rgb8unorm-srgb"、"etc2-rgb8a1unorm"、"etc2-rgb8a1unorm-srgb"、"etc2-rgba8unorm"、"etc2-rgba8unorm-srgb"、"eac-r11unorm"、"eac-r11snorm"、"eac-rg11unorm"、"eac-rg11snorm"、"astc-4x4-unorm"、"astc-4x4-unorm-srgb"、"astc-5x4-unorm"、"astc-5x4-unorm-srgb"、"astc-5x5-unorm"、"astc-5x5-unorm-srgb"、"astc-6x5-unorm"、"astc-6x5-unorm-srgb"、"astc-6x6-unorm"、"astc-6x6-unorm-srgb"、"astc-8x5-unorm"、"astc-8x5-unorm-srgb"、"astc-8x6-unorm"、"astc-8x6-unorm-srgb"、"astc-8x8-unorm"、"astc-8x8-unorm-srgb"、"astc-10x5-unorm"、"astc-10x5-unorm-srgb"、"astc-10x6-unorm"、"astc-10x6-unorm-srgb"、"astc-10x8-unorm"、"astc-10x8-unorm-srgb"、"astc-10x10-unorm"、"astc-10x10-unorm-srgb"、"astc-12x10-unorm"、"astc-12x10-unorm-srgb"、"astc-12x12-unorm"、"astc-12x12-unorm-srgb"
```

不要被上面这些值吓到，上面那么多数值格式都是由下面几种数据类型组合而成的：

1. rgba：带透明度的RGB颜色值

2. bgra：就是将 RGB 颜色值顺序反过来，透明度 a 的顺序不变

   > 一般来说我们习惯的颜色顺序是 rgb，但是对于计算机而言 bgr 这个顺序更加符合它的读取顺序。
   >
   > 颜色 rgb 中的每一个值都是十进制的数值。

3. srgb：开头的 s 是英文单词 standard(标准) 的缩写

   > srgb 是微软联合爱普生、惠普等多家厂商共同制定出的一套颜色标准，无论是在显示器、数码图像采集、打印、扫描、投影等设备中共用同一套颜色标准。

4. 数值类型：uint(无符号整数)、sint(有符号整数)、float(浮点数)、unorm(归一化后的无符号整数)、snorm(归一化后的有符号整数)

   > unorm 是单词 unsigned normalized(已归一化) 的简写
   >
   > snorm 是单词 signed normalized 的简写

   > 下面关于纹理中 unorm 和 snorm 的解释，来源于 udumbara007 的一篇文章：https://blog.csdn.net/udumbara007/article/details/88343238
   >
   > unorm 表示归一化处理的无符号整数，此种格式数据在资源中被解释为无符号整数，在 shader 中解释为 (0.0 - 1.0) 之间的浮点数。以 2 位无符号整数为例，00,01,10,11 (即0,1,2,3) 分别对应的浮点数为 0.0，1/3、2/3、1。
   >
   > snorm 表示归一化处理的有符号整数，这种格式数据在资源中被解释为有符号整数，在 shader 中则被解释为 (-1.0 - 1.0) 之间的浮点数。

5. 纹理压缩：etc2、eac、astc、bc1、bc2、bc3、bc6h、bc7 ... 这些都是不同框架下所支持的纹理压缩格式。

   > etc 是单词 Ericsson Texture Compression 的简写
   >
   > astc 是单词 Adaptive Scalable Texture Compression 的简写 
   >
   > bc 是单词 Block Compression (块压缩) 的简写

6. 深度与阴影：depth(深度)、stencil(模板阴影)



<br>

**纹理数据格式归纳：**

纹理格式的名称指定了 顺序、位数 和 数据类型。

| 纹理格式                     | 包含纹理                                                     |
| ---------------------------- | ------------------------------------------------------------ |
| 8-bit 格式                   | "r8unorm", "r8snorm", "r8uint", "r8sint"                     |
| 16-bit 格式                  | "r16uint", "r16sint", "r16float", "rg8unorm", "rg8snorm", "rg8uint", "rg8sint" |
| 32-bit 格式                  | "r32uint", "r32sint", "r32float", "rg16uint", "rg16sint", "rg16float", <br />"rgba8unorm", "rgba8unorm-srgb", "rgba8snorm", "rgba8uint", "rgba8sint", <br />"bgra8unorm", "bgra8unorm-srgb" |
| Packed(压缩) 32-bit formats  | "rgb9e5ufloat", "rgb10a2unorm", "rg11b10ufloat"              |
| 64-bit 格式                  | "rg32uint", "rg32sint", "rg32float", "rgba16uint", "rgba16sint", "rgba16float" |
| 128-bit 格式                 | "rgba32uint", "rgba32sint", "rgba32float"                    |
| 深度/模板格式                | "stencil8", "depth16unorm", "depth24plus", "depth24plus-stencil8", "depth32float" |
| "depth24unorm-stencil8" 特性 | "depth24unorm-stencil8"                                      |
| "depth32float-stencil8" 特性 | "depth32float-stencil8"                                      |
| BC 压缩格式                  | "bc1-rgba-unorm", "bc1-rgba-unorm-srgb", "bc2-rgba-unorm", "bc2-rgba-unorm-srgb", "bc3-rgba-unorm", "bc3-rgba-unorm-srgb", "bc4-r-unorm", "bc4-r-snorm", "bc5-rg-unorm", "bc5-rg-snorm", "bc6h-rgb-ufloat", "bc6h-rgb-float", "bc7-rgba-unorm", "bc7-rgba-unorm-srgb" |
| ETC2 压缩格式                | "etc2-rgb8unorm", "etc2-rgb8unorm-srgb", "etc2-rgb8a1unorm", "etc2-rgb8a1unorm-srgb", "etc2-rgba8unorm", "etc2-rgba8unorm-srgb", "eac-r11unorm",     "eac-r11snorm", "eac-rg11unorm", "eac-rg11snorm" |
| ASTC 压缩格式                | "astc-4x4-unorm", "astc-4x4-unorm-srgb", "astc-5x4-unorm", "astc-5x4-unorm-srgb", "astc-5x5-unorm", "astc-5x5-unorm-srgb", "astc-6x5-unorm", "astc-6x5-unorm-srgb", "astc-6x6-unorm", "astc-6x6-unorm-srgb", "astc-8x5-unorm", "astc-8x5-unorm-srgb", "astc-8x6-unorm", "astc-8x6-unorm-srgb", "astc-8x8-unorm",     "astc-8x8-unorm-srgb", "astc-10x5-unorm", "astc-10x5-unorm-srgb", "astc-10x6-unorm", "astc-10x6-unorm-srgb", "astc-10x8-unorm", "astc-10x8-unorm-srgb",     "astc-10x10-unorm", "astc-10x10-unorm-srgb", "astc-12x10-unorm", "astc-12x10-unorm-srgb", "astc-12x12-unorm", "astc-12x12-unorm-srgb" |

> "depth24plus" 和 "depth24plus-stencil8" 深度格式可以实现 24 位无符号归一化值("depth24unorm") 或 32 为 IEEE 754 浮点值("depth32float")。

> BC  压缩需要设备支持 "texture-compression-bc"，是微软主要支持的纹理压缩格式。
>
> ETC 压缩需要设备支持 “texture-compression-etc2”，安卓系统 CPU 渲染的只支持 etc1，GPU 渲染的才支持 etc2。
>
> ASTC 压缩需要设备支持 “texture-compression-astc”，ASTC 是苹果系统支持的纹理压缩格式。



<br>

> 本文中关于纹理的言论仅为个人目前的一些粗浅认知，若有错误请谅解。
>
> 另外说一句：不要被劝退。



<br>

**usage：纹理的使用标记**

usage 值的类型为 number，且必须是以下值中的一个：

| 字面值                            | 实际值 | 含义         |
| --------------------------------- | ------ | ------------ |
| GPUTextureUsage.COPY_SRC          | 0x01   | 作为拷贝来源 |
| GPUTextureUsage.COPY_DST          | 0x02   | 作为拷贝目标 |
| GPUTextureUsage.TEXTURE_BINDING   | 0x04   | 作为纹理绑定 |
| GPUTextureUsage.STORAGE_BINDING   | 0x08   | 作为存储绑定 |
| GPUTextureUsage.RENDER_ATTACHMENT | 0x10   | 作为渲染附件 |

> RENDER_ATTACHMENT 我暂时把它翻译成 “渲染附件”，它实际的意思是：颜色渲染 或 深度模板渲染



<br>

上面 3 个配置项 size、format、usage 为必填项，下面介绍 3 个非必填项。



<br>

**mipLevelCount：纹理的级数**

mipLevelCount 的值类型为 正整数，默认值为 1。

> mip 是单词 multum in parvo 的简写，英文直译是 “细小多毛”，在图形学中是指 “多级纹理映射”。
>
> 我们可以知道 mipLevel 暗含的意思为：不同大小、不同层级。
>
> 而 mipLevelCount 的意思是：纹理不同大小、不同层级的数量，也就是 “纹理的级数”。



<br>

**sampleCount：纹理的采样次数**

sampleCount 的值必须是 1 或 4，默认为 1。

> 你可以简单地把纹理采样理解为：渲染器为了让渲染结果更加平滑、为了抗锯齿感而进行的某种处理手段。



<br>

**dimension：纹理的维度**

dimension 的值必须是以下中的一个：

```
'1d'、'2d'、'3d'
```

默认值为 '2d'。



<br>

创建一个 GPUTexture 的简单示例：

```
const texture = device.createTexture({
    size: [64,64],
    format: 'bgra8unorm',
    usage: GPUTextureUsage.COPY_SRC | GPUTextureUsage.COPY_DST
})
```



<br>

#### GPUTexture的属性和方法

GPUTexture 只有一个只读属性 label。

GPUTexture 有 2 个方法：

1. destroy()：销毁自身
2. createView()：创建纹理视图(GPUTextureView)



<br>

#### 纹理视图(GPUTextureView)

通过调用 GPUTexture 实例的 .createView() 方法可以得到一个 GPUTextureView(纹理视图)。

**通过纹理视图(GPUTextureView)可以访问纹理资源(GPUTexture)。**

1. 纹理(GPUTexture)用于在内存(显存)中存储最原始的纹理资源数据

2. 只有通过 纹理视图(GPUTextureView) 才可以访问 GPUTexture 中的数据资源

3. 纹理视图甚至可以将 GPUTexture 中的 纹理数据 A 伪装成 纹理数据 B，或者将 纹理格式 A 伪装成 纹理格式 B。

   > 不保证以上观点一定正确



<br>

**createView()的可选参数：GPUTextureViewDescriptor**

在调用 .createView() 方法时可以不添加任何参数，也可以添加 GPUTextureViewDescriptor 类型的参数。

```
interface GPUTextureViewDescriptor
  extends GPUObjectDescriptorBase {
  format?: GPUTextureFormat;
  dimension?: GPUTextureViewDimension;
  aspect?: GPUTextureAspect;
  baseMipLevel?: GPUIntegerCoordinate;
  mipLevelCount?: GPUIntegerCoordinate;
  baseArrayLayer?: GPUIntegerCoordinate;
  arrayLayerCount?: GPUIntegerCoordinate;
}
```

GPUTextureViewDescriptor 中的各项都是可选参数，他们具体的含义为：

1. format：纹理格式

   > 若不设置则与 GPUTexture 中的 format 一致

2. dimension：纹理的维度，但是请注意这里的可选值与 GPUTexture 不太一样

   > GPUTexture 的 dimension 可选值为 '1d'、'2d'、'3d'
   >
   > 但是这里的 dimension 可选值为 '1d(一维图像)'、'2d(二维图像)'、'2d-array(二维图像数组)'、'cube(立方体贴图)'、'cube-array(包含 n 个立方体贴图的阵列)'、'3d(三维图像)'

3. aspect：用到该纹理的哪些方面，可选值为 'all(全部)'、'stencil-only(仅模板)'、'depth-only(仅深度)'

   > 若不设置则默认值为 'all'

4. baseMipLevel：指定该纹理的多级纹理的基础等级

   > 若不设置则默认值为 0

5. mipLevelCount：纹理的级数

   > 若不设置则与 GPUTexture 中的 mipLevelCount 一致

6. baseArrayLayer：基础的阵列图层级

   > 若不设置则默认值为 0

7. arrayLayerCount：阵列图的数量

   > 额~，此处翻译可能不太正确



<br>

一个简单的示例代码：

```
gpuTexture.createView({
    format: 'bgra8unorm',
    dimension: '2d',
    aspect: 'all',
    baseMipLevel: 0,
    mipLevelCount: 1,
    baseArrayLayer: 0,
    arrayLayerCount: 1
})
```



<br>

#### 外部纹理(GPUExternalTexture)

GPUExternalTexture 是包装外部视频对象的可采样纹理。

GPUExternalTexture 对象的内容可能不会改变，无论是从 WebGPU 内部(它仅可采样) 还是从 WebGPU 外部(例如由于视频帧推进)。



<br>

和 GPUExternalTexture 有关联的一个类是 GPUBindGroupLayout(资源绑定)，但是我们还没学到这个类，所以不做过多讲述。



<br>

**导入外部纹理：**

通过 GPUDevice 的 .importExternalTexture() 方法可以将外部视频导入并创建 GPUExternalTexture 实例。

在调用 `importExternalTexture()` 方法时需要传递一个参数，这个参数的类型为 GPUExternalTextureDescriptor。

```
importExternalTexture(
  descriptor: GPUExternalTextureDescriptor
): GPUExternalTexture;

interface GPUExternalTextureDescriptor
  extends GPUObjectDescriptorBase {
  source: HTMLVideoElement;
  colorSpace?: GPUPredefinedColorSpace;
}
```



<br>

**source**：外部视频对应的 HTMLVideoElement 元素

**colorSpace**：预定义颜色空间，是一个可选参数，默认值为 undefined，目前该值仅可以被设置为 "srgb"。



<br>

在 React + TypeScript 中一个简单示例：

```
const videoRef = useRef<HTMLVideoElement>(null)

//伪代码
const externalTexture = device?.importExternalTexture({
  source: videoRef.current,
  colorSpace: 'srgb'
})

return (
  <video ref={videoRef} src='./assets/video/hello.mp4' width="320" height="240" />
)
```



<br>

**采样外部纹理：**

外部纹理(external textures)在 WGSL 中用 texture_external 表示，可以使用 textureLoad 和 textureSampleLevel。关于 采样 本文不做过多介绍。

> 注：上面这句话中的 “外部纹理” 是一种统称，并不是指 GPUExternalTexture。



<br>

由于本人对 纹理 的理解并不深，尤其是涉及到纹理的底层知识，我是一边网上查资料一边硬着头皮写下本文的。

接下来要学习的内容对于普通前端开发者而言会越来越底层、越来越难以理解。

没有什么好说的，硬着头皮学吧。



<br>

本文到此结束，接下来我们将学习 采样器(GPUSampler)。
