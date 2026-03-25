# Midscene: AI 驱动的 UI 自动化测试

> 让 AI 成为浏览器操作员，用自然语言编写测试脚本

---

## 什么是 Midscene.js？

Midscene.js 是由字节跳动 Web Infra 团队开源的一款 **AI 驱动的 UI 自动化测试工具**。它的核心理念是"让 AI 成为浏览器操作员"，通过多模态大语言模型（LLM）理解界面内容，允许用户使用自然语言编写测试脚本，从而大幅降低自动化测试的门槛和维护成本。

---

## 核心特性

| 特性 | 说明 |
|------|------|
| **自然语言驱动** | 用中文或英文描述操作步骤（如"点击登录按钮"），无需复杂的 CSS/XPath 定位 |
| **跨平台支持** | 支持 Web (Chrome/Edge)、Android (ADB)、iOS (WebDriverAgent) 甚至桌面应用 |
| **多种脚本格式** | 支持 JavaScript SDK、YAML 脚本以及 Chrome 插件录制 |
| **可视化调试** | 提供执行报告、高亮显示操作元素，支持 Playground 交互式调试 |

---

## 主要文档与资源

- 🌐 官方文档: [https://midscenejs.com](https://midscenejs.com/)（支持中文：[https://midscenejs.com/zh](https://midscenejs.com/zh)）
- 📦 GitHub 仓库: [https://github.com/web-infra-dev/midscene](https://github.com/web-infra-dev/midscene)

---

## 快速上手

Midscene.js 提供了多种使用方式，最常用的是 **Chrome 插件模式**（适合快速验证和录制）和 **YAML 脚本格式**（适合自动化测试工程）。

### 方式一：Chrome 插件

这是最直观的上手方式，适合录制脚本和调试。

#### 1. 安装插件

- 访问 Chrome 网上应用店搜索 "Midscene" 安装
- 或者从 GitHub Releases 下载 `.zip` 包，在 Chrome 扩展管理页面 (`chrome://extensions/`) 开启"开发者模式"后加载解压后的文件夹

#### 2. 配置模型

- 点击插件图标，进入设置页面
- 配置你的 LLM API Key（支持 OpenAI GPT-4o, Qwen-VL, 或本地部署的模型如 UI-TARS）
- ⚠️ 注意：需确保模型具备视觉识别能力（Multimodal）

#### 3. 录制与执行

- 打开目标网页
- 在插件面板中输入自然语言指令，例如："搜索'iPhone 15'，然后点击第一个商品"
- 点击运行，AI 会自动规划步骤、定位元素并执行操作
- 支持将录制的操作导出为 YAML 或 JS 脚本

### 方式二：YAML 脚本

Midscene 定义了一种 YAML 格式的脚本，方便开发者快速编写自动化脚本，并提供了对应的命令行工具来快速执行这些脚本。

#### 示例脚本

```yaml
web:
  url: https://www.bing.com

tasks:
  - name: 搜索天气
    flow:
      - ai: 搜索 "今日天气"
      - sleep: 3000
      - aiAssert: 结果显示天气信息
```

#### 执行命令

```bash
midscene ./bing-search.yaml
```

命令行会输出执行进度，并在完成后生成可视化报告。整个运行过程大幅简化了开发者做环境配置的复杂度。

> 📖 参考文档：https://midscenejs.com/zh/yaml-script-runner.html

---

## 综合运用：录制 + YAML 混合工作流

推荐的最佳实践是 **用录制的方式辅助编写 YAML 脚本**：

1. 使用 Chrome 插件录制操作步骤
2. 把对应的步骤拷贝到本地 YAML 文件
3. 调试并优化测试用例

---

## 自动化测试中的迭代优化

优化的方向主要有以下几种：

1. **公共配置和敏感数据提取到环境变量**
2. **JSON 数据驱动的批量表单素材添加**
3. **不同类型的表单有差别，实现类型的 YAML 路由**

---

## Token 消耗量分析

> 录制脚本和编写 YAML 脚本调试时都需要消耗 token；每次产品发布前，用于自动化全面回归测试也需要消耗 token。

### 示例分析：资产添加用例

以一个资产添加的用例为例，分析完整的操作流程：

**测试场景：**
- 登录系统
- 找到页面
- 增加主机

**完整 YAML 脚本：**

```yaml
web:
  url: "${TEST_BASE_URL}/web-app/app-admin/login"
  viewportWidth: 1280
  viewportHeight: 960
  acceptIncertCerts: true

tasks:
  - name: "Login to Disaster Recovery Platform"
    flow:
      - aiInput: '${TEST_USERNAME}'
        locate: "Account input field with placeholder '请输入用户名'"
      - aiInput: '${TEST_PASSWORD}'
        locate: "Password input field with label '密码'"
      - aiTap: "Login button at the bottom of the form"
      - aiTap: "Login button with text '登录'"
      - aiWaitFor: "页面跳转到主控制台或仪表盘，显示导航菜单或欢迎信息"
      - aiAssert: "User is logged in and dashboard or main page is visible"

  - name: "Navigate to Disaster Recovery Assets and Open Host Creation Form"
    flow:
      - aiTap: "Top navigation bar 'Assets' tab"
      - aiTap: "Left sidebar menu item with icon and text 'Disaster Recovery Assets'"
      - aiTap: "Tab link with text 'Host: 130' in asset statistics bar"
      - aiTap: "Button with text 'Add Asset' in disaster recovery assets section"
      - sleep: 1000

  - name: "Fill Host Asset Basic Information"
    flow:
      - aiInput: '192.168.110.110'
        locate: "Asset Name input field at the top of the create host asset form"
      - aiTap: "Radio button option with text 'Linux' in the Operating System field"
      - aiInput: '192.168.110.110'
        locate: "Host IP input field with label 'Host IP' and placeholder 'Please enter host IP'"
      - sleep: 500

  - name: "Configure Authentication Credentials"
    flow:
      - aiInput: '22'
        locate: "Connection Port input field in Authentication Credential 1 area with label 'Connection Port'"
      - aiInput: 'root'
        locate: "Username input field in Authentication Credential 1 area with label 'Username' and placeholder 'Please enter username'"
      - aiInput: "${HOST_PASSWORD}"
        locate: "Password input field with label 'Password' and placeholder 'Please enter host password'"
      - sleep: 500

  - name: "Select Business System and Data Center"
    flow:
      - aiTap: "Business System dropdown selector with label 'Business System' and placeholder 'Please select business system'"
      - aiTap: "Dropdown option with text 'Default Business System' in business system selector"
      - aiTap: "Data Center dropdown selector with label 'Data Center' and placeholder 'Please select data center'"
      - aiTap: "Dropdown option 'Production Center' in the expanded list of 'Data Center' selector"
      - sleep: 500

  - name: "Submit Host Asset Creation"
    flow:
      - aiTap: "Submit button with text 'Confirm' in bottom right corner"
      - sleep: 2000
      - aiAssert: "Host asset creation form is closed and asset list is displayed"
```

### Token 消耗统计

| 指标 | 数值 |
|------|------|
| **采用模型** | qwen3.5-flash |
| **总消耗** | ~561 千 Tokens |
| **输入 Tokens** | 550.3 千 Tokens |
| **输出 Tokens** | 10.7 千 Tokens |

### 成本估算

| 模型 | 输入价格 | 输出价格 |
|------|---------|---------|
| qwen3.5-flash | 0.0002 元/千 tokens | 0.002 元/千 tokens |
| qwen3.5-Plus | 0.0008 元/千 tokens | 0.008 元/千 tokens |

**执行成本：**
- qwen3.5-flash：约 **0.13 元**
- qwen3.5-Plus：约 **0.53 元**

**规模化预估：**
- 实际达到基本可用的规模时，用例数量按当前的 100 倍估算
- 总成本约 **13 元**（qwen3.5-flash）或 **53 元**（qwen3.5-Plus）

---

## 其他大模型价格对比

### 阿里云通义千问

| 模型 | 输入价格 | 输出价格 |
|------|---------|---------|
| qwen3.5-flash | 0.2 元/百万 tokens | 2 元/百万 tokens |
| qwen3.5-Plus | 0.8 元/百万 tokens | 8 元/百万 tokens |

### 火山引擎豆包

| 模型 | 输入长度 | 输入价格 | 缓存存储 | 缓存输入 | 输出价格 |
|------|---------|---------|---------|---------|---------|
| doubao-seed-2.0-lite | [0, 32] KB | 0.6 元/百万 | 0.017 元/百万/小时 | 0.12 元/百万 | 3.6 元/百万 |
| doubao-seed-2.0-lite | (32, 128] KB | 0.9 元/百万 | 0.017 元/百万/小时 | 0.18 元/百万 | 5.4 元/百万 |
| doubao-seed-2.0-lite | (128, 256] KB | 1.8 元/百万 | 0.017 元/百万/小时 | 0.36 元/百万 | 10.8 元/百万 |

> doubao-seed-1.6-lite 价格比 doubao-seed-2.0-lite 低一半

### DeepSeek

| 模型 | 输入（缓存未命中） | 输出 |
|------|------------------|------|
| deepseek-chat | 2 元/百万 tokens | 3 元/百万 tokens |

---

## 总结

Midscene.js 通过 AI 驱动的自然语言交互，大幅简化了 UI 自动化测试的编写和维护工作。结合 YAML 脚本格式和 Chrome 插件录制功能，开发者可以快速构建、调试和迭代自动化测试用例。

虽然基于 LLM 的执行会产生一定的 token 消耗，但考虑到其大幅提升的开发效率和降低的维护成本，对于需要频繁进行 UI 测试的项目来说，是一个值得考虑的选择。
