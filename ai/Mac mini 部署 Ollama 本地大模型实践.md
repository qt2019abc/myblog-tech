# Mac mini 部署 Ollama 本地大模型实践

Mac mini 搭配 Apple Silicon 芯片和统一内存，已经可以作为一台轻量级本地大模型推理服务器使用。对于个人知识库、局域网内工具调用、开发测试和离线模型体验来说，Ollama 是一条上手成本很低的路线：安装简单、模型管理直接，并且在 Apple Silicon 上可以自动使用 Metal 做 GPU 加速。

本文基于一次 Mac mini 部署 Ollama 的实践记录整理，覆盖环境准备、安装启动、模型下载、GPU 验证、性能测试、局域网 API 开放、安全配置和常见排查。文中的用户名、内网 IP、SSH 地址、本地路径和访问地址均已脱敏或泛化。

## 一、环境规划

本次实践使用的是一台 Apple Silicon Mac mini，适合作为局域网内的小型推理节点。

示例环境如下：

| 项目 | 示例配置 |
| --- | --- |
| 设备 | Mac mini |
| 芯片 | Apple M 系列 |
| GPU | Apple 集成 GPU，支持 Metal |
| 内存 | 16GB 统一内存 |
| 管理用户 | `<mac-user>` |
| 局域网地址 | `<mac-mini-ip>` |
| SSH 地址 | `<mac-user>@<mac-mini-ip>` |
| Ollama 默认端口 | `11434` |
| Apple Silicon Homebrew 路径 | `/opt/homebrew` |
| 模型目录 | `/Users/<mac-user>/.ollama/models` |

16GB 统一内存比较适合运行 4B 到 9B 级别的量化模型。12B 或 14B 的 4-bit 模型可以尝试，但速度和内存余量会明显下降；20B 以上模型通常不适合长期作为日常服务运行。

这里要特别注意：统一内存由 CPU 和 GPU 共同使用，并不是额外显存。因此模型大小、上下文长度、并发数量和系统后台应用都会影响稳定性。

## 二、基础准备

### 1. 固定局域网地址

如果 Mac mini 要作为局域网推理服务，建议先在路由器或 DHCP 服务中绑定固定地址。

常见做法是：

```text
将 Mac mini 当前联网网卡的 MAC 地址
绑定到固定内网 IP，例如 <mac-mini-ip>
```

这样可以避免重启、租约更新或网络切换后 API 地址变化。

### 2. 开启 SSH

在 macOS 中打开：

```text
系统设置 -> 通用 -> 共享 -> 远程登录
```

确认允许指定管理用户访问，然后从管理电脑测试：

```bash
ssh <mac-user>@<mac-mini-ip>
```

如果希望 Mac mini 长期作为服务节点运行，建议在系统节能设置中开启“唤醒以供网络访问”，并避免频繁进入深度休眠。

## 三、安装 Homebrew

如果尚未安装 Homebrew，可以在 Mac mini 上执行：

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Apple Silicon 默认安装在 `/opt/homebrew`。安装后配置 shell 环境：

```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
brew --version
```

如果 `brew --version` 能正常输出版本信息，说明 Homebrew 已可用。

## 四、安装并启动 Ollama

安装 Ollama：

```bash
brew install ollama
```

使用 Homebrew services 启动：

```bash
brew services start ollama
```

检查服务状态：

```bash
brew services info ollama
```

检查本机 API：

```bash
curl http://127.0.0.1:11434/api/version
```

查看日志：

```bash
tail -f /opt/homebrew/var/log/ollama.log
```

常用服务管理命令：

```bash
brew services start ollama
brew services stop ollama
brew services restart ollama
```

## 五、下载和运行模型

建议先选择 7B 到 9B 左右的模型验证链路。以 `qwen3.5:9b` 为例：

```bash
ollama pull qwen3.5:9b
ollama run qwen3.5:9b
```

常用模型管理命令：

```bash
ollama list
ollama ps
ollama stop qwen3.5:9b
ollama rm qwen3.5:9b
```

模型默认保存在：

```text
/Users/<mac-user>/.ollama/models
```

查看模型磁盘占用：

```bash
du -sh ~/.ollama/models
```

模型下载期间，`blobs` 目录中可能出现带有 `partial` 后缀的临时分片。下载完成后模型才会出现在 `ollama list` 中，不建议手动删除正在下载的分片。

## 六、验证 GPU 加速

Ollama 在 Apple Silicon 上会自动通过 Metal 使用 GPU，不需要安装 CUDA。

保持模型正在运行，在另一个终端执行：

```bash
ollama ps
```

如果 `PROCESSOR` 一栏显示类似：

```text
100% GPU
```

说明模型推理主要运行在 GPU 上。

