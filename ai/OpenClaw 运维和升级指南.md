# OpenClaw 运维和升级指南

OpenClaw 是一个可以自托管的个人 AI 助手网关。安装成功只是第一步，真正长期使用时，更重要的是把模型凭证、聊天通道、Gateway 服务、日志排查和版本升级这些运维动作梳理成固定流程。本文基于官方文档和一次 macOS 环境下的实践记录，整理一份可复用的 OpenClaw 运维和升级 runbook。

## 一、准备工作

### 1. 准备模型凭证

安装和初始化时需要至少准备一个可用的模型提供商凭证。实践中可以按自己的使用场景选择：

- Qwen
- MiniMax
- OpenAI、Anthropic、Google 等官方文档支持的模型提供商

建议在正式安装前先确认 API Key 是否可用、余额是否充足，并记录当前使用的模型名称。OpenClaw 的 onboarding 过程会引导填写模型提供商和 API Key。

### 2. 准备聊天通道

通道不是安装的硬性前置条件。如果暂时不配置通道，也可以先通过本机 Control UI 与 OpenClaw 交流。

常见通道包括：

- WhatsApp
- Telegram
- Discord
- 飞书
- Slack / Teams 等其他聊天应用

如果要让手机或团队聊天工具直接触发 OpenClaw，需要在 Gateway 运行后再执行通道登录或安装对应的 channel gateway。

### 3. 推荐运行环境

官方文档建议使用 Node.js 环境，当前推荐 Node 24，Node 22.14+ 也受支持。macOS 下通过 npm、pnpm 或官方安装脚本安装相对简单，适合个人和小团队先跑通。

安装前可以先检查：

```bash
node --version
npm --version
```

## 二、安装 OpenClaw

### 1. 安装 CLI

如果已经自行管理 Node 环境，可以直接使用 npm 安装：

```bash
npm install -g openclaw@latest
```

如果使用 pnpm，也可以执行：

```bash
pnpm add -g openclaw@latest
pnpm approve-builds -g
```

pnpm 首次全局安装后建议执行 `pnpm approve-builds -g`，避免带构建脚本的依赖未被授权。

### 2. 初始化并安装后台服务

安装完成后执行 onboarding：

```bash
openclaw onboard --install-daemon
```

这个过程会完成模型配置、Gateway 配置和后台服务安装。macOS 上通常会安装为 LaunchAgent，Linux/WSL2 上通常会安装为 systemd 用户服务。

### 3. 验证 Gateway 状态

初始化完成后先做基础检查：

```bash
openclaw --version
openclaw doctor
openclaw gateway status
```

如果 Gateway 已经正常运行，默认会监听 `18789` 端口。可以通过浏览器打开：

```text
http://127.0.0.1:18789/
```

也可以使用：

```bash
openclaw dashboard
```

打开 Control UI。在 Control UI 中可以进行聊天、会话管理、配置检查和部分更新操作。

### 4. 配对 WhatsApp 并启动 Gateway

如果要接入 WhatsApp，可以先登录通道：

```bash
openclaw channels login
```

按照命令行提示扫描二维码或输入凭证完成配对。然后启动 Gateway：

```bash
openclaw gateway --port 18789
```

如果 Gateway 已经作为后台服务运行，日常更推荐使用服务管理命令，而不是重复启动多个前台进程。

## 三、版本升级流程

OpenClaw 迭代较快，升级建议按“更新、检查、重启、验证”的闭环执行。

### 1. 推荐方式：openclaw update

官方文档当前推荐优先使用：

```bash
openclaw update
```

它会识别当前安装方式，拉取新版本，执行 `openclaw doctor`，并重启 Gateway。升级前如果想先预览动作，可以使用：

```bash
openclaw update --dry-run
```

如果要切换到 beta channel：

```bash
openclaw update --channel beta
```

### 2. 手动升级全局包

如果当前是 npm 全局安装，可以执行：

```bash
npm install -g openclaw@latest
openclaw doctor
openclaw gateway restart
openclaw health
```

如果当前是 pnpm 全局安装，可以执行：

```bash
pnpm add -g openclaw@latest
pnpm approve-builds -g
openclaw doctor
openclaw gateway restart
openclaw health
```

这套命令的关键不是“安装最新版”本身，而是升级后必须重新执行 doctor、重启 Gateway，并确认 health 状态。

### 3. 处理 openclaw -V 仍显示旧版本

有时升级完成后，`openclaw -V` 或 `openclaw --version` 仍显示旧版本，例如：

```bash
openclaw -V
```

如果终端仍命中旧的可执行文件缓存，可以先执行：

```bash
rehash
```

然后重新检查：

```bash
openclaw -V
```

示例输出：

```text
OpenClaw 2026.4.29 (a448042)
```

如果仍不正确，再检查全局包路径：

```bash
npm prefix -g
which openclaw
echo "$PATH"
```

确认当前 shell 优先命中的是刚升级后的全局 bin 目录。

### 4. 飞书 Gateway 升级

飞书通道如果通过独立 gateway 包安装，需要单独升级。示例命令：

```bash
npx -y @larksuite/openclaw-lark@2026.4.10 install --version 2026.4.10 --tools-version 1.0.42
```

升级后建议回到 OpenClaw 侧做一次完整检查：

```bash
openclaw doctor
openclaw gateway restart
openclaw channels status --probe
```

如果飞书侧通道仍不在线，再检查飞书应用权限、事件订阅地址、回调密钥和对应 gateway 的运行日志。

## 四、基础运维命令

### 1. Gateway 运行与服务管理

Gateway 是 OpenClaw 的核心进程，负责消息路由、通道连接、Control UI 和 WebSocket 通信。

