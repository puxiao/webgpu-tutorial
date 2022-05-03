# 04 WebGPU基础之显卡适配器(GPUAdapter)与显卡设备(GPUDevice)

**显卡适配器(GPUAdapter) 与 显卡设备(GPUDevice) 是 WebGPU 最为基础的 2 个类。**

<br>

> 本文直接将 “GPU” 笼统得翻译为 “显卡”，实际上 GPU (graphics processing unit) 更加精准的翻译应该是 “图形处理器”。
>
> adapter： [əˈdæptər] ，直译为 适配器
>
> device： [dɪˈvaɪs]，直译为 装置、设备

<br>

> 如果你不接受将 GPUAdapter 翻译为 “显卡适配器”，那你可以称呼其为 “GPU适配器”，将 GPUDevice 称呼为 “GPU设备”。

> 我看有人将 device 翻译为 “驱动”，即将 GPUDevice 翻译为 “显卡驱动” 或 “GPU驱动”，我认为不是那么合适，本文一概翻译为 “设备”。



<br>

**特别强调：**

本文的知识点来源于 https://www.w3.org/TR/webgpu/ 。

1. 由于目前 WebGPU 的标准并未最终确立，所以本文以及后续文章中的知识点或许会发生变动。
2. 由于个人水平有限，对于某些概念的理解可能会存在偏差，本文不足之处，还请担待。



<br>

#### 基础概念：GPUAdapter、GPUDevice

我们先看一张图，来梳理 GPUAdapter、GPUDevice、浏览器 之间的关系。