如果发现频繁使用交换空间、系统响应变慢或模型被反复卸载，通常需要降低模型规格、减少并发，或缩短上下文长度。16GB 机器上建议同时只加载一个 9B 左右模型，并将上下文控制在 4K 到 8K 范围内。

## 七、基本性能测试

本次测试使用局域网远程调用 `/v1/chat/completions` 接口，模型为 9B 级别量化模型，上下文长度为 4096 tokens。

一组典型结果如下：

| 指标 | 示例结果 | 说明 |
| --- | ---: | --- |
| 模型启动时间 | 约 5 秒 | 模型首次加载并启动推理 |
| 输入 Token 数 | 约 160 | 本次请求提示词长度 |
| 输入处理速度 | 约 180 tokens/s | 提示词预处理速度 |
| 输出 Token 数 | 约 1300 | 本次生成较长回答 |
| 输出生成速度 | 约 18 tokens/s | 每个输出 Token 约几十毫秒 |
| 纯推理总耗时 | 约 75 秒 | 输入处理和输出生成合计 |
| API 总耗时 | 约 80 秒 | 包含模型启动及接口开销 |
| HTTP 状态 | 200 | 远程接口调用成功 |

这组结果说明：当输出较长时，请求总耗时主要由输出 Token 数决定，并不一定是服务异常。

可以粗略估算：

```text
输出耗时 = 输出 Token 数 / 输出生成速度
```

例如：

```text
1300 tokens / 18 tokens/s = 约 72 秒
```

如果模型保持驻留，后续请求可以省去首次加载的几秒钟。

常见输出长度对应的生成耗时大致如下：

| 最大输出长度 | 预计生成耗时 |
| ---: | ---: |
| 300 tokens | 约 17 秒 |
| 512 tokens | 约 29 秒 |
| 1000 tokens | 约 57 秒 |
| 1300 tokens | 约 72 秒 |

交互应用建议启用流式输出，并限制最大输出长度，改善用户感知延迟。

OpenAI 兼容接口示例：

```json
{
  "model": "qwen3.5:9b",
  "messages": [
    {
      "role": "user",
      "content": "你的问题"
    }
  ],
  "stream": true,
  "max_tokens": 512
}
```

终端中可以使用 verbose 模式查看单次推理耗时：

```bash
ollama run qwen3.5:9b --verbose
```

服务端日志：

```bash
tail -f /opt/homebrew/var/log/ollama.log
```

## 八、开放局域网 API

Ollama 默认只监听本机地址：

```text
127.0.0.1:11434
```

如果需要让局域网其他服务器直接调用，可以设置：

```text
OLLAMA_HOST=0.0.0.0:11434
```

当前服务由 Homebrew 管理时，LaunchAgent 配置通常位于：

```text
/Users/<mac-user>/Library/LaunchAgents/homebrew.mxcl.ollama.plist
```

首次添加配置：

```bash
plist=~/Library/LaunchAgents/homebrew.mxcl.ollama.plist

/usr/libexec/PlistBuddy \
  -c "Add :EnvironmentVariables:OLLAMA_HOST string 0.0.0.0:11434" \
  "$plist"
```

如果配置项已经存在，改用：

```bash
/usr/libexec/PlistBuddy \
  -c "Set :EnvironmentVariables:OLLAMA_HOST 0.0.0.0:11434" \
  "$plist"
```

重新加载服务：

```bash
launchctl bootout gui/$(id -u) "$plist"
launchctl bootstrap gui/$(id -u) "$plist"
```

确认监听地址：

```bash
lsof -nP -iTCP:11434 -sTCP:LISTEN
```

预期监听地址由：

```text
127.0.0.1:11434
```

变为：

```text
*:11434
```

Homebrew 升级或重新生成服务文件后，应再次检查 `OLLAMA_HOST` 是否保留。

## 九、远程调用测试

从局域网其他服务器测试模型列表：

```bash
curl http://<mac-mini-ip>:11434/api/tags
```

调用聊天接口：

```bash
curl http://<mac-mini-ip>:11434/api/chat \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "qwen3.5:9b",
    "messages": [
      {
        "role": "user",
        "content": "你好"
      }
    ],
    "stream": false
  }'
```

OpenAI 兼容接口基础地址：

```text
http://<mac-mini-ip>:11434/v1
```

如果客户端支持 OpenAI compatible API，可以把 Base URL 设置为上述地址，并将模型名填写为 `ollama list` 中看到的名称。


## 十、新增OLLAMA配置参数

例如：添加环境变量 OLLAMA_KEEP_ALIVE:24h

使用的是 Homebrew 的 launchd 服务，直接在该 plist 的 `EnvironmentVariables` 下添加即可。

```
PLIST=/Users/tom/Library/LaunchAgents/homebrew.mxcl.ollama.plist

cp "$PLIST" "$PLIST.bak"

/usr/libexec/PlistBuddy \
  -c 'Add :EnvironmentVariables:OLLAMA_KEEP_ALIVE string 24h' \
  "$PLIST"

plutil -lint "$PLIST"
```