常用命令：

```bash
openclaw gateway status
openclaw gateway status --deep
openclaw gateway install
openclaw gateway start
openclaw gateway stop
openclaw gateway restart
openclaw gateway uninstall
```

如果只是本地临时运行：

```bash
openclaw gateway --port 18789
```

如果端口被占用，日志中可能出现 `EADDRINUSE` 或 `another gateway instance is already listening`。此时先检查是否已经有后台服务在运行：

```bash
openclaw gateway status --deep
```

多数个人场景一台机器只需要一个 Gateway。除非明确要做隔离或应急实例，不建议同时启动多个 Gateway。

### 2. Control UI

Gateway 默认同时提供 Control UI：

```text
http://127.0.0.1:18789/
```

Control UI 可以用于：

- 本机聊天测试
- 查看 Gateway 状态
- 管理会话
- 检查模型和工具状态
- 执行部分更新和重启操作

如果页面无法打开，优先检查：

```bash
openclaw gateway status
openclaw logs --follow
openclaw doctor
```

### 3. 通道管理

登录或配对通道：

```bash
openclaw channels login
```

查看通道探测状态：

```bash
openclaw channels status --probe
```

通道异常时，建议先确认 Gateway 是否正常，再检查具体通道凭证。对于 WhatsApp 这类扫码登录通道，设备解绑、会话过期、网络变化都可能导致需要重新配对。

### 4. 配置文件

OpenClaw 的默认配置文件通常位于：

```text
~/.openclaw/openclaw.json
```

常见配置包括：

- Gateway 端口
- 认证 token 或 password
- 模型提供商与模型配置
- 允许访问的号码或群组规则
- `allowFrom`、`requireMention` 等消息过滤策略

修改配置后通常需要重启 Gateway：

```bash
openclaw gateway restart
```

如果使用了 SecretRef 或密钥管理能力，更新密钥后可以尝试：

```bash
openclaw secrets reload
```

## 五、日志与故障排查

### 1. 查看日志

最常用的日志命令：

```bash
openclaw logs --follow
```

如果是前台运行 `openclaw gateway`，也可以直接在当前终端查看实时输出。

后台服务日志位置和系统有关：

- macOS：默认在 `~/.openclaw/logs/gateway.log` 和 `~/.openclaw/logs/gateway.err.log`
- Linux：可以通过 `journalctl --user -u openclaw-gateway.service -n 200 --no-pager` 查看
- Windows：可查看对应 Scheduled Task 状态

### 2. Gateway 无法启动

建议按以下顺序排查：

```bash
openclaw status --all
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

常见原因：

- 端口 `18789` 被占用
- 配置文件缺失或字段不兼容
- Gateway 绑定到非本地地址但没有配置认证
- 模型 API Key 无效
- 通道登录态失效
- 服务入口仍指向旧版本 OpenClaw

### 3. 远程访问注意事项

如果只在本机使用，保持默认 loopback 访问即可。如果要从外部访问 Gateway，建议通过 Tailscale、SSH 隧道或反向代理暴露，并配置 token/password 认证。

不要在没有认证的情况下把 Gateway 直接暴露到公网。OpenClaw 能调用模型和工具，一旦被误用，风险不仅是聊天内容泄露，还可能产生实际成本或执行非预期操作。

## 六、常用运维场景

### 场景一：修改配置后重启服务

1. 编辑 `~/.openclaw/openclaw.json`。
2. 保存配置。
3. 执行：

```bash
openclaw doctor
openclaw gateway restart
openclaw gateway status
```

4. 打开 Control UI 或聊天通道验证。

### 场景二：添加新的聊天通道

1. 确认 Gateway 正常运行：

```bash
openclaw gateway status
```

2. 登录通道：

```bash
openclaw channels login
```

3. 按提示完成扫码或凭证录入。
4. 检查通道状态：

```bash
openclaw channels status --probe
```

5. 在 Control UI 或对应聊天工具中发送测试消息。

### 场景三：升级后出现异常，回滚到指定版本

如果 npm 全局安装后需要回滚：

```bash
npm install -g openclaw@<version>
openclaw doctor
openclaw gateway restart
openclaw health
```

如果 pnpm 全局安装后需要回滚：

```bash
pnpm add -g openclaw@<version>
pnpm approve-builds -g
openclaw doctor
openclaw gateway restart
openclaw health
```

回滚后仍需确认 Gateway 服务入口是否已经指向当前版本。如果不确定，可以重新执行：

```bash
openclaw gateway install --force
openclaw gateway restart
```

## 七、个人实践建议

1. 安装方式固定下来。不要一会儿 npm、一会儿 pnpm、一会儿源码安装，否则排查版本路径会变复杂。
2. 每次升级都记录升级前版本、升级后版本和 Gateway 状态。
3. 配置文件修改前先备份一份，特别是通道和模型凭证较多时。
4. Gateway 服务优先用 `openclaw gateway restart` 管理，不要直接杀进程。
5. 远程访问必须配置认证，并优先走 Tailscale 或 SSH 隧道。
6. 飞书、WhatsApp 这类通道要把 OpenClaw 主程序和通道 gateway 分开看，主程序升级不一定等于通道组件也升级。

## 参考资料

- [OpenClaw 官方文档](https://docs.openclaw.ai/)
- [OpenClaw Install](https://docs.openclaw.ai/install)
- [OpenClaw Updating](https://docs.openclaw.ai/install/updating)
- [OpenClaw Gateway Runbook](https://docs.openclaw.ai/gateway/index)
- [OpenClaw Control UI](https://docs.openclaw.ai/control-ui)
