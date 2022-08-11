# 19 WebGPU示例之顶点缓冲区传递多个参数

**本文讲解一下顶点缓冲区如何传递多个参数到顶点渲染阶段。**



<br>

> 本文实际上是对上一篇文章 “WebGPU示例之顶点缓冲区(VertexBuffer)” 的补充说明。



<br>

在开始学习本节课之前，我们先说一下最近 WebGPU API 中一些变更。

> 有一部分变化我们在上一节 “顶点缓冲区” 中已经讲过了，下面是在那个基础上的进一步补充。

**变化1：无需显式配置画布上下文的 size 属性值**

```diff
const canvsConfig: GPUCanvasConfiguration = {
    device,
    format,
-    size: [canvas.clientWidth, canvas.clientHeight],
    compositingAlphaMode: 'opaque'
}
context.configure(canvsConfig)
```

在初始化配置 GPU画布上下文(GPUCanvasContext) 时，无需再显式添加 size 属性。

默认情况下 WebGPU 会直接使用 `<canvas>` 标签上添加的 width 、height 属性值。

> 同理，当后续需要更改画布上下文尺寸时，直接使用 `canvas.width = xxx` 、`canvas.height = xxx` 设置即可。



<br>

**变化2：画布上下文配置项中废弃了 compositingAlphaMode，改为 alphaMode**

```diff
const canvsConfig: GPUCanvasConfiguration = {
     device,
     format,
-    compositingAlphaMode: 'opaque'
+    alphaMode: 'opaque'
}
```

可能是嫌之前的 "compositingAlphaMode" 单词太长，所以现在改成了 “alphaMode”



<br>

综上所述，也就是说我们之前写的绘制三角形示例中，关于画布上下文的代码最终变动如下：

```diff
const canvsConfig: GPUCanvasConfiguration = {
     device,
     format,
-    size: [canvas.clientWidth, canvas.clientHeight],
-    compositingAlphaMode: 'opaque'
+    alphaMode: 'opaque'
}
```



<br>

**变化3：在 .wgsl 中标记当前为哪个着色阶段的关键词 @stage(xxx) 发生了变化**

最新的 API 中不再需要 "stage" 这个关键词。

1. `@stage(vertex)` 修改为 `@vertex`
2. `@stage(fragment)` 修改为 `@fragment`



<br>

> 修改之后显得更加简洁了。



<br>

本文正式开始。



<br>

**先拿 Vue/React 来举例**

在上一篇文章后半部分，我们当时使用一个可以显示 姓名 和 年龄的 Vue/React 组件进行了一个举例。

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

在这个例子中，props 中包含 2 个变量 name 和 age。

在有些情况之下，我们可能会选择将 props 中的 2 个变量合并成 1 个：

```
const PersonComponent = ({ info }) => {
    return (
        <div>
            <h2>{ info.name }</h2>
            <span>{ info.age }</span>
        </div>
    )
}
```



<br>

也就是说，组件 props 的 2 种形式：

1. 定义多个独立的属性作为参数
2. 只定义 1 个属性，但这个属性本身可以包含多个不同参数字段



<br>

讲这段是为了表达... ？



<br>

重点来了，我们再回头看一下上一篇文章中 管线配置顶点缓冲区的代码：

```diff
const pipeline = device.createRenderPipeline({
    layout: 'auto',
    vertex: {
         ...
+        buffers: [{
+            arrayStride: 2 * 4,
+            attributes: [{
+                shaderLocation: 0,
+                offset: 0,
+                format: 'float32x2'
+            }]
+        }]
    },
    ...
}
```

我们的示例中，只需要在 顶点着色阶段 传入 1 个参数。所以当时我们讲了这段话：

```
我们注意到上面配置代码中 buffers 实际是一个数组，也就意味着 我们可以传递多个 缓冲区 到入口函数中，也意味着 main 可以接收多个参数。
```



<br>

这段话本身并没有问题，但是重点来了：

**你是否发现 不光 buffers 是一个数组，它的每一个元素的 attributes 也是一个数组。**



<br>

???



<br>

