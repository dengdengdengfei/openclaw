# Moltbot 启动与排障笔记（本次踩坑汇总）

> 适用场景：macOS 上跑 Gateway + Control UI，Windows 通过 SSH/Tailscale 远程访问；并启用 Telegram channel。
>
> 这份文档是“能启动 + 遇到问题怎么查/怎么修”的操作手册，重点记录这次实际遇到的坑。

## 0. 重要安全提醒（一定要看）

- **不要把任何 token / API key / bot token 写进仓库、截图、日志或聊天记录。**
- Control UI / Gateway 的 token 相当于“管理员密码”。如果要公网/局域网暴露端口，务必启用鉴权，并限制来源。

## 1. 当前结论（我们这次最终跑通的形态）

- Gateway **非 dev** 跑在 `127.0.0.1:18789`（loopback only）。
- Windows 访问 Control UI 采用 **SSH 端口转发**：把远端的 loopback 端口映射到本机，再在浏览器打开本机地址。
- Telegram 之所以 `fetch failed`，根因是网关主机无法直连 `api.telegram.org`；**仅设置 shell 的 `http_proxy/https_proxy` 不一定能让 Node/undici 生效**（取决于实现/版本/库用法）。
- 解决方式是配置：`channels.telegram.proxy = "http://127.0.0.1:7890"`（示例），让 Telegram provider 明确走代理。

## 2. 从源码安装与构建

在仓库根目录（本项目为 monorepo）：

```bash
cd moltbot
pnpm install
pnpm build
pnpm ui:build
```

说明：
- `pnpm build` 产物包含 `dist/` 的后端/网关代码。
- `pnpm ui:build` 会产出 `dist/control-ui/`，Gateway 会用它提供 Web UI。

## 3. 启动 Gateway（非 dev / 正常端口 18789）

推荐用 CLI（更稳定，也更贴近文档）：

```bash
# 方式 A：直接用已安装的 CLI（推荐）
# 如果你是从源码跑，也可以用 `pnpm moltbot` 或 `node scripts/run-node.mjs` 的等价方式。

moltbot gateway run --port 18789 --bind loopback --auth token --token "<YOUR_GATEWAY_TOKEN>" --allow-unconfigured
```

等价（从源码执行）：

```bash
CLAWDBOT_GATEWAY_TOKEN="<YOUR_GATEWAY_TOKEN>" \
node scripts/run-node.mjs gateway --port 18789 --bind loopback --auth token --allow-unconfigured
```

常用选项：
- `--force`：如果端口被占用，强制杀掉占用进程再起。
- `--verbose`：更详细日志。
- `--dev`：会创建 dev 配置目录并改默认端口（适合开发，不适合你这次的“固定 18789”）。

参考文档：
- docs/cli/gateway.md

## 4. Windows 远程访问（推荐：SSH 端口转发）

### 4.1 为什么你 curl Tailscale IP 会失败？

你之前执行过类似：

```bash
curl -vkI https://100.x.x.x:18789/
```

失败的常见原因：
- Gateway 默认只 **bind 到 `127.0.0.1`**，所以从 `100.x`（tailnet IP）访问不到。
- Gateway 默认是 **HTTP/WebSocket**，不是 HTTPS；用 `https://` 访问会导致握手失败。

### 4.2 正确做法：SSH 转发

在 Windows 上：

```bash
ssh -N -L 18789:127.0.0.1:18789 <user>@<mac-host>
```

然后在 Windows 浏览器打开：

- `http://127.0.0.1:18789/`

如果 UI 需要 token：
- 直接在 UI 里填写 token，或使用 UI 提供的连接设置入口。

（如果你想走 WebSocket URL，一般是 `ws://127.0.0.1:18789`。）

## 5. 这次遇到的主要问题与处理办法（按时间/症状归类）

### 5.1 端口 18789 被占用（EADDRINUSE / 启动失败）

现象：Gateway 起不来，提示端口被占用；或 `gateway stop` 不生效。

原因：
- 有一个“手动启动的 node 进程”在占用端口，但它不是通过 `gateway install` 管理的系统服务，所以 `gateway stop` 提示类似 `service not loaded`。

处理：
- 用 `--force` 启动（推荐）：

