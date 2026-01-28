昨天咱们聊了 ClawdBot 这个刷屏项目的产品层面。

今天，小源想带你深入看看它的技术架构。

看完这篇你就能理解：**为什么这个东西能做到那些看起来很酷的事情。**

## 架构核心：一个 Gateway 统治一切

ClawdBot 的整个架构，其实可以用一句话概括：

<u>**一个长期运行的 Gateway，控制所有的消息通道，通过 WebSocket 协议连接各种客户端。**</u>

听起来挺简单对吧？

但魔鬼就在细节里。

![](https://raw.githubusercontent.com/sijicuisite/articles-materials/main/articles/week-2026-01-26/2026-01-28-Wed/images/clawdbot-hype.jpg)

### Gateway 是什么？

你可以把 Gateway 理解成一个**中央指挥部**。

它负责：
- 维持和各个消息平台的连接（WhatsApp、Telegram、Slack、Discord 等等）
- 接收来自这些平台的消息
- 把消息路由给 AI Agent 处理
- 把 AI 的回复发送回去
- 管理所有客户端的连接（Mac 应用、命令行工具、Web 界面）

整个架构长这样：

```
WhatsApp / Telegram / Slack / Discord / Signal
               │
               ▼
       ┌───────────────┐
       │   Gateway     │
       │ (控制平面)     │
       │ ws://127.0.0.1│
       └───────┬───────┘
               │
       ┌───────┴───────┐
       │               │
    AI Agent      CLI/macOS App
```

关键点在于：<u>**Gateway 是整个系统的唯一真理源**。</u>

它是唯一一个持有 WhatsApp 会话、Telegram Bot Token、Slack API 密钥的地方。

所有的客户端，包括你的 Mac 应用、命令行工具，甚至是 iOS/Android 节点，都必须通过 WebSocket 连接到这个 Gateway 才能工作。

### 为什么选择 WebSocket？

这是个聪明的设计决策。

传统的做法是让每个客户端自己维护和消息平台的连接。但这样会有几个问题:

1. **状态同步困难** - 如果你在手机上回复了消息，电脑上的客户端可能不知道
2. **资源浪费** - 每个客户端都要维持长连接
3. **安全隐患** - API 密钥散落在多个设备上

WebSocket 的好处是:
- 双向通信，服务器可以主动推送消息给客户端
- 单个长连接，比 HTTP 轮询高效得多
- 所有敏感凭证只存在 Gateway 一个地方

![](https://raw.githubusercontent.com/sijicuisite/articles-materials/main/articles/week-2026-01-26/2026-01-28-Wed/images/ai-capabilities-vs-risks-split-screen.png)

## 三层架构：Gateway、Channels、Nodes

ClawdBot 的架构可以分成三层。

### 第一层：Gateway 核心

Gateway 运行在你的某台设备上（可能是 Mac Mini,也可能是云服务器）。

默认监听 `127.0.0.1:18789` 端口。

启动命令很简单:

```bash
moltbot gateway --port 18789 --verbose
```

Gateway 做了这些事:
1. **维护消息通道连接** - 和 WhatsApp/Telegram 等平台保持长连接
2. **暴露 WebSocket API** - 供各种客户端连接
3. **消息路由** - 把收到的消息发给 AI,把 AI 的回复发回去
4. **事件推送** - 主动推送 `presence`(在线状态）、`heartbeat`(心跳）、`cron`(定时任务）等事件

### 第二层：Channels（消息通道）

这一层负责和具体的消息平台对接。

ClawdBot 支持的通道包括:
- **WhatsApp** - 通过 Baileys 库
- **Telegram** - 通过 grammY 库
- **Slack** - 通过 Bolt 框架
- **Discord** - 通过 discord.js
- **Signal** - 通过 signal-cli
- **iMessage** - 通过 imsg
- 还有各种扩展通道：BlueBubbles、Matrix、Microsoft Teams 等

每个通道都是独立的模块，遵循统一的接口规范。

这样的好处是：<u>**要支持新的消息平台，只需要写一个新的 Channel 适配器就行**。</u>

### 第三层：Nodes（节点）

这一层是 ClawdBot 的特色功能。

Nodes 是指连接到 Gateway 的各种**设备节点**，比如:
- macOS 电脑
- iOS 手机
- Android 手机
- 甚至是无头（headless）服务器

Nodes 可以提供各种**能力**(capabilities）:
- `camera.*` - 拍照、录像
- `screen.record` - 屏幕录制
- `location.get` - 获取位置
- `canvas.*` - 渲染可视化界面
- `system.run` - 执行系统命令（macOS 独有）

这意味着什么？

<u>**意味着 AI 可以通过 WhatsApp 给你发消息，让你的手机拍张照，然后把照片发回聊天窗口。**</u>

或者让你的 Mac 执行一个 Python 脚本，把结果发到 Telegram。

![](https://raw.githubusercontent.com/sijicuisite/articles-materials/main/articles/week-2026-01-26/2026-01-28-Wed/images/ai-robot-car-racing-scene.png)

## 连接生命周期：客户端怎么和 Gateway 通信?

让我们看看一个客户端（比如 Mac 应用）是怎么和 Gateway 交互的。

### 1. 握手阶段

客户端首先发送 `connect` 请求:

```
Client --> Gateway: req:connect
        (params: {auth: {token: "xxx"}， device: {...}})

Gateway --> Client: res (ok)
        (payload: hello-ok, 包含当前状态快照)
```

如果 Gateway 设置了 `CLAWDBOT_GATEWAY_TOKEN`，客户端必须提供正确的 token,否则连接直接被关闭。

这是第一道安全防线。

但问题是:**大部分用户根本不知道要设置这个 token**。

### 2. 订阅事件

连接成功后,Gateway 会主动推送各种事件:

```
Gateway --> Client: event:presence
        (哪些通道在线、哪些离线)

Gateway --> Client: event:tick
        (定期心跳，确保连接存活)
```

客户端可以基于这些事件更新 UI。

### 3. 发送 Agent 请求

当用户在 WhatsApp 上发消息给 ClawdBot 时:

```
WhatsApp --> Gateway --> AI Agent
        (消息内容、上下文、可用工具)

AI Agent --> Gateway --> WhatsApp
        (AI 的回复，可能包含流式输出)
```

整个过程是**异步**的。

Gateway 先返回一个 `runId` 表示任务已接受，然后通过 `event:agent` 流式推送 AI 的输出。

最后发送 `res:agent` 表示任务完成。

这种设计让用户体验很流畅——你在 WhatsApp 里看到的是逐字输出的回复，而不是等半天才蹦出一整段。

## 安全机制

ClawdBot 也设计了一些安全防护机制。

### 1. Device Pairing（设备配对）

新设备连接时，需要通过配对流程。

Gateway 会生成一个**配对码**，用户需要通过命令行确认：

```bash
moltbot pairing approve <channel> <code>
```

确认后，设备会被加入白名单。这个机制防止了未授权设备连接到 Gateway。

### 2. DM Policy（私信策略）

针对 Telegram/WhatsApp 等平台的陌生人消息，ClawdBot 提供了两种策略：

- `pairing` - 陌生人发消息时，Bot 会返回一个配对码
- `open` - 接受所有人的消息（需要显式配置）

默认是 `pairing` 模式。

### 3. Tool Policy（工具策略）

ClawdBot 允许配置**哪些工具可以在哪些上下文中使用**。

比如：
- `system.run` 只能在本地 CLI 使用，不能通过远程消息触发
- `browser.navigate` 可以限制访问的域名范围

这些策略给了用户灵活的控制权限。

![](https://raw.githubusercontent.com/sijicuisite/articles-materials/main/articles/week-2026-01-26/2026-01-28-Wed/images/github-issues-count.jpg)

## 架构设计的亮点

看完整个架构，可以总结几个设计亮点：

### 1. 单一控制平面

Gateway 作为唯一的控制中心，统一管理所有消息通道和客户端连接。

这种设计让状态管理变得简单，避免了分布式系统的复杂性。

### 2. 协议标准化

所有客户端（Mac 应用、CLI、iOS/Android 节点）都通过同一套 WebSocket 协议与 Gateway 通信。

这让添加新的客户端类型变得很容易。

### 3. 模块化通道设计

每个消息平台都是独立的 Channel 模块，遵循统一接口。

<u>**要支持新平台，只需要实现一个新的适配器。**</u>

### 4. 能力扩展机制

Nodes 可以向 Gateway 注册自己的能力（相机、屏幕录制、位置等）。

AI Agent 可以根据需要调用这些能力，实现跨设备的自动化。

## 写在最后

ClawdBot 的架构设计展示了一个有趣的思路：

<u>**通过统一的 Gateway 控制平面，把 AI 能力延伸到你所有的设备和通讯渠道。**</u>

Gateway + Channels + Nodes 的三层设计，清晰、灵活、可扩展。

WebSocket 协议保证了实时性和双向通信能力。

模块化的 Channel 设计让支持新平台变得很容易。

Nodes 机制则打通了不同设备之间的协作。

这个架构本身是**技术上可行且优雅的**。

它证明了一件事：**在本地设备上运行的 AI 助手，可以做到和云端服务一样强大甚至更强大。**

因为它能直接访问你的文件、应用、设备能力，不需要经过云端中转。

这种**端侧 AI**的思路，值得更多项目借鉴。

当然，强大的能力也意味着需要更仔细的权限管理和安全设计。

但这是另一个话题了。

![](https://raw.githubusercontent.com/sijicuisite/articles-materials/main/articles/week-2026-01-26/2026-01-28-Wed/images/agentic-ai-freedom-security-balance.png)

---

**关注小源，每日更新最新的 AI 资讯和深度文章。**
