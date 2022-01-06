# 02 如何在浏览器中调试WebGPU?

**答：目前阶段只能在各大浏览器的开发版中才可以开启和调试WebGPU。**



<br>

#### 各大浏览器的开发版？

由于目前 WebGPU 并未正式发布，所以在各大浏览器的标准版中都不可以使用或调试 WebGPU。

但是，各大浏览器 Chrome、Firefox、Safari、Edge 的开发版都早已支持 WebGPU。



<br>

#### 各大浏览器的开发版长什么样子？

![](https://puxiao.com/webgpu_tutorial/imgs/browser_dev.jpg)

上图中，左侧为浏览器对应的开发版图标，右侧为标准版。



<br>

**浏览器开发版下载地址：**

Chrome：https://www.google.com/intl/zh-CN/chrome/dev/

Safari：https://developer.apple.com/safari/technology-preview/

Firefox：https://www.mozilla.org/zh-CN/firefox/developer/

Eged：https://www.microsoftedgeinsider.com/en-us/download



<br>

> 本文提到的浏览器 “标准版” 就是正式版，而 “开发版” 实际上是一个统称。
>
> 不同浏览器的开发版或许有自己特殊的称呼，例如 Chrome 的开发版也被称呼为 “金丝雀版”。



<br>

再次强调，本文以下提到的各个浏览器均是指它们的开发版。



<br>

#### Chrome如何开启WebGPU ?

第1步：打开 Chrome 开发版，在地址栏中输入：`about://flags`

第2步：在打开的页面中，搜索 "webgpu"，找到 `Unsafe WebGPU`，将其设置为 `Enabled`。

![](https://puxiao.com/webgpu_tutorial/imgs/unsafe_webgpu.jpg)

> 那段英文的翻译为：
>
> **不安全WebGPU**
>
> 开启实验性WebGPU API的访问。警告：由于WebGPU API尚未实现GPU沙盒，因此可以读取其他进程的GPU数据——在 Mac、Windows、Linux、Fuchsia 系统中。

> 修改设置后，需重启 Chrome 开发版浏览器



<br>

#### 验证WebGPU是否可用

打开浏览器调试工具，在 Console 面板中验证：

```
console.log(navigator.gpu)
```

如果输出有值 `GPU`，即证明此时浏览器已可以使用 WebGPU 了。

![](https://puxiao.com/webgpu_tutorial/imgs/navigator_gpu.jpg)

<br>

至此，可以在Chrome中开发 WebGPU 了。



<br>

其他浏览器开启 WebGPU 的操作几乎相同。



<br>

#### Firefox开启WebGPU

第1步：打开 Firefox 开发版，在地址栏中输入：`about:config`

第2步：搜索 WebGPU，在出现的 `dom.webgpu.enabled` 项目中，通过点击右侧的 双箭头 图标，将当前值修改为 true。

第3步：重启浏览器后，同样在控制台执行 `console.log(navigator.gpu)`，若打印有值即表示已开启成功。



<br>

#### Edge开启WebGPU

第1步：打开 Edge 开发版，在地址栏中输入：`edge://flags/`

第2步：搜索 WebGPU，在出现的 `Unsafe WebGPU` 项目中，点击右侧的 启用按钮。

第3步：重启浏览器后，同样在控制台执行 `console.log(navigator.gpu)` ...



<br>

#### Safari开启WebGPU

由于我没有苹果电脑和苹果手机，所以我实际并没有在 Safari 上真正测试过。

以下操作步骤是我在网上看到的。

**macOS：**

1. Safari > Preferences
2. 点击 Show Develop menu in menu bar
3. Develop > Experimental Features
4. 在出现的选项列表中 勾选 WebGPU
5. 重启 Safari

<br>

**iOS：**

1. 打开系统设置，找到 Safari
2. 点击 高级(Advanced)
3. 点击 试验特性(Experimental Features)
4. 找到并打开 WebGPU 选项
5. 重新打开 Safari



<br>

假设你的 苹果电脑 或 苹果手机 按照上面步骤并未找到 WebGPU，那有可能是因为你的系统不是较新的版本。

> macOS版本应该大于 10.13
>
> iOS 版本应该需要大于 11



<br>

#### 查看WebGPU在线示例

当你觉得你已经将自己的浏览器开启过 WebGPU 后，那么可以访问这个网页：

https://austin-eng.com/webgpu-samples/samples/helloTriangle

如果你能看到红色三角形，则表示你的浏览器已开启 WebGPU。

![](https://puxiao.com/webgpu_tutorial/imgs/helloTriangle.jpg)

> 我的微信公众号：WebGPU 的 logo 就源于这个红色三角图形。

> 特别提醒：你不需要去认真看它的代码，否则很容易被劝退。具体 WebGPU 的代码开发，我们后面会逐渐学习，先不用操之过急。



<br>

#### 关于 Chrome 的 origin trial token 特别补充：

Chrome 特别希望你在开发测试 WebGPU 的过程中，将一些使用感受、Bug 反馈给他们。

反馈的前提是你需要先注册一个 Google Developer(开发者) 的账号。

官网地址：https://developers.google.com/

然后，你还需要申请一个 `origin trial token for WebGPU (源测试令牌)`，将申请到的 token(令牌) 添加到你的网页中，以便于 Chrome 对你的网页情况进行信息收集、统计分析。

<br>

> origin trial token 直白翻译为 `产地来源证`，本文统一将其称呼为：源测试令牌

> 我看网上有人将 `origin trial` 翻译成 “起源试验”。



<br>

**申请一个 origin trial token**

1. 申请前提1：你需要有一个 `科学爱国上网神器`，否则 Google 你都访问不了，就别说后面的事情了。
2. 申请前提2：你需要有一个 google developer 账户

<br>

第1步：打开申请地址 https://developer.chrome.com/origintrials/#/view_trial/118219490218475521

> 这个页面第一次访问略有会有点慢，会显示 "Loading..."，耐心等待一会。

打开之后，页面如图：

![](https://puxiao.com/webgpu_tutorial/imgs/trial_token1.jpg)

点击 `REGISTER` 按钮，继续下一步。

<br>

第2步：在新页面中，“Web Origin” 输入你的站点域名，“Expected usage” 选择 “0 to 10,000” 即可。

同时勾选页面上所有的选项。

> 下半截那些选项几乎都是 Chrome 的免责条款。告诉你 WebGPU 处于内测阶段，若发生什么未知意外还请您多担待。

![](https://puxiao.com/webgpu_tutorial/imgs/trial_token2.jpg)

点击 “REGISTER” 继续下一步。

<br>

申请成功后，你会看到这样的一个结果页：

![](https://puxiao.com/webgpu_tutorial/imgs/trial_token3.jpg)

> 这个是我之前申请时候的结果页

记得将得到的 Token 复制出来。



<br>

> 若在使用 WebGPU 的过程中有信息需要反馈，则可点击 “FEEDBACK” 按钮提交你的反馈信息。
>
> 或者你可以访问 WebGPU 的 Github 讨论区：https://github.com/gpuweb/gpuweb/discussions



<br>

**使用 origin trial token**

一共有 2 种方式将申请到的 token 添加到网页中。

第1种：在网页头部信息中添加一个 `meta` 标签，例如：

```
<meta http-equiv="origin-trial" content="your-token">
```

> 将之前申请到的 token 填入 content 中

<br>

第2种：如果可以配置服务器程序，例如 Koa、Nginx 等，可以在他们的返回头部信息(HTTP header) 中统一添加 `Origin-Trial`信息，例如：

```
Origin-Trial: your-token
```



<br>

上面第1种做法最为简单，推荐使用。

上面第2种做法，需要你有一定的后端服务器经验。



<br>

至此 Chrome 的 origin trial token 补充完毕。



<br>

> 以上关于 origin trial token 的作用和介绍仅为我个人的理解，若有不正确欢迎指正。
>
> 最权威的可参考官方的解释：http://googlechrome.github.io/OriginTrials/developer-guide.html



<br>

浏览器已配置好，那么接下来就要开始写代码了。

由于我个人非常喜欢 TypeScript，所以下一节，我们先基于 React + TypeScript 搭建一个最为简单的 WebGPU 项目。