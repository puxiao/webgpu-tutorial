# 23 WebGPU空间体系：MVP矩阵介绍与示例

**WebGPU 中的空间坐标转换体系和其他 3D 引擎类库一样，都是通过 MVP 矩阵来进行转换的。**



<br>

> 下面这段理论知识，实际上并不是 WebGPU 特有的，而是所有 3D 引擎都采用的空间变换体系。
>
> 准确来说，它属于图形学中的基础理论知识点。



<br>

**对于 3D 空间体系而言，通常可以归纳为以下 5 种不同的空间坐标体系：**

1. 局部空间(Local Space)，也可以称呼为 物体空间(Object Space)
2. 世界空间(World Space)
3. 观察空间(View Spac)，也可以称呼为 视觉空间(Eye Space)
4. 裁剪空间(Clip Space)
5. 屏幕空间(Screen Space)

以上就是一个顶点在最终被转化为片段之前所需要经历的 5 个空间状态。



<br>

> 对于一个原始的模型顶点坐标而言，它对应的分别是：局部坐标、世界坐标、观察坐标、裁剪坐标、屏幕坐标



<br>

**不同的 2 个空间坐标体系之间可以通过 1 个矩阵进行变换。**

**上述 5 个空间坐标系中只有前 3 个空间是可以进行编程的，后 2 个空间是由 GPU 自动进行的，所以我们只需要 3 个矩阵即可完成整套坐标体系的转换。**



<br>

**5个空间和 3 个矩阵：**

5 个空间坐标体系和 3 个变换矩阵，他们的依次顺序为：

1. 局部空间坐标 与 世界空间坐标 之间的是 模型矩阵(model matrix)
2. 世界空间坐标 与 视图空间坐标 之间的是 视图矩阵(view matrix)
3. 视图空间坐标 与 裁切空间坐标 之间的是 投影矩阵(projection matrix)
4. 裁切空间坐标 与 屏幕空间坐标 之间是由 GPU 自动完成的，所以它俩中间不再需要可编程的矩阵



<br>

为了加深印象，我们把上面那段话再换一种方式表述一遍：

1. 局部空间坐标 通过 模型矩阵 转变成 世界空间坐标
2. 世界空间坐标 通过 视图矩阵 转变成 视图空间坐标
3. 视图空间坐标 通过 投影矩阵 转变成 裁切空间坐标
4. 裁切空间坐标 会由 GPU 自动转变成 屏幕空间坐标



<br>

**模型矩阵(model matrix)、视图矩阵(view matrix)、投影矩阵(projection matrix) 它们 3 个简称为 MVP 矩阵。**





<br>

> 模型矩阵 实际上可以由 N 个矩阵相乘(叠加)而来，例如在 Three.js 中可以创建多个嵌套的空间
>
> 视图矩阵 决定我们从哪个视角去看 3D 场景，此时使用的是正交投影
>
> 投影矩阵 决定我们以哪种投影方式去最终渲染呈现，例如 透视投影 等



<br>

> 正交投影和透视投影是我们最常见的 2 种投影，除此之外还可以根据某些特定需求实现一些自定义投影，例如 小孔成像 投影。



<br>

**MVP 矩阵背后的共同目标和诉求：在不更改模型原始的顶点坐标信息情况下，通过矩阵转换来获得模型不同的渲染结果。**



<br>

> 为什么要强调 “不更改模型原始的顶点坐标信息” ？
>
> 假设有一个立方体模型：
>
> 1. 你总不能为了放大、旋转、平移模型而每次都去修改模型的原始顶点坐标吧
> 2. 你总不能为了切换模型的不同视角而每次都去修改模型的原始顶点坐标吧
> 3. 你总不能为了查看不同的透视效果而每次都去修改模型的原始顶点坐标吧
>
> 所以，最佳的方案就是 不修改模型的原始顶点坐标信息，只是通过矩阵转换来实现以上任何你想要的渲染效果。



<br>

**什么是视椎？透视投影背后的实现原理是什么？**

> 这部分属于基础的图形学知识点，网上有很多文章讲解。

透视投影的实现原理，简单可以分为以下几个步骤：

1. 我们创建 2 个前后平行、但大小不一、且中间有一定距离的  2 个平面，形成一个视椎

   > 这 2 个平面分别是 近平面(near)、远平面(far)

2. 先将 3D 物体以 正交投影 的方式投影到 近平面中，得到一个画面

3. 然后再根据 3D 物体不同顶点的 z 值，按照某种比例对画面内容元素(三角面)进行 不同比例的缩小，并投影到远平面中

<br>

最终就会根据 z 值的不同将内容元素呈现出大小不同的样子，这就是透视投影。

> 如果 前后 2 个平面的大小相同，那这就是 正交投影。



<br>

> 以上关于视椎的讲解仅为个人的理解，即使讲得不对，那也差不到哪去，大概就是这样的一个意思。



<br>

**图形学和线性代数**

图形学是一门独立的学科，其中一个核心就是线性代数：

