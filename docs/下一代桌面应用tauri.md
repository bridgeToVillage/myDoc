之前我写过一篇文章，介绍了`pake`这个小玩具，今天我来说一说它的亲爹`Tauri`。

---

做过桌面应用打包的前端同学，对 `Electron` 的名声肯定是如雷贯耳。它可以用我们熟悉的 Web 技术构建跨平台的桌面应用，VS Code、Slack、Discord 等众多知名应用都是基于它构建的。然而，`Electron` 的“缺点”也很明显：打包体积大、内存占用高，特别是在弱网传输的时候，这个毛病影响很大。。

接下来，我来介绍一个正在悄摸摸崛起的挑战者——`Tauri`。我掐指算过了，它会是下一代桌面应用框架！

### Tauri 是什么？

我先简单介绍一下 `tauri` 框架。
`Tauri` 是一个为构建更小、更快、更安全的桌面应用程序而设计的框架。它和 `Electron` 一样，允许你使用任何前端框架（如 React、Vue、Svelte 或原生 HTML/CSS/JS）来构建用户界面。
但它的核心思想与 `Electron` 有本质区别：

1. **更小的体积**：`Electron` 内置了一个完整的 Chromium 浏览器，所以每个应用都至少要带上 100MB+ 的基础文件。而 `Tauri` 则使用操作系统自带的 WebView（Windows 上的 Webview2，macOS 上的 WebKit，Linux 上的 WebKitGTK）。这意味着你的应用本身不包含浏览器，体积可以轻松做到 10MB 以下，甚至更小。
2. **更低的资源占用**：由于不自带浏览器，`Tauri` 应用的内存占用和 CPU 消耗远低于 `Electron` 应用，运行起来更加轻快。
3. **后端使用 Rust**：`Tauri` 的后端核心是用 Rust 编写的。Rust 以其高性能、内存安全和并发性著称，这为桌面应用提供了坚实、安全的后端基础。你可以通过简单的 JS API 调用 Rust 代码，处理文件系统、执行 Shell 命令等操作，既安全又高效。
4. **安全优先**：`Tauri` 默认采用“权限最小化”原则。你需要在一个叫 `allowlist`（允许列表）的配置文件中，明确开启你的应用需要使用的 API（如文件读写、网络请求等），这大大降低了应用的安全风险。
   总而言之，`Tauri` 用“借力”的方式，解决了 `Electron` 最大的体积和性能问题，同时用 Rust 提供了一个强大而安全的后端，这正是它被称为“下一代”框架的原因。目前`Tauri`版本已经到了2.0，本文后面用的也是这个版本。

---

### Windows 系统安装 Tauri

接下来我带你在 `windows` 系统安装一下 `tauri` 框架，后面有空我再介绍一下在 `ubuntu` 系统中的安装方法。
在开始之前，`Tauri` 需要两个核心依赖：Rust 语言环境和 C++ 编译工具链。

#### 1. 安装 Rust

