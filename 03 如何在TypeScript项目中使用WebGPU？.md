# 03 如何在TypeScript项目中使用WebGPU？

**答：安装 WebGPU 对应的 .d.ts 类型包 @webgpu/types**



<br>

> 提示：本文很短。



<br>

#### 创建一个基于 React + TypeScript 的 WebGPU 项目

* 第1步：使用 Create-React-App 初始化一个项目

  > 注：我使用的 create-react-app 是最新版 5.0，这个版本创建的 React 项目采用 webpack 5 构建项目。

  ```
  yarn create react-app hello-webgpu --template typescript
  ```

* 第2步：安装 WebGPU 对应的 .d.ts 类型包 `@webgpu/types`

  ```
  yarn add @webgpu/types
  ```

  第3步：修改 `tsconfig.json`，添加下面一行配置

  ```diff
  {
    "compilerOptions": {
  +    "typeRoots": [ "./node_modules/@webgpu/types", "./node_modules/@types"]
    }
  }
  ```

  > 修改好配置文件后，为了确保一定生效，建议重启一次 VSCode。




<br>

> 如果你使用 Vue3，那么上面除了第 1 步不同外，剩余 2 步骤完全相同。



<br>

#### 按照惯例，写个 "Hello world"

*hello-webgpu/src/App.tsx*

```
import { useEffect, useState } from 'react';

function App() {
  const [gpuAvailable, setGPUAvailable] = useState(false)

  useEffect(() => {
    if (navigator.gpu) {
      navigator.gpu.requestAdapter().then(adapter => {
        if (adapter) {
          setGPUAvailable(true)
        }
      })
    }
  },[])

  return (
    <div>
      当前 WebGPU: { gpuAvailable ? '可用' : '不可用' }
    </div>
  );
}

export default App;
```



<br>

本文到此结束。

接下来，我们将开始认真学习一下 WebGPU 的几个入门函数。