1. 向量：点乘，叉乘，归一化...
2. 矩阵：相加、相乘、转置、交换律...
3. 齐次坐标、四元素、欧拉角、万向锁...
4. ....

非常多个知识点，需要花费大量时间和精力去学习。

我们不需要达到手工推导数学公式的程度，但至少需要知道那些知识点背后的含义。

不会线性代数，那么你在学习和使用任何一门 3D 引擎时将无比艰难，因为你无法理解他们究竟在说什么。

学习图形学和线性代码，目前最好的教程是：B 站上 严令琪的 《现代计算机图形学入门》



<br>

关于 MVP 的理论知识就简单讲解到这里，接下来转到实际的代码中。



<br>

首先我们需要在项目中安装一个 NPM 包：gl-matrix

**gl-matrix 介绍**

```
yarn add gl-matrix
```

gl-matrix 是一个 JS 版的 矩阵和向量 类库。

源码仓库：https://github.com/toji/gl-matrix

它的核心作者是 `Brandon Jones`，该作者的身份为：谷歌浏览器 WebGPU/WebXR 开发者

> 他同时也是 WebGPU 标准的核心开发人员之一。

> 他也向 Three.js 提交了和 WebXR 一些相关的 PR。



<br>

> 尽管在 gl-matrix 的官方介绍中提到它是用于 WebGL 使用的，但是线性代数是通用的，所以我们在 WebGPU 中可以放心使用它。



<br>

> Three.js 中自己有一套独立的 数学 包：
>
> https://github.com/mrdoob/three.js/tree/dev/src/math
>
> 它为 Three.js 提供更加丰富的 线性代数 相关对象，但目前我们使用 gl-matrix 已经足够了。



<br>

**gl-matrix 快速使用示例**

gl-matrix 中常用的几种对象：

1. 矩阵：mat2、mat2d、mat3、mat4
2. 向量：vec2、vec3、vec4
3. 四元素：quat、quat2
4. 其他：glMatrix (一些小工具性质的常量和方法)



<br>

假设我们需要使用 3 维向量、和 4 维矩阵，那么首先引入他们：

```
import { vec3, mat4 } from 'gl-matrix'
```



<br>

在 gl-matrix 源码中，他不是通过 class，而是通过 module 形式来组织定义 vec3、mat4 的。

也就是说 vec3、mat4 他们都是函数，而不是类。

因此你不能通过 `new vec3()` 这种形式来创建一个 vec3 实例，而是要通过它们的 .create() 函数来创建。

```
const myVec3 = vec3.create()
const myMatrix4 = mat4.create()
```



<br>

> 这一点和 Three.js 是不同的，在 Three.js 中 Vector3 是一个类，你可以直接 `new Vector3()`



<br>

在上述示例代码中，我们分别得到了实例 myVec3、myMatrix4，接下来就可以调用它们对应的各种方法来进行各种变换计算了。

具体的用法，可以参考他们的官方文档：

1. https://glmatrix.net/docs/module-vec3.html
2. https://glmatrix.net/docs/module-mat4.html



<br>

> 注：由于它所谓的 官方文档 实际上是由 JSDoc 生成的，所以在使用过程中与其查看文档，不如直接查看它的 index.d.ts 更加直接。



<br>

前期铺垫终于讲完，接下来上代码。

我们还是基于之前渲染的 红色三角形 示例，在它上面做一些修改。



<br>

**MVP 矩阵示例：**

我们先定义 MVP 对应的 3 个矩阵，再定义一个最终合并后的 mvp 矩阵：

```
//创建模型矩阵，以下简称 M，我们暂时对 M 什么也不做
const modelMatrix = mat4.create()

//创建视图矩阵，以下简称 V，我们暂时对 V 什么也不做
const viewMatrix = mat4.create()

//创建投影矩阵，以下简称 P，我们暂时对 P 什么也不做
const prejectionMatrix = mat4.create()

//将以上 3 个矩阵相乘，相乘的顺序实际上是 P x V x M，得到最终的 mvp 矩阵
const mvpMatrix = mat4.create()
mat4.multiply(mvpMatrix, projectionMatrix, viewMatrix) //由于 multiply() 每次只能记录 2 个矩阵的相乘，所以我们先计算 P x V，得到 PV
mat4.multiply(mvpMatrix, mvpMatrix, modelMatrix) //再计算 PV x M，这样就得到了 P x V x M 最终相乘结果
```

> `multiply()` 第1个参数用于存储第 2、第 3 个参数相乘的结果



<br>

**我们只是简单示例，暂时将 mvp 矩阵相乘的计算工作放到了 JS CPU 中进行，实际上可以分别把 M、V、P 矩阵传入到顶点着色器中，然后在顶点着色器中，也就是 GPU 中进行相乘计算，其运算性能要比在 CPU 中高很多。**



<br>

特别强调：矩阵相乘不具乘法备交换律，因此矩阵相乘的前后顺序不能变，我们必须严格按照 P, V, M 这个先后顺序进行相乘。