**buffers 为一个数组，它的每一个元素对应一个 顶点缓冲区。**

**在同一个顶点缓冲区配置项中，又可以通过其 attributes 数组来添加多个参数的不同描述。**

**每一个参数描述的对应的 shaderLocation 值不同、缓冲区读取的起始索引值 offset 也不同，从而实现同一个顶点缓冲区携带不同的参数。**



<br>

> 这里的 buffers 相当于 Vue/React 组件中的 props



<br>

**也就是说，当我们希望传递多个参数到 顶点渲染阶段 时，有 2 种方式：**

第1种：不同的参数值存放在不同的 顶点缓冲区 中，然后依次将这些不同的顶点缓冲区 填入到 buffers 中

> 这就好像上面举例中的 `const PersonComponent = ({ name, age }) => { ... }`

<br>

第2种：将所有的参数值都存放在同一个 顶点缓冲区 中，然后依次配置多个不同的 参数描述 填入到 attributes 中

> 这就好像上面举例中的 `const PersonComponent = ({ info }) => { ... }`



<br>

当然，这两种方式之间还可以相互组合。



<br>

**实际开发中的运用：位置、UV、法线**

通常情况下 顶点缓冲区 应该分为 3 个参数：

1. 顶点位置
2. 顶点UV
3. 顶点法线



<br>

由于我们刚开始学习，所以暂时先不以 UV 和 法线 来举例了。

我们假定顶点的 x,y 为第 1 个参数，而 z 为第 2 个参数 来举例。



<br>

**第1种形式：使用多个缓冲区**

> 下面是一些伪代码

```diff
+ const zVertexArray = new Float32Array([ 0.0, 0.0, 0.0 ])

+ const zVertexBuffer = device.createBuffer({ ... })

+ device.queue.writeBuffer(zVertexBuffer, 0, zVertexArray)

const pipeline = device.createRenderPipeline({
    vertex: {
        ...
        buffers: [
            {
                arrayStride: 2 * 4,
                attributes: [{
                    shaderLocation: 0,
                    offset: 0,
                    format: 'float32x2'
                }]
            },
+            {
+                arrayStride: 1 * 4,
+                attributes: [{
+                    shaderLocation: 1,
+                    offset: 0,
+                    format: 'float32'
+                }]
+            }
        ]
    }
})
 
passEncoder.setVertexBuffer(0, xyVertexBuffer)
+ passEncoder.setVertexBuffer(1, zVertexBuffer) //设置插槽索引值 为 1
```



<br>

**第2种形式：同一个缓冲区，但使用多个参数描述**

```diff
const xyzVertexArray = new Float32Array([
    0.0, 0.5, 0.0,
    -0.5, -0.5, 0.0,
    0.5, -0.5, 0.0
])

const xyzVertexBuffer = device.createBuffer({
    size: xyzVertexArray.byteLength,
    usage: GPUBufferUsage.VERTEX | GPUBufferUsage.COPY_DST
})

device.queue.writeBuffer(xyzVertexBuffer, 0, xyzVertexArray)


//下面是管线描述中顶点缓冲区的配置伪代码
buffers: [
    {
        arrayStride: 3 * 4,
        attributes: [
            {
                shaderLocation: 0,
                offset: 0,
                format: 'float32x2'
            }, 
+            {
+                shaderLocation: 1,
+                offset: 2 * 4,
+                format: 'float32'
+            }
        ]
    }
]

+ passEncoder.setVertexBuffer(0,xyzVertexBuffer)
```



<br>

**无论以上哪种形式，对于 vert.wgsl 代码而言是相同的。**

我们只需要在 vert.wgsl 代码中明确入口函数有 2 个参数即可。

```
@vertex
fn main(@location(0) xy: vec2<f32>, @location(1) z: f32) -> @builtin(position) vec4<f32> {
    return vec4<f32>(xy, z, 1.0);
}
```



<br>

至此，我们已经搞清楚了 如何传递数据到 顶点(vertex)渲染阶段。

本文到此结束，下一节我们将学习另外一种传递数据到 WGSL 中的方式：绑定组