```bash
moltbot gateway run --port 18789 --force ...
```

- 或者手动找 PID 并 kill（不推荐作为常态流程）。

### 5.2 Control UI 提示：`unauthorized: gateway token mismatch`

现象：UI 能打开，但操作/连接时提示 token mismatch。

原因：
- **服务端 token**（`gateway.auth.token` 或 `CLAWDBOT_GATEWAY_TOKEN`）与 **浏览器里缓存的 token** 不一致。
- UI 会把 token 缓存在浏览器 localStorage（所以你改了服务端 token 后，客户端可能还在用旧 token）。

处理：
- 在 UI 的连接/设置里更新 token；或清理浏览器站点数据。
- 确保你启动 gateway 时使用的 token 与 UI 输入一致。

### 5.3 配置页报错：`Unsupported schema node. Use Raw mode.`

现象：在 Control UI 的 Config 页面，Form 模式无法渲染某些配置结构（例如 `channels.telegram.accounts` 这类“动态 key 的对象”）。

处理：
- 在 Config 页面切换到 **Raw mode**，直接编辑 JSON5。

这点容易和网关的 `--raw-stream`（原始流日志）混淆：
- UI 的 Raw mode：配置编辑器的显示模式。
- Gateway 的 `--raw-stream`：把模型流事件写 jsonl，完全是另一回事。

### 5.4 Telegram 启动时报 `fetch failed` / `setMyCommands failed`

现象（日志中常见）：
- `TypeError: fetch failed`
- `[telegram] setMyCommands failed: Network request for 'setMyCommands' failed!`

根因：
- 网关主机到 `https://api.telegram.org/` 的网络不通（被墙、DNS、IPv6、公司网络策略等）。

这次有效的修复方式：
- 给 Telegram provider 配置明确的代理：

```json5
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "<YOUR_TELEGRAM_BOT_TOKEN>",
      "proxy": "http://127.0.0.1:7890"
    }
  }
}
```

说明：
- 仅靠 shell 的 `http_proxy/https_proxy/all_proxy` 不一定能解决（取决于 Node/undici 以及该 provider 是否读取 env proxy）。
- `channels.telegram.proxy` 是更“确定能生效”的路径。

自检清单：
- 代理进程是否真的在网关机上监听：`127.0.0.1:7890`。
- 用 curl 验证代理可达 Telegram：

```bash
curl -I --proxy http://127.0.0.1:7890 https://api.telegram.org/
```

参考文档：
- docs/channels/telegram.md

### 5.5 “不要跳过任何启动项”（channels 没启动）

现象：Gateway 能起来，但 Telegram/其他 provider 不会启动。

原因：
- 使用了 `CLAWDBOT_SKIP_CHANNELS=1`（常见于 dev 模式脚本），会跳过 channels 初始化。

处理：
- 启动时不要设置 `CLAWDBOT_SKIP_CHANNELS=1`。
- 或避免使用 `pnpm gateway:dev` 之类的 dev 脚本来跑“生产形态”。

## 6. 推荐的“稳定启动流程”（给以后复用）

1) 安装/构建：

```bash
pnpm install
pnpm build
pnpm ui:build
```

2) 启动（loopback + token）：

```bash
moltbot gateway run --port 18789 --bind loopback --auth token --token "<YOUR_GATEWAY_TOKEN>" --allow-unconfigured
```

3) Windows 访问（SSH 转发）：

```bash
ssh -N -L 18789:127.0.0.1:18789 <user>@<mac-host>
```

4) 浏览器打开：

- `http://127.0.0.1:18789/`

5) Telegram 需要代理时，在配置中加：

- `channels.telegram.proxy: "http://127.0.0.1:7890"`

## 7. 参考链接（仓库内文档）

- docs/cli/gateway.md
- docs/gateway/remote-gateway-readme.md
- docs/channels/telegram.md

---

如果你希望下一步把“只允许 127.0.0.1”改成“允许 tailnet + 本机 + HTTPS 域名”，建议优先走 `moltbot gateway run --bind tailnet --tailscale serve` 这一套官方路径（见 docs/cli/gateway.md），而不是直接放开监听到 `0.0.0.0`。
