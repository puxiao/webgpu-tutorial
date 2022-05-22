# webgpu-tutorial
从今天 2021年12月22日 开始学习和探索 WebGPU。



<br>

本系列教程对应的示例代码仓库：https://github.com/puxiao/react-webgpu-samples



<br>

我新创建的另外一个仓库，专门用来讲解 WebGPU 的着色器语言 WGSL

仓库地址：https://github.com/puxiao/wgsl-tutorial



<br>

**为什么要学习WebGPU？**

* 十几年前 Flash 的出现颠覆了网页表现形式

* 后来 HTML5 的出现又逐渐取代了 Flash

* 再后来 WebGL 的出现让浏览器端可以实时渲染出 3D 场景

  > 第一次在浏览器上看到可交互的 3D 模型，那种震撼是吓人的。

* 此刻 WebGPU 又要来取代 WebGL 了

  > 尽管 WebGPU 目前还处于 草案阶段，估计正式版 v1.0 还要半年后才会发布。

WebGPU——下一代 Web 图形技术。



<br>

**你会 WebGPU 吗，你就要开始写教程了？**

我不会 WebGPU，我也是才开始学习，但是我个人的学习风格是边学习边写教程。

1. 用教会别人的形式来学习，这种方式对我个人而言真的很有用。

   > 我从 网页设计师 转行 Web 开发就全部是用这种方式走过来的。

2. 若我的学习分享能够帮助同样的新手小白，能给我点个赞，这反向会激励我持续的学习。

   > 每次看到 Github 上有人 starred 或 forked 我写的东西，真的很开心。



<br>

**我的前端技术栈介绍：**

1. 前端 JS：React Hooks、TypeScript

   > 如果你习惯使用 Vue，看本教程是完全没有问题的。
   >
   > 我强烈建议你一定要使用 TypeScript。

2. 图形学：Three.js、Cesium.js、3D 图形化 基础知识

   > 我对原生 WebGL 并不熟，也不会写 GLSL，我只是会 Three.js。
   >
   > 3D 图形化的一些基础知识我是会的，例如 点乘、叉乘、向量归一、矩阵转换、齐次坐标、球极坐标、管线渲染 等等。
   >
   > 我会一些基础的 Cesium.js，我不会 Babylon.js。

3. 后端：Koa、MongoDB、Nginx、Docker、CentOS ...

   > 我认为前端一定要懂点网络通信、Shell 命令，这对于理解前端开发非常有帮助。



<br>

**我的学习和写作计划：**

1. WebGPU 基础知识

2. WebGPU 常见类、属性和方法

3. Three.js 中 WebGPU 相关类的源码阅读

   > 目前 Three.js 中的 WebGPU 相关类代码并未出现在其官方文档中。
   >
   > 代码位于：https://github.com/mrdoob/three.js/tree/dev/examples/jsm/renderers/webgpu

4. WGSL 基础学习

   > GLSL 是 WebGL 渲染器开发语言
   >
   > WGSL 是 WebGPU 的渲染器开发语言

5. 我会单独创建一个目录，用来存放 WebGPU 相关新闻或学习资料。

   > 或许我会尝试翻译某些文章



<br>

**本系列教程中，我打算开始添加配图。**

> 我以前写的 React hooks、Three.js 等教程，几乎都是纯文字，没有配图。



<br>

**学习 WebGPU 你需要提前具备的一些知识：**

1. ES6 各种新的语法特性，尤其是 Promise、async/await

   > WebGPU 中大量的操作都是异步的，因此你一定要会 Promise

2. 图形学：无论 WebGL 还是 WebGPU，图形学基础是一定要掌握的，否则你根本没办法去理解 3D 图形化开发中遇到的很多概念和用法。

   > 对于前端人而言，图形学才是我们学习 3D 开发最大的难点。
   >
   > 目前有一个事实，那些原本做 3D 开发的人，例如 3D 建模 渲染、3D 游戏开发、U3D 的从业者他们转行做 Web 3D，对我们前端进行 `降维打击`，我们在他们面前真的就是 学渣、被秒杀。



<br>

> 如果精力可以，最好去学习一下 Rust，因为这个是大趋势，对于开发 WebGPU 帮助非常大。
>
> 我计划明年也学习一下 Rust。



<br>

**特别强调：**

我也是刚开始学习，若中间有理解不到位，讲错的地方，还请担待。

若能对我的错误指正，不胜感激。

本教程地址：https://github.com/puxiao/webgpu-tutorial

> 若发现我有写错的地方，欢迎提交 Issues



<br>

**欢迎与我交流：**

Github：https://github.com/puxiao

邮箱：yangpuxiao@gmail.com

微信：78657141(同QQ)

![](https://raw.githubusercontent.com/puxiao/webgpu-tutorial/main/imgs/me_qrcode01.jpg)



<br>

**教程同步：**

本教程文章也会同步发布到我申请的个人微信公众号里，公众号名称就叫：WebGPU

> 目前还真的没有其他公众号专注于 WebGPU，所以这个名称我就申请到了。

![](https://raw.githubusercontent.com/puxiao/webgpu-tutorial/main/imgs/me_qrcode02.jpg)

<br>

除了写作，若还有精力，我会尝试将文章转成视频，发布在我的个人微信视频号里。

> 个人微信视频号名称也是：WebGPU

> 从侧面也反映出，目前前端搞 WebGPU 的人真的很少，弯道超车的机会来了，加油吧。



<br>

**关于版权：**

1. 若你发现我引用了你的文章、图片、PPT 或视频截图、或者其他你认侵犯到你权利的地方，请通知我。
2. 若你想转发我的文章，请先告诉我，我希望至少你能注明文章来源和作者。



<br>

**前端同仁，加油，一起学习 WebGPU ！**

