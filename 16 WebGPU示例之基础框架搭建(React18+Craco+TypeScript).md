# 16 WebGPU示例之基础框架搭建(React18+Craco+TypeScript)

**我们通过 create-react-app 来创建 React18 项目，然后通过 craco 来添加 webpack 配置，再通过 typescript 来做类型检查，以此来创建一个运行 WebGPU 的前端项目。**



<br>

本文要讲的是基于 react + webpack ，当然或许你平时使用的是 vue + vite。

无论是哪种框架或编译方式，这都不太重要，对于编写 WebGPU 核心的 JS 代码他们都是相同的。



<br>

> 其实原本计划本篇就开始讲解如何绘制三角形，但是想了想，觉得有必要先把 前端项目环境配置 讲解清楚。
>
> 后面我们所有编写的 WebGPU 示例都将在这个项目背景中进行。



<br>

#### 第1步：使用 create-react-app 创建一个 React18 项目

我使用的是 create-react-app 5.0.1 这个版本，相对于 4.x.x 版本，它的不同之处在于：

1. 它创建的项目是基于 webpack5 编译项目，而不是 webpack4
2. 它默认会安装最新版的 react，而目前最新版的 react 就是 react 18



<br>

**初始化项目命令：**

```
yarn create react-app hello-webgpu --template typescript
```

> 创建成功后，打开 package.json，检查一下 react 和 react-dom 是不是 `^18.0.0`。



<br>

**可选操作：安装 node-sass**

如果你不使用 sass，忽略这一步。

```
yarn add node-sass@6.0.0
```

> 最新版的 node-sass 我这边一直安装不成功，所以选择安装 6.0.0 版本。



<br>

#### 第2步：安装并配置 @webgpu/types

**安装命令：**

```
yarn add @webgpu/types
```

> 目前 @webgpu/types 最新版本为 0.1.13



<br>

**修改tsconfig.json：**

1. 修改 target 将 es5 改为 esnext
2. 添加 typeRoots 配置
3. 增加 alias(路径别名) 对应的 paths 配置

```diff
{
    "compilerOptions": {
-        "target": "es5",
+        "target": "esnext",

+        "paths": {
+            "@/components/*": ["./src/components/*"],
+            "@/hooks/*": ["./src/hooks/*"],
+            "@/shader/*": ["./src/shader/*"]
+         },

+        "typeRoots": [
+            "node_modules/@types",
+            "node_modules/@webgpu/types"
        ]
    }
}
```



<br>

#### 第3步：安装并配置 @craco/craco

> 注：使用 creact-react-app 创建的 react 项目我们默认是无法修改 webpack 相关配置的，除非你使用  `yarn run eject` 强行暴露配置。而 @craco/craco 可以做到不 eject 的情况下让我们添加修改 webpack 配置。



**安装命令：**

```
yarn add @craco/craco
```



<br>

**修改package.json的scripts：**

将原本编译使用的 react-scripts 修改为 craco

```diff
"scripts": {
-    "start": "react-scripts start",
+    "start": "craco start",
-    "build": "react-scripts build",
+    "build": "craco build",
-    "test": "react-scripts test",
+    "test": "craco test",
    "eject": "react-scripts eject"       
}
```



<br>

#### 第4步：添加对 .wgsl 的支持，顺道也配置一下 alias

> 这是本文的重点

由于 WebGPU 的渲染器语言为 WGSL，所以以后渲染器(shader)相关的代码我们都会写到以 .wgsl 为后缀的文件中。

但是无论是 webpack 还是 vite，他们默认都不认识 这种 .wgsl 文件资源，所以都需要我们配置一下。



<br>

**添加global.d.ts：**

在项目 src 目录下创建 global.d.ts 文件，增加对 .wgsl 文件的类型描述。

```
declare module '*.wgsl' {
    const shader: string;
    export default shader;
}
```

> 上面的配置意思是告诉 typescript 以后凡是 import 为 .wgsl 的文件，都把它的返回结果当做字符串。



<br>

**添加craco.config.ts：**

在与 src 目录的平级处，创建 craco.config.ts 文件，用来添加 webpack 相关的配置。

> 如果你不使用 typescript，那么忽略本文中所有和 typescript 相关的配置，此时你创建的文件名应该是 craco.config.js



<br>

**我们需要告诉 webpack 如何处理 .wgsl 文件资源**。

> 这里我们也顺道配置一下 alias

```
//craco.config.ts

import path from 'path'
module.exports = {
    webpack: {
        alias: {
            "@/components": path.resolve(__dirname, "src/components/"),
            "@/hooks": path.resolve(__dirname, "src/hooks/"),
            "@/shader": path.resolve(__dirname, "src/shader/")
        },
        configure: {
            module: {
                rules: [
                    { test: /\.wgsl$/, type: "asset/source" }
                ],
            }
        }
    }
}
```



<br>
至此，我们已经完成了整体项目的配置。

在接下来写示例时，我们可以通过 `import xxx form './shader/xx.wgsl'` 这种方式引入 .wsgl 代码了。



<br>

补充：上面的操作是针对 webpack5 的，如果你使用的是 webpack4，那么你需要做的是：

1. 安装 raw-loader

   > webpack5 已经内置了 raw-loader，而 webpack4 需要你自己添加

2. 修改上述配置项中关于 rules 的配置方式

   ```diff
   - { test: /\.wgsl$/, type: "asset/source" }
   + { test: /\.wgsl$/, use: "raw-loader" }
   ```



<br>

#### 补充：vite 配置 .wgsl 的方式

在 vite 中是通过 url 的 query 参数 `?raw` 来表明 .wgsl 文件类型的。

```
import xxx from './shader/xx.wgsl?raw'
```

> raw 这个词是本身翻译为 生肉，实际上就是指 “未经处理” 的原始数据

> vite  还有其他支持的参数：?import、?html-proxy、?worker、?shareworker、?url ，具体含义本文不做过多介绍。



<br>

与此同时，vite 项目需要修改 tsconfig.json 中 types 的值：

```
{
    "compilerOptions": {
        "types": ["vite/client", "@webgpu/types"]
    }
}
```



<br>

#### 第5步：VSCode安装 wgsl 代码高亮插件

在 VSCode 插件中搜索：wgsl，找到并安装一个名为 `wgsl-analyzer` 的插件。

> 补充说明：
>
> 事实上目前 VSCode 中只有 3 个和 wgsl 相关的插件：WGSL、wgsl-analyzer、WGSL Literal
>
> 尽管第 1 个插件 WGSL 使用者数量最多，但是它没有格式化 .wgsl 文件的能力，而 wgsl-analyzer 虽然看上去使用者数量比不过前者，但是它同时支持代码高亮和格式化 .wgsl 代码，所以我推荐使用它。



<br>

#### 第6步：安装谷歌开发版浏览器(Chrome Canary)

目前 WebGPU 并未正式发布，所以我们只能运行在 开发版 的浏览器中。

这里我们选择 谷歌金丝雀(canary)版浏览器。

> 目前最新版为 103。



<br>

**金丝雀版下载地址：**

https://www.google.com/intl/zh-CN/chrome/canary/

> 当然你可能无法访问该网址



<br>

**开启WebGPU：**

第1步：打开 Chrome 开发版，在地址栏中输入：`about://flags`

第2步：在打开的页面中，搜索 "webgpu"，找到 `Unsafe WebGPU`，将其设置为 `Enabled`。



<br>

至此，我们已经配置好了基本的 WebGPU 前端项目，那么接下来就开始真正编写示例了。

先从最基础的，绘制一个 三角形开始。