若该键已存在而需更新，改用：

```
/usr/libexec/PlistBuddy \
  -c 'Set :EnvironmentVariables:OLLAMA_KEEP_ALIVE 24h' \
  /Users/tom/Library/LaunchAgents/homebrew.mxcl.ollama.plist
```

重新加载服务使配置生效：

```
launchctl bootout gui/$(id -u) "$PLIST"
launchctl bootstrap gui/$(id -u) "$PLIST"
```

验证 launchd job 已读到变量：

```
launchctl print gui/$(id -u)/homebrew.mxcl.ollama \
  | grep -A 8 -B 2 OLLAMA_KEEP_ALIVE
```

再预热并检查模型保活时间：

```
curl -s http://127.0.0.1:11434/api/generate \
  -d '{"model":"qwen3.5:9b","keep_alive":"24h"}' >/dev/null

ollama ps
```

`UNTIL` 应显示约 `24 hours from now`，而非 4 分钟。

注意：Homebrew 升级、重装或再次 `brew services restart ollama` 时，可能重建 `homebrew.mxcl.ollama.plist` 并覆盖手工改动；届时需检查并重新加入该环境变量。
## 十一、安全配置

Ollama API 默认没有账号、Token 或访问控制。设置 `OLLAMA_HOST=0.0.0.0:11434` 后，所有能访问该地址的局域网设备都可能调用模型。

如果只有少数服务器需要调用，更推荐保留 Ollama 默认本机监听，并在调用服务器上建立 SSH 隧道：

```bash
ssh -N -L 11434:127.0.0.1:11434 <mac-user>@<mac-mini-ip>
```

建立隧道后，调用服务器通过本机地址访问：

```text
http://127.0.0.1:11434
```

如果必须直接开放局域网端口，建议额外部署带 IP 白名单或身份认证的反向代理。需要特别注意：`OLLAMA_ORIGINS` 只控制浏览器跨域访问，不提供服务端身份认证。

最低限度建议：

- 只在可信内网开放 `11434/TCP`。
- 不要直接暴露到公网。
- 通过防火墙或反向代理限制来源 IP。
- 不要把 SSH 用户、真实 IP、反向代理账号密码写入公开文档。
- 对多人共用场景增加认证、审计和调用配额。

## 十二、运维与故障排查

检查服务：

```bash
brew services info ollama
pgrep -alf ollama
lsof -nP -iTCP:11434 -sTCP:LISTEN
```

检查模型和 GPU：

```bash
ollama list
ollama ps
```

检查日志：

```bash
tail -n 200 /opt/homebrew/var/log/ollama.log
```

接口无法访问时，建议按顺序确认：

1. `<mac-mini-ip>` 是否仍是 Mac mini 当前地址。
2. Ollama 服务是否处于运行状态。
3. `11434` 是否监听在 `*:11434`，而不是仅监听 `127.0.0.1`。
4. 调用服务器与 Mac mini 之间网络是否可达。
5. macOS 防火墙、路由器或 VLAN 策略是否拦截 `11434/TCP`。
6. 请求使用的模型是否已经下载完成，并出现在 `ollama list` 中。
7. 当前模型是否因为内存压力被卸载或重启。

如果请求很慢，可以继续检查：

- 输出 Token 是否过多。
- 是否未启用流式输出。
- 模型是否每次都重新加载。
- 上下文长度是否过大。
- 是否有多个客户端并发请求。
- Mac mini 是否出现内存压力或大量 swap。

## 十三、部署验收清单

完成部署后建议逐项验收：

- `ssh <mac-user>@<mac-mini-ip>` 可以正常登录。
- `brew services info ollama` 显示服务正在运行。
- `ollama list` 能看到已下载模型。
- `ollama run qwen3.5:9b` 可以正常生成回答。
- `ollama ps` 显示模型使用 GPU。
- 获准的远程服务器可以访问 `/api/tags`。
- OpenAI 兼容接口 `/v1` 可被客户端调用。
- Mac mini 重启后 Ollama 服务能够自动恢复。
- API 地址、模型名称和安全访问方式已经记录在内部文档中。

## 十四、总结

Mac mini 部署 Ollama 的关键不在安装本身，而在几个实践细节：固定内网地址、确认 Metal GPU 加速、控制模型大小和上下文长度、正确开放局域网 API，并补上访问控制。

对于 16GB 统一内存的 Mac mini，9B 左右量化模型是比较平衡的选择。它可以承担个人和小团队的本地推理、工具验证和局域网 API 服务；如果要面向多人或高并发使用，就需要进一步引入队列、限流、反向代理和更明确的模型容量规划。小机器也能干正经活，只是要让它在合适的负载里发光。