<br>

> 补充：什么是乘法交换律？
>
> 假设我们计算 2 个数字相乘，例如 2 x 3，调换相乘的前后顺序，改为 3 x 2，其最终结果是完全相同的，这种就符合乘法交换律。
>
> 但是对于 2 个矩阵相乘而言，例如 `matrixA x matrixB` 和 `matrixB x matrixA` 的运算结果是不相同的，矩阵相乘不符合乘法交换律。

> 与乘法交换律对应的还有：乘法结合律、乘法分配律



<br>

> 实际上这里还牵扯到一个知识点：矩阵的左乘和右乘
>
> 自己百度搜索吧。



<br>

接下来，我们会将 mvpMatrix 矩阵以资源绑定的形式传递到顶点着色器中。

<br>

```
//创建 mvp 矩阵对应的缓冲区
const matrixBuffer = device.createBuffer({
    size: mvpMatrix.length * 4,
    usage: GPUBufferUsage.UNIFORM | GPUBufferUsage.COPY_DST
})

//将矩阵对应的缓冲区写入到队列中
device.queue.writeBuffer(matrixBuffer, 0, mvpMatrix as Float32Array)

//在绑定组中，我们将矩阵对应的缓冲区设定为 binding:1
const bindGroup = device.createBindGroup({
    layout: pipeline.getBindGroupLayout(0),
    entries: [
        {
            binding: 0,
            resource: {
                buffer: colorBuffer
            }
        },
        {
            binding: 1,
            resource: {
                buffer: matrixBuffer
            }
        }
    ]
})
```



<br>

修改顶点着色器，得到资源绑定缓冲区中 binding 为 1 的缓冲区数据

```
@group(0) @binding(1) var<uniform> mvp:mat4x4<f32>;
@vertex
fn main(@location(0) xyz: vec3<f32>) -> @builtin(position) vec4<f32> {
    return mvp * vec4<f32>(xyz, 1.0);
}
```

> 上述代码中的 `mvp * vec4<f32>(xyz, 1.0)`，相当于将 mvp 矩阵应用到 顶点坐标 xyz 中。



<br>

运行代码，你会发现没有报错，但也没有任何变化。

这是因为我们最初创建的 3 个 M、V、P 矩阵还没有做任何变化，接下来，我们只需要修改这 3 个矩阵，就能看到不同的效果了。



<br>

**修改M、V、P矩阵**

修改之前，我们重复一遍通常情况下，这 3 个矩阵它们所承担的作用：

1. M：模型矩阵，用来对顶点坐标位置进行变换，例如 平移、缩放、旋转
2. V：视图矩阵，用来对观察角度进行变换
3. P：投影矩阵，用来对投影(例如透视投影)进行变换



<br>

**示例之：平移、缩放、旋转**

```
import { mat4, vec3 } from 'gl-matrix'

mat4.translate(modelMatrix, modelMatrix, vec3.fromValues(0.0, 0.0, -10.0)) //我们将三角形所有顶点的 z 值减 10.0
mat4.scale(modelMatrix, modelMatrix, vec3.fromValues(0.5, 0.5, 0.5)) //我们将三角形的 x,y,z 都缩放到 0.5
mat4.rotate(modelMatrix, modelMatrix, Math.PI / 4, vec3.fromValues(0.0, 0.0, 1.0)) //我们将三角形以 (0.0,0.0,1.0) 旋转了 Math.PI/2
```



<br>

> 特别补充：3D 空间以某个轴(向量)旋转遵循的是 右手螺旋法则
>
> 所谓右手螺旋法则是指：用右手握住旋转的轴，大拇指指向该轴的 正 方向，此时四指弯曲的方向即为沿该轴旋转的 正方向。

> 3D 旋转中正方向 是 逆时针方向。



<br>

**示例之：视图变换**

> 也就是镜头视角变换

```
const viewMatrix = mat4.create()
mat4.lookAt(viewMatrix, vec3.fromValues(0.0, 0.0, 0.5), vec3.fromValues(0.0, 0.0, 0.0), vec3.fromValues(0.0, 1.0, 0.0))
```



<br>

**示例之：透视投影**

```
const projectionMatrix = mat4.create()
mat4.perspective(projectionMatrix, Math.PI / 4, canvas.width / canvas.height, 0.1, 100)
```



<br>

由于目前我们示例中创建的是一个 “平面”的纯红色三角形，很难看出其立体透视效果，所以这里就不展示 MVP 矩阵变换后的最终渲染效果了。



<br>

本文主要从理论角度，讲述了 MVP 矩阵变换，理解了 MVP 矩阵后，会对整个 Web 3D 开发有一个质的突破。



<br>

为了更好凸显出 MVP 矩阵变换后的 3D 效果，我们需要先去学习另外一个知识点：颜色插值。

本文到此结束，下一节我们将学习一下 “颜色插值”，这样就可以渲染出颜色丰富多彩的三角形，而不是纯色三角形。