![adapter_device](https://raw.githubusercontent.com/puxiao/webgpu-tutorial/main/imgs/adapter_device.jpg)

他们的逻辑关系为：

**假设浏览器支持 WebGPU，那么浏览器中的 JS 主线程 或 Web Worker 可以通过 GPUAdapter(显卡适配器) 来获取当前系统中的 GPUDevice(显卡设备)。**



<br>

**细节补充：**

操作系统分为以下几种情况：

1. Windows
2. Linux
3. Mac OS
4. Android
5. ...

<br>

同一操作系统还可以分为以下几种情况：

1. PC 台式电脑

2. 笔记本电脑

   > PC 台式电脑 和 笔记本电脑 一个很重要的差别地方在于：笔记本电脑需要考虑 电源，所以 WebGPU 在设计之初就要考虑 耗电量 这个关键因素。

<br>

系统上的显卡设备存在以下几种情况：

1. 没有显卡 或 显卡异常不可用
2. 有集成显卡
3. 有独立显卡
4. 既有集成显卡，又有独立显卡
5. 拥有多个类型的显卡
6. ...

结论：**WebGPU 不光要考虑不同的操作系统，还要考虑系统上不同的显卡设备，因此 WebGPU 需要创建一个 GPUAdapter(显卡适配器) 来统一负责获取系统中的显卡设备(GPUDevice)。**



<br>

**你以为本文提到的 显卡设备(GPUDevice) 就真的是系统中的显卡设备吗？**

不是的。

1. 系统上的显卡是由不同厂商生产的。

2. 即使同一显卡在不同的操作系统上又是由不同的底层图形 API 驱动的，WebGPU 所支持的 3 个底层图形架构分别是：DirectX 12(即 D3D12)、Metal、Vulkan。

   > DirectX 12 属于 微软
   >
   > Metal 属于 苹果
   >
   > Vulkan 属于 Khronos Group

结论：**为了适配不同的显卡设备以及系统底层图形架构，因此 WebGPU 需要创建 GPUDevice(显卡设备) 来统一负责 “系统中的显卡设备”。**



<br>

> GPUDevice 在通过 GPUQueue 来负责发送执行命令，关于 GPUQueue 我们会以后再讲解。



<br>

**抹平差异是所有框架的基本原则。**

就如同京东凹凸实验室开源的 Taro ，可以抹平市面上所有常见小程序之间的差异。



<br>

小总结：

1. GPUAdapter 显卡适配器：用于抹平和获取不同类型的 GPUDevice。
2. GPUDevice 显卡设备：用于抹平不同底层图形框架下的显卡设备，提供发送执行 GPU 渲染或计算命令的能力。



<br>

通过以上的情况梳理，我相信你已经对 浏览器、GPUAdapter、GPUDevice 有了基础的认知。



<br>

#### 获取方法：requestAdapter()、requestDevice()

使用 `navigator.gpu.requestAdapter()` 可以异步获取得到一个 GPUAdapter 实例。

使用 GPUAdapter 实例的 `requestDevice()` 可以异步获取得到一个 GPUDevice 实例。

代码示例：

```
if (navigator.gpu) {
    navigator.gpu.requestAdapter().then(adapter => {
        if (adapter) {
            adapter.requestDevice().then(device => {
                if (device) {
                    console.log('GPU is available')
                }
            })
        }
    })
}
```

上面代码可以简化为：

```
const adapter = await navigator.gpu.requestAdapter()
const device = await adapter.requestDevice()
```



<br>

下面将具体讲解一下 GPUAdapter 和 GPUDevice 的属性和方法。

但是请注意，下面讲解的属性和方法采用的是 `@webgpu/types` 0.1.9 版本中定义的 WebGPU 属性和方法。

这里面和 https://www.w3.org/TR/webgpu/ 会有细微出入，但是我们不管，我们就以 `@webgpu/types` 中定义的为准。

> @webgpu/types 也是由 WebGPU 核心团队成员维护的，所以我们就完全相信 @webgpu/types。



<br>

#### GPUAdapter的属性和方法

| 属性名            | 读写 | 值类型               | 对应含义                                     |
| ----------------- | ---- | -------------------- | -------------------------------------------- |
| __brand           | 只读 | "GPUAdapter"         | 该值为固定的字符串 "GPUAdapter"              |
| name              | 只读 | string               | 适配器的名称，默认得到的 name 值为 "default" |
| features          | 只读 | GPUSupportedFeatures | 支持的功能列表                               |
| limits            | 只读 | GPUSupportedLimits   | 受限的功能列表                               |
| isFallbackAdapter | 只读 | boolean              | 是否是 "备用适配器"                          |

> 曾经还有一个属性 isSoftware，不过这个属性已经被废弃，我们也无需知道它了。



<br>

| 方法            | 对应作用                    |
| --------------- | --------------------------- |
| requestDevice() | 异步返回一个 GPUDevice 实例 |



<br>

**requestDevice() 参数详情：**

我们可以通过添加参数，告诉适配器我们希望得到什么样的 GPUDevice(显卡设备)。

1. 不使用参数 或 参数为 undefined

   ```
   adapter.requestDevice()
   ```

   此时 显卡适配器( adapter )会返回一个默认的 显卡设备实例( GPUDevice )。

2. 使用参数 `descriptor?: GPUDeviceDescriptor`

   ```
   //返回一个性能相对不高，但是耗电量较低的显卡设备，通常是指 集成显卡
   adapter.requestDevice({ powerPreference: 'low-power' })
   ```

   ```
   //返回一个性能相对较高，同时耗电量也高的显卡设备，通常是指 独立显卡
   adapter.requestDevice({ powerPreference: 'high-performance' })
   ```

   请注意，尽管我们可以这样添加参数，以希望得到不同的显卡设备，但是这个前提是我们的系统中本身就包含不同的显卡设备。

   假设我的电脑系统中只有集成显卡，根本没有独立显卡，那么此时即使我们添加有参数 `{ powerPreference: 'high-performance' }`，但是 GPUAdapter(显卡适配器) 依然会返回的是 集成显卡。



<br>

**什么时候使用不同的参数？**

假设我们的网页或应用此时并不需要特别高的渲染或通用计算，那么 WebGPU 官方是推荐和鼓励我们采用 `powerPreference: 'low-power'` 这个参数，这样有可能降低系统的耗电量，尤其是针对笔记本电脑。



<br>

#### GPUDevice的属性和方法

| 属性名            | 读写 | 值类型                                                       | 对应含义                                                     |
| ----------------- | ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| __brand           | 只读 | "GPUDevice"                                                  | 该值为固定的字符串 "GPUDevice"                               |
| features          | 只读 | GPUSupportedFeatures                                         | 支持的功能列表                                               |
| limits            | 只读 | GPUSupportedLimits                                           | 受限的功能列表                                               |
| queue             | 只读 | GPUQueue                                                     | 该显卡设备对应的命令队列                                     |
| lost              | 只读 | Promise<GPUDeviceLostInfo>                                   | 显卡设备丢失相关信息，包含 reason(丢失原因)、message(相关信息) |
| onuncapturederror | 读写 | ((this: GPUDevice,ev: GPUUncapturedErrorEvent) => any) \| null | 未捕获错误对应的处理函数                                     |



<br>

| 方法                                                         | 对应作用                               |
| ------------------------------------------------------------ | -------------------------------------- |
| destroy()                                                    | 销毁当前对象                           |
| createBuffer(descriptor: GPUBufferDescriptor): GPUBuffer     | 创建GPUBuffer缓冲区实例                |
| createTexture(descriptor: GPUTextureDescriptor): GPUTexture  | 创建GPUTexture纹理实例                 |
| createSampler(descriptor?: GPUSamplerDescriptor): GPUSampler | 创建GPUSampler采样器实例               |
| importExternalTexture(descriptor: GPUExternalTextureDescriptor): GPUExternalTexture | 导入外部纹理                           |
| createBindGroupLayout(descriptor: GPUBindGroupLayoutDescriptor): GPUBindGroupLayout | 创建GPUBindGroupLayout资源绑定实例     |
| createPipelineLayout(descriptor: GPUPipelineLayoutDescriptor): GPUPipelineLayout | 创建GPUPipelineLayout管线着色器映射    |
| createBindGroup(descriptor: GPUBindGroupDescriptor): GPUBindGroup | 创建GPUBindGroup要绑定的资源组         |
| createShaderModule(descriptor: GPUShaderModuleDescriptor): GPUShaderModule | 创建GPUShaderModule着色器模块对象      |
| createComputePipeline(descriptor: GPUComputePipelineDescriptor): GPUComputePipeline | 创建GPUComputePipeline计算管线         |
| createRenderPipeline(descriptor: GPURenderPipelineDescriptor): GPURenderPipeline | 创建GPURenderPipeline渲染管线          |
| createComputePipelineAsync(descriptor: GPUComputePipelineDescriptor): Promise<GPUComputePipeline> | 异步创建创建GPUComputePipeline计算管线 |
| createRenderPipelineAsync(descriptor: GPURenderPipelineDescriptor): Promise<GPURenderPipeline> | 异步创建GPURenderPipeline渲染管线      |
| createCommandEncoder(descriptor?: GPUCommandEncoderDescriptor): GPUCommandEncoder | 创建GPUCommandEncoder命令编码          |
| createRenderBundleEncoder(descriptor: GPURenderBundleEncoderDescriptor): GPURenderBundleEncoder | 创建GPURenderBundleEncoder渲染束编码   |
| createQuerySet(descriptor: GPUQuerySetDescriptor): GPUQuerySet | 创建GPUQuerySet查询结果集              |
| pushErrorScope(filter: GPUErrorFilter): undefined            | 向堆栈添加一个错误                     |
| popErrorScope(): Promise<GPUError \| null>                   | 从堆栈删除一个错误                     |

> 以上很多方法涉及的对象还没有具体学习，所以个别翻译和解释可能不准确。



<br>

从上面 GPUDevice 的众多方法可以看出 GPUDevice 才是整个 WebGPU 最为核心的类。

通过 GPUDevice 可以延展出整个 WebGPU 的架构体系。

在后续的文章中，我们会逐一学习上面方法中涉及到的各个类，例如 GPUQueue、GPUBuffer、GPUTexture、GPUSampler...

本文到此结束。