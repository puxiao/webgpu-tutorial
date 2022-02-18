# 08 WebGPU基础之采样器(GPUSampler)

**采样器(GPUSampler)包含纹理寻址和纹理过滤，通过它们来决定以哪种缩放形式将纹理应用到模型表面渲染中。**



<br>

简单来说：

1. GPUTexture：纹理，以某种格式存储具体的纹理数据
2. GPUTextureView：纹理视图，可访问纹理数据(GPUTexture)
3. GPUSampler：采样器，决定以哪种缩放形式来得到具体的纹素(texel)



<br>

在学习 GPUSampler 之前，我们先学习一下图形学中 采样器 的概念。

#### 采样器(Sampler)

采样器主要包含：纹理UV寻址、纹理过滤(分类与方式)

采样器可用于顶点着色器和片元着色器。

> 顶点着色器和片元着色器属于图形学中基础的知识点，如果你不理解可自己查阅相关资料。
>
> 用最简单的话来说(或许不精准)，顶点着色器负责渲染模型顶点的，而片元着色器负责渲染模型 面 。
>
> 模型的顶点位置和其他信息(例如颜色、透明度、法线等)决定了 “模型的面” 的最终的样子。



<br>

**纹理UV寻址：**

1. wrap：重复模式，当 UV 坐标超出 0-1 范围后，回到纹理的起始位置并重新开始计算
2. mirror：镜像模式，当 UV 坐标超出 0-1 范围后，沿纹理结束位置反方向回到起始位置
3. clamp：钳位模式，当 UV 坐标超出 0-1 范围后，将结束位置的最后一个纹素作为剩余的纹素
4. border：边框模式，当 UV 坐标超出 0-1 范围后，将某个固定颜色作为剩余的纹素

![](https://puxiao.com/webgpu_tutorial/imgs/sampler_uv.jpg)



<br>

**纹理过滤分类：**

1. minFilter：缩小过滤器，当样本足迹小于或等于 1 纹素时的采样方式

2. magFilter：放大过滤器，当样本足迹大于 1 纹素时的采样方式

3. mipmap：图层金字塔过滤器，指定在两个 mipmap 级别之间进行的采样方式

   > 在图形学相关文章中，一般情况下是不会将 mipmap 进行中文翻译的，这里使用与 mipmap 意思相近的另外一种称呼 “图形金字塔”。



<br>

**纹理过滤方式：**

1. nearest：邻近取值
2. linear：线性插值
3. bilinear：双线性插值
4. bicubic：双三次插值

![](https://puxiao.com/webgpu_tutorial/imgs/sampler_filter.jpg)



<br>

**PhotoShop中的采样器**

> 我在转行前端开发之前，做过 7 年的网页设计。

在 PS 中将一个高宽均为 100 像素的图片 拉伸到 200像素，可以看到也遵循相同的 “采样” 选项。

![](https://puxiao.com/webgpu_tutorial/imgs/sampler_filter_ps.jpg)



> 刚才我们使用的是 "邻近取值"、“双线性插值”、“双三次插值”，而 PS 中使用的是 “邻近”、“两次线性”、“两次立方”。
>
> 他们只是用词不同，表达的意思完全相同。



<br>

接下来开始转入 GPUSampler 的学习中。

#### GPUSampler：采样器

通过 GPUDevice 的 .createSampler() 方法可以创建 GPUSampler 实例。

```
createSampler( descriptor?: GPUSamplerDescriptor): GPUSampler;

interface GPUSamplerDescriptor
  extends GPUObjectDescriptorBase {
  addressModeU?: GPUAddressMode;
  addressModeV?: GPUAddressMode;
  addressModeW?: GPUAddressMode;
  magFilter?: GPUFilterMode;
  minFilter?: GPUFilterMode;
  mipmapFilter?: GPUFilterMode;
  lodMinClamp?: number;
  lodMaxClamp?: number;
  compare?: GPUCompareFunction;
  maxAnisotropy?: number;
}
```



<br>

下面我们对 .createSampler() 方法所对应参数的配置项逐一讲解。

以下参数均为可选参数。

| 配置项        | 值类型                                                       | 对应含义                                                     |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| label         | string                                                       | 自定义标记                                                   |
| addressModeU  | "clamp-to-edge"(默认值) \| "repeat" \| "mirror-repeat"       | 纹理宽度                                                     |
| addressModeV  | "clamp-to-edge"(默认值) \| "repeat" \| "mirror-repeat"       | 纹理高度                                                     |
| addressModeW  | "clamp-to-edge"(默认值) \| "repeat" \| "mirror-repeat"       | 纹理深度                                                     |
| magFilter     | "nearest"(默认值) \| "linear"                                | 样本足迹小于等于 1 纹素时的采样行为                          |
| minFilter     | "nearest"(默认值) \| "linear"                                | 样本足迹大于 1 纹素时的采样行为                              |
| mipmapFilter  | "nearest"(默认值) \| "linear"                                | 在两个 minmap 级别之间进行采样的行为                         |
| lodMinClamp   | number，默认值为 0                                           | 规定使用最小的细节级别                                       |
| lodMaxClamp   | number，默认值为 32                                          | 规定使用最大的细节级别                                       |
| compare       | "never" \| "less" \| "equal" \| "less-equal" \| "greater" <br />\| "not-equal" \| "greater-equal" \| "always" | 指定采样器的比较条件依据<br />假设有输入值的话，会将输入值与采样纹理值进行比较，并将比较结果用于纹理过滤操作中 |
| maxAnisotropy | number，默认值为 1                                           | 最大 各向异性过滤值<br />取值范围为 1 - 16                   |

GPUAddressMode：

1. "clamp-to-edge"：纹理坐标被限定在 0.0 - 1.0 之间，含 0.0 与 1.0
2. "repeat"：重复
3.  "mirror-repeat"：镜像重复

GPUFilterMode：

1. "nearest"：邻近取值，也就是使用最接近纹理坐标的纹素的值
2. "linear"：线性插值，也就是选择两个纹素并返回他们之间的线性插值

GPUCompareFunction：

> 比较的双方分别是：提供的值 与 纹理采样的值，使用 0.0f 表示通过，使用 1.0f 表示失败(未通过)

1. "never"：永远认定为 不通过
2. "less"：如果提供的值 小于 采样值，则认定为通过
3. "equal"：如果提供的值 等于 采样值，则认定为通过
4. "less-equal"：如果提供的值 小于等于 采样值，则认定为通过
5. "greater"：如果提供的值 大于 采样值，则认定为通过
6. "not-equal"：如果提供的值 不等于 采样值，则认定为通过
7. "greater-equal"：如果提供的值 大于 采样值，则认定为通过
8. "always"：永远认定为 通过



<br>

**名词解释：最大各向异性过滤值(maxAnisotropy)**

> 关于 maxAnisotropy 的解释来源于 fangcun 关于 Vulkan 的文章：  
> https://zhuanlan.zhihu.com/p/58628459

maxAnisotropy 用来限定计算最终颜色所使用的样本个数。

1. maxAnisotropy 的值越小 采样越快，但采样结果质量较低
2. maxAnisotropy 的值越大 采样越慢，但采样结果质量较高

目前还没有图形硬件能够使用超过 16 个样本的，所以 maxAnisotropy 的值被限定在 1 - 16 。



<br>

**名词解释：比较采样器**

在一些 OpenGL 文章中会将这种情况描述为：**采样器的比较模式**

简单来说就是这个采样器的用途是将采样值与输入值按照某种条件规则进行比较的。



<br>

#### createSampler() 创建 GPUSampler 成功的前提条件：

1. GPUDiver 实例是有效的
2. .lodMinClamp 的值需大于等于 0
3. .lodMaxClamp 的值需大于等于 .lodMinClamp 的值
4. .maxAnisotropy 的值需大约等于 1
5. 当 .maxAnisotropy 的值大于 1 时 .magFilter、.minFilter、mipmapFilter 的值必须等于 "linear"



<br>

#### GPUSampler 的属性和方法

目前 GPUSampler 实例仅有一个只读属性 .label。

**特别说明：**

在 https://www.w3.org/TR/webgpu/#sampler-interface 中提到 GPUSampler 有以下 3 个内部插槽。

1. descriptor：只读属性，即 createSampler() 方法中的参数
2. isComparison：值类型为 boolean，表示是否被当做 `比较采样器`
3. isFiltering：值类型为 boolean，表示是否对纹理的多个样本进行加权



<br>

**名词解释：内部插槽**

在目前的 WebGPU 草案官方文档中提到的 “内部插槽” 并不是指公开的属性或方法，而是指内部隐藏的属性或方法。这些 `内部插槽` 是给 WebGPU 规范制定者使用的，并不是给我们使用的。



<br>

关于采样器(GPUSampler)的讲解到此为止。

接下来，我们将学习 资源绑定(Resource Binding)。

