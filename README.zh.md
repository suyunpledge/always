# always

**一个跑在浏览器里的个人 AI 工作台 —— 通过本仓库的安卓壳，也可以作为原生 App 使用。**

> 本仓库只包含安卓 WebView 壳。它所包装的平台 **always** 是一个独立的 Next.js Web 应用；壳只是把它送上安卓的薄封装。下面先讲平台是什么，再讲壳怎么构建。

## always 是什么

always 把原本散落在七八个标签页里的 AI 工具收进一处：多模型对话、联网搜索、代码生成、图片创作、长篇写作。它是 Web 应用（可作为 PWA 安装），所以同一份会话能跟着你从桌面浏览器走到手机，再走到安卓壳里。

用应用自己的话说：*多模型对话、联网搜索、代码生成与图片创作，一处搞定。*

### 三个工作模式

应用围绕三个模式组织，每个模式有各自的界面和工具链，而不是同一个聊天框换个名字：

| 模式 | 用途 | 亮点 |
| --- | --- | --- |
| **对话模式** | 日常对话与调研 | 多模型对话、知识问答、联网搜索、图片理解 |
| **编程模式** | 处理代码 | 代码生成、调试、review，并做了工具 token 优化，长工具输出不会把上下文吃光 |
| **写作模式** | 小说与长文创作 | 章节管理、文笔画像、分级记忆注入，让助手跨章节保持连贯 |

### 三个模式之外

- **图片生成**，并带一个作品画廊
- **仓库审查** —— 指向一个仓库，让它过一遍代码
- **知识库**，可挂到对话里用
- **网页提取** —— 把页面正文抽出来当上下文
- **GitHub 绑定**，服务于审查流程
- **分级记忆**接入：历史对话自动检索注入，而不是整包塞进上下文
- **多用户账号 + 局域网登录**，自建实例可以安全地多人共用
- **亮/暗主题**，以及可安装到桌面的 PWA 清单

### 部署形态

always 是按自托管设计的。Web 应用跑在纯 Node（`server.js`）后面，单端口对外，页面由 Next.js 提供，后端以 API 路由形式暴露（鉴权、对话、模型、图片、知识库、代码审查、爬取、GitHub、记忆服务等）。典型部署是一台主机跑 Web 应用，旁边跑 Ollama 和本地记忆服务。作者自己的实例跑在非标准端口的纯 HTTP 上，并被用作壳的默认目标 —— 照搬这套配置之前，请先看下面的安全说明。

## 安卓壳（本仓库）

这是一个单 Activity 的 WebView 容器，把 always Web 应用包装成原生安卓应用。壳本身零业务逻辑，所有能力都在 Web 端持续演进 —— 因此**壳代码几乎不需要维护**，网站更新后 APK 无需重新打包，刷新即得。

> 默认指向作者的演示服务器（HTTP）。克隆本仓库后请先修改 `java/com/aiplatform/app/MainActivity.java` 里的 `START_URL` 再构建，详见下文。

### 功能特性（壳层实现）

- **明文 HTTP 放行**（`usesCleartextTraffic`）：演示服务器是 HTTP，未开启时 Android 9+ 会白屏
- **JS 对话框接管**（`onJsAlert / onJsConfirm / onJsPrompt`）：Web 端的 `window.confirm`（如「解除绑定」「提交代码」）若不接管会静默失败
- **GitHub OAuth 可用**：所有 http/https 均留在 WebView 内导航，授权回跳不会跳到系统浏览器；UA 伪装为现代 Chrome
- **文件选择**（`onShowFileChooser`）：支持 Web 端的聊天传图 / 附件上传
- **下载支持**（DownloadListener）：Web 端导出的文件走系统下载管理器
- **登录态持久化**：`onPause` 时 `CookieManager.flush()`

### 构建步骤

依赖：JDK 17、Android SDK（build-tools 34 + platform android-34）、Python 3（Pillow，生成图标用）。

1. 修改目标站点：`java/com/aiplatform/app/MainActivity.java` → `START_URL`
2. 生成图标：`python make_icons.py`（输出到 `res/mipmap-*`）
3. 构建：PowerShell 里运行 `.\build-apk.ps1`
   （手工链：`aapt2 compile/link → javac → d8 → zipalign → apksigner`，产物输出到仓库外 `..\always.apk`）

脚本顶部的 `SDK / PLATFORM / JDK` 三个路径按本机位置调整；签名密钥不随仓库分发，首次构建时脚本会自动生成（storepass/keypass 见脚本内常量），**请自行保管好自己的 keystore**，切勿提交。

### 安全说明

- 本壳加载的是 HTTP 站点并已放行明文流量 —— 若你的站点上了 HTTPS，请记得收紧（移除 `usesCleartextTraffic` 或改 `MIXED_CONTENT_NEVER_ALLOW`）
- WebView 默认信任 Web 端内容；请确保你加载的服务器是自己的

### 目录结构

```
AndroidManifest.xml                      清单（usesCleartextTraffic / 权限 / Activity）
java/com/aiplatform/app/MainActivity.java   全部壳逻辑（~300 行）
res/mipmap-*/ic_launcher.png             启动图标（make_icons.py 生成）
make_icons.py                            图标生成脚本（Pillow）
add_dex.py                               把 classes.dex 注入 base.apk
build-apk.ps1                            一键构建脚本
```

### 安全边界（诚实说明）

这份代码是一个**自用/演示**性质的壳，安全模型有意保持简单，以下边界请知悉：

- **地址混淆不是加密**：`START_URL` 用 XOR + Base64 存放，密钥就在二进制里——它只防"反编译一眼看到服务器地址"，任何愿意花一分钟的人都能还原。不要把它当机密机制。
- **全链路明文**：后端当前是 HTTP（裸 IP + 非常规端口），且壳放行了明文流量。这意味着在同一网络下，登录态与内容对同网段的攻击者可见。SSL 相关处理（`onReceivedSslError` 默认拒绝）在这个前提下是**为未来 HTTPS 准备的**，不构成本代际的保护。
- **端口 3389 是历史原因**，不是刻意伪装成 RDP：部署时云主机安全组只放行了 22/3389，直接复用了它。后续上 HTTPS 会换回 443。
- **不建议作为生产客户端分发**：如果要发给别人用，请先把后端上 HTTPS（配好证书后，把 `network_security_config.xml` 里的 `<pin-set>` 填上证书公钥哈希，即完成证书锁定）。

### 后续计划

1. 后端 HTTPS（域名或自签 + 证书锁定）
2. 服务端已具备：多用户数据隔离、公开模式强制鉴权、生成图不裸奔（见服务端私有仓库）