`Tauri` 的后端是 Rust，所以首先需要安装 Rust。
访问官网：[https://rust-lang.org/tools/install/](https://rust-lang.org/tools/install/)
下载并运行 `rustup-init.exe`，按照提示安装即可。安装完成后，它会提示你重启终端或重新加载环境变量。

#### 2. 安装 Microsoft C++ 生成工具

在 Windows 上编译 Rust 和一些原生 Node.js 模块需要 C++ 构建工具。
访问官网：[https://visualstudio.microsoft.com/zh-hans/visual-cpp-build-tools/](https://visualstudio.microsoft.com/zh-hans/visual-cpp-build-tools/)
下载“Microsoft C++ 生成工具”。在安装程序中，只用勾选“使用 C++ 的桌面开发”，然后安装即可。

![](https://files.mdnice.com/user/5287/8b9c73cc-7113-4c8b-92c5-d06209c6b60b.png)

#### 3. 确认环境

打开一个新的终端（CMD 或 PowerShell），输入以下命令，如果都能正常显示版本号，说明环境就绪了。

```bash
rustc --version
npm --version
```

---

### Tauri 初体验：打包一个远程 URL

#### 1. 准备一个前端项目

我直接把一个项目复制过来，把代码管理相关文件夹删除，依赖也可以删除，然后存在一个文件夹里，就用这个项目来试下效果。

3. 在项目根目录初始化 `npm` 项目：
   
   ```bash
   npm init -y
   ```
   
   这会生成一个 `package.json` 文件。
   
   #### 2. 初始化 Tauri
   
   把这个实验项目用vscode打开，然后执行：
   
   ```bash
   # 安装 Tauri CLI 作为开发依赖
   npm install -D @tauri-apps/cli@latest 
   # 初始化 Tauri 项目
   npm run tauri init
   ```
   
   执行 `init` 命令时，它会问你几个问题，比如应用名称、窗口标题等，一路回车使用默认值即可。

这会创建一个 `src-tauri` 目录，这就是 `Tauri` 框架的后端部分。里面包含了 Rust 代码和关键的配置文件 `tauri.conf.json`。

#### 3. 尝鲜 Tauri

安装好了之后，咱先尝个鲜，把本地跑起来的应用打包进去呢？
看起来，如果只是打包一个url，这个包会相当的小。如果我把这个前端应用在服务器上用nginx部署，然后再用`tauri`打包，会不会比较有意思？
是的，这是一个非常有趣的应用场景！你可以创建一个“壳应用”，它本身不包含任何业务逻辑，只是加载一个远程的 Web 地址。这样做的好处是：

* **体积极小**：应用本身只有几 MB，用户下载飞快。
* **热更新**：你只需要更新服务器上的网页，所有用户的桌面应用内容就自动更新了，无需重新分发安装包。
  要实现这个，只需在 `tauri.conf.json` 配置文件中修改 `window.url` 字段：
  
  ```json
  // src-tauri/tauri.conf.json
  {
  "build": {
    "devUrl": "https://your-nginx-deployed-app.com" // 改成你的网址
    }
  }
  }
  ```
  
  不过，这种方式的缺点是应用部署必须得配一个nginx，安装过程不够简单，有点拖泥带水。

---

### Tauri 实战：打包本地前端应用

所以，我依然想用tauri打包一个包含所有前端应用内容的本地包，不依赖于url。来吧！
我带你创建一个完整的前端项目并将其打包成一个独立的桌面应用。

#### 3. 修改配置

前面只是尝鲜，只用到`devUrl`这个配置。接下来将是最关键的一步！你需要根据你的项目结构，修改 `src-tauri/tauri.conf.json` 文件。我这里给出一个配置文件的模板，里面有详细说明：

```json
{
  "$schema": "../node_modules/@tauri-apps/cli/config.schema.json",
  "productName": "Tauri Demo",
  "version": "0.1.0",
  // 这里要注意，会有一些格式要求，按提示改就行
  "identifier": "com.tauri.demo",
  "build": {
    "beforeDevCommand": "npm run dev",
    "beforeBuildCommand": "pm run build",
    // 告诉 Tauri 你的前端静态文件（HTML, JS, CSS）在哪个目录
    "frontendDist": "../dist"
  },
  "app": {
    "windows": [
      {
        "label": "main", // 给窗口一个唯一的 label
        "title": "Tauri Demo",
        "width": 800,
        "height": 600,
        "resizable": true,
        "fullscreen": false
      }
    ],
    "security": {
      "csp": null // 内容安全策略，开发阶段可以设为 null
    }
  },
  "bundle": {
    "active": true,
    "targets": "all",
    "icon": [
      "icons/32x32.png",
      "icons/128x128.png",
      "icons/128x128@2x.png",
      "icons/icon.icns",
      "icons/icon.ico"
    ]
  },
  "plugins": {} // 插件配置，后面会用到
}
```

**关键点解释**：

* `frontendDist`: 这是 Tauri 2.0 的新字段，替代了旧版的 `distDir`。它指向你构建完成后的前端资源目录。对于vue项目，直接指向根目录 `"../dist"` 即可。
* `beforeDevCommand` / `beforeBuildCommand`: 意思是在执行  `Tauri`的构建之前，会执行这些项目本身的 `npm run dev` 或 `npm run build` 脚本。
* `app.windows`: 窗口配置现在被放在 `app` 对象下。给窗口加了一个 `label: "main"`，这在权限配置时会用到。
* `plugins`: Tauri 2.0 的核心功能（如文件系统、Shell）都通过插件提供。需要在这里声明使用的插件。

#### 4. 开发与运行

配置完成后，就可以启动开发模式了。直接运行即可：

```bash
npm run tauri dev
```

这会先执行`npm run dev`，然后编译 Rust 后端，然后直接弹出一个桌面窗口，里面加载的就是你的 `index.html` 页面。当你修改项目文件并保存时，窗口会自动刷新，这就是热重载。

#### 5. 构建最终安装包

当你对应用满意后，就可以执行构建命令了：

```bash
npm run tauri build
```

下面这张图是执行`npm run tauri:build`后的效果，它会先执行`npm run build`，构建本地的打包文件，然后再从dist文件夹下面获取静态文件并打包为最终包。

![](https://files.mdnice.com/user/5287/3c8ed8fe-7650-438e-af7d-8f5f0a02fe1c.png)

构建完成后，你可以在 `src-tauri/target/release/bundle/` 目录下找到生成的安装包。在 Windows 上，你会看到一个 `.msi` 安装文件和一个免安装的 `.exe` 文件。这个 `.exe` 文件就是我们的最终成果，体积非常小，可以分发给用户直接运行！

---

### 总结

`Tauri` 以其轻量、高性能和高安全性的特性，为前端开发者打开了一扇通往现代桌面应用开发的新大门。虽然生态和社区相比 `Electron` 还在成长中，但其技术理念的先进性已经吸引了大量关注。如果你正准备开发一款桌面应用，或者对 `Electron` 的性能和体积感到困扰，不妨来尝一尝 `Tauri` 这道“新菜”，相信它会给你带来惊喜！

--------------------------------

后续，我将探索`Tauri`框架的`IPC`通信，给应用增加与后端通信的能力，敬请期待。
