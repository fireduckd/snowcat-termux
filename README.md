# SnowCatQQ on Android (Termux + Debian)

> **非官方实践记录 / Community deployment notes**  
> 在 Android 13 上，使用 Termux、PRoot Debian（ARM64）、Linux QQ、SnowCat Framework、OneBot 12 与 DeepSeek API 构建 QQ 自动回复机器人。
>
> 本仓库记录的是**特定环境中的本地部署与调试经验**，不是 SnowCat / QQ / DeepSeek 的官方安装包，也不保证不同手机、ROM、QQ 版本可以直接复现。

## 功能概览

- 在 Android 的 **Termux → PRoot Debian** 中运行 Linux QQ 和 SnowCat。
- 使用 **TigerVNC / X11** 提供 QQ 所需图形显示（默认 `DISPLAY=:1`，VNC 端口 `5901`）。
- 使用 SnowCat **WebUI** 管理服务，示例访问地址：`http://127.0.0.1:6099/webui/`。
- 使用 **OneBot 12 WebSocket 服务端**（示例端口 `3005`）连接 Node.js 机器人。
- 通过 DeepSeek Chat Completions API 生成 QQ 私聊自动回复。
- 可选：从本人聊天文本整理**语言风格档案**，让回复更贴近个人短句和口语习惯。

**架构：**

```text
Android
└── Termux
    └── PRoot Debian (ARM64)
        ├── TigerVNC / X11 (:1 / 5901)
        ├── Linux QQ 3.2.35-54395（本地测试版本）
        │   └── SnowCat Framework（本地 Linux 适配）
        │       ├── WebUI (:6099，配置可能不同)
        │       └── OneBot 12 WebSocket (:3005，配置可能不同)
        └── Node.js /root/deepseek-qq/bot.cjs
            ├── DeepSeek API
            └── 可选：style-data/style-profile.json
```

## 重要说明

1. **这里使用的是 SnowCat，不是 NapCat。** 某些历史目录、入口文件或环境变量可能仍含 `napcat` 字样；这不代表本项目改成了 NapCat。
2. 使用的是本地测试过的 **Linux QQ 3.2.35-54395 / ARM64**。版本升级后原生接口、偏移和启动行为可能变化。
3. Linux 适配器、注入入口和 Framework 可能包含针对本地环境的改动；**本文不提供未经验证的通用构建教程**。
4. 不要上传 QQ 安装包、未获授权的二进制文件、登录数据、账号数据库、敏感日志或第三方私有代码。遵循相关项目许可证和服务条款。
5. **本文出现的账号、Token、Key 均以占位符说明，不附带真实凭据。**

## 已验证的本地目录（按需调整）

| 路径 | 用途 |
|---|---|
| `/root/snowcat-test/` | SnowCat / QQ 本地运行目录 |
| `/root/snowcat-test/qq-isolated/qq` | Linux QQ 启动程序 |
| `/root/snowcat-test/framework/` | 实际部署的 Framework 构建文件 |
| `/root/snowcat-build/` | 本地源码与构建工作区 |
| `/root/snowcat-electron-home/` | QQ 独立 HOME，可能包含登录/个人数据，**不要上传** |
| `/root/deepseek-qq/bot.cjs` | DeepSeek 自动回复机器人 |
| `/root/deepseek-qq/style-data/` | 可选的风格统计与聊天文本（原始文本必须私有） |

## 运行前检查

在 **Termux** 中进入 Debian：

```bash
termux-wake-lock
proot-distro login debian
```

在 **Debian** 中确认基本组件存在：

```bash
command -v vncserver
command -v xdpyinfo
command -v node
ls -l /root/snowcat-test/qq-isolated/qq
```

上面的命令是**检查**，不是完整依赖安装器。首次部署还需要自行准备 Linux QQ、SnowCat 适配构建、图形环境和 Node.js 依赖。

### 1. 启动并验证 VNC

```bash
vncserver -list
```

如果 `:1` 尚未启动：

```bash
vncserver :1 -geometry 1280x720 -depth 24
```

验证：

```bash
DISPLAY=:1 xdpyinfo >/dev/null 2>&1 \
  && echo 'VNC 正常' || echo 'VNC 异常'
```

**仅当 VNC 正常时再启动 QQ。** 如果 `vncserver` 卡住、显示 `(stale)`，先检查 VNC 进程和日志，不要无限重复启动。

### 2. 启动 SnowCat + Linux QQ

在 **Debian** 中：

```bash
cd /root/snowcat-test

env -u ELECTRON_RUN_AS_NODE \
  -u LD_PRELOAD \
  -u LD_LIBRARY_PATH \
  DISPLAY=:1 \
  HOME=/root/snowcat-electron-home \
  NAPCAT_DISABLE_BYPASS=1 \
  ./qq-isolated/qq --no-sandbox --disable-gpu \
  > ./snowcat-clean.log 2>&1
```

这条命令会在前台持续运行。`--no-sandbox` 是本地 PRoot 环境采用的启动参数，**会降低 Electron 沙箱保护**；不应直接套用到通常的桌面 Linux 环境。

需要让它持续运行时，请保持该终端会话存活，或在明确了解 Android 后台限制的情况下使用 `tmux`/服务管理工具。`tmux` **无法阻止 Android 杀死整个 Termux 进程组**。

### 3. 检查 WebUI 和 OneBot

在另一个 **Termux 或 Debian** 终端：

```bash
curl -s -o /dev/null -w 'WebUI: %{http_code}\n' \
  http://127.0.0.1:6099/webui/

curl -s -o /dev/null -w 'OneBot: %{http_code}\n' \
  http://127.0.0.1:3005/
```

- `WebUI: 200`：WebUI HTTP 页面可访问。
- `OneBot: 426`：通常表示目标端口正在等待 WebSocket Upgrade，**不等于已完成 Token 认证或机器人能正常回复**。
- `000`：HTTP 客户端未拿到响应；优先排查服务是否启动、监听端口、网络或 QQ 初始化状态。

这套部署使用 **OneBot 12**。如果你还有其他业务占用 `3001`，可把 OneBot 12 WebSocket 配为 `3005`，并同步修改机器人连接地址。

## DeepSeek 自动回复机器人

既有机器人代码位于 `/root/deepseek-qq/bot.cjs`，Node.js 通过 `ws` 连接 SnowCat 的 **OneBot 12 WebSocket**，使用 DeepSeek Chat Completions API 生成回复。

**本 README 不包含完整 `bot.cjs` 源码或可直接运行的安装包**；请按本地实际实现检查地址、OneBot 12 事件结构和认证方式。

现有本地脚本运行前需要的环境变量：

| 环境变量 | 作用 |
|---|---|
| `DEEPSEEK_API_KEY` | DeepSeek API 密钥 |
| `ONEBOT_TOKEN` | SnowCat OneBot WebSocket 认证 Token |
| `ALLOWED_QQ` | 允许触发自动回复的 QQ 号码 |

在 **Debian** 中使用隐藏输入，避免把密钥直接写入 Shell 历史：

```bash
cd /root/deepseek-qq

IFS= read -r -s -p 'DeepSeek API Key: ' DEEPSEEK_API_KEY; echo
export DEEPSEEK_API_KEY

IFS= read -r -s -p 'OneBot Token: ' ONEBOT_TOKEN; echo
export ONEBOT_TOKEN

export ALLOWED_QQ='<允许的好友QQ号>'

node bot.cjs
```

> 新的 Shell 会话不会自动继承旧会话里手动输入的变量。不要将密钥直接写入 README、Git 提交、公开日志或命令示例。若凭据曾出现在截图或公开聊天中，应立即轮换。

### OneBot 12 注意事项

- 应按 OneBot 12 处理事件：`type: 'message'`、`detail_type: 'private'` 等，而不是直接沿用 OneBot 11 的 `post_type` / `message_type` 结构。
- 心跳 `type: 'meta'` 和输入状态通知 `type: 'notice'` 不是聊天消息，机器人应忽略。
- 如果 WebSocket 需要认证，按你的 SnowCat 配置发送 `Authorization: Bearer <ONEBOT_TOKEN>`；密钥中不要混入换行、终端提示符或 `export` 命令。
- 不要在 OneBot 端口未监听时反复启动多个机器人实例，否则会持续 `ECONNREFUSED`。

## 可选：学习自己的聊天语言风格

本地测试的数据来源为**自己发出的微信文字消息**，不是包含双方上下文的完整聊天对。测试样本为 **291 条消息**，仅用于初步统计说话习惯，不代表完成模型微调或能还原特定情境下的真实回复。

目录示例：

```text
/root/deepseek-qq/style-data/
├── dialogues.txt         # 原始整理文本，必须私有
├── my-messages.txt       # 提取出的个人消息，必须私有
├── analyze.py            # 可选，本地统计脚本
├── style-stats.json      # 本地统计结果
└── style-profile.json    # 给 DeepSeek 使用的风格规则
```

示例统计（来自本地这批样本）：

| 指标 | 值 |
|---|---:|
| 消息数量 | 291 |
| 平均长度 | 9 字符 |
| 中位长度 | 7 字符 |
| 不超过 12 字符的比例 | 81.4% |

据此可以使用简单风格规则：**优先短句、少标点、自然口语、避免客服腔**。不要机械重复口头禅，也不要仅凭个人消息推断对方的反应。

实施方式是：本地读取 `style-profile.json` 中的 `instructions`，将其作为 DeepSeek 请求的提示词之一。**这属于提示词风格适配，不是 Fine-tuning（模型微调）**。

> 原始微信聊天文本可能包含其他人的隐私。请在本地先匿名化、去除敏感内容，只使用本人有权处理的数据；不要将完整聊天历史直接提交到公开仓库或远程 API。

## Framework 构建与部署提示

在这一环境中，Framework 源码工作区与实际部署目录不同：

- 源码：`/root/snowcat-build/packages/snowcat-framework/`
- 构建输出：`/root/snowcat-build/packages/snowcat-framework/dist/`
- 运行部署：`/root/snowcat-test/framework/`

`framework/napcat.mjs` 可能只是一个约百字节的导出入口，真正代码在 `snowcat-*.js` 等分块中。因此**不能只看 `napcat.mjs` 文件大小或仅在入口里 grep** 就判断修复是否生效。

在确认本地构建工具已配置后，示例编译命令：

```bash
cd /root/snowcat-build
pnpm --filter snowcat-framework exec vite build
```

**只在构建成功并确认目标修复已进入 `dist` 后**，再考虑部署：

```bash
cp -a /root/snowcat-build/packages/snowcat-framework/dist/. \
  /root/snowcat-test/framework/
```

部署前应先停止相关进程并备份运行目录。不要随意删除旧 `snowcat-*.js`：不同入口和缓存可能仍引用它们。**源码有修改不等于运行时已经加载新代码**。

### 登录状态恢复问题

遇到过如下现象：QQ 提示“当前账号已登录，无法重复登录”，但 SnowCat 仍停留在扫码页、OneBot 未初始化。

排查发现一版 Framework 仅在 `onQRCodeLoginSucceed` 时继续初始化，而 `onUserLoggedIn` 只记录错误。针对**已登录会话恢复**曾做本地源码尝试：查询登录列表，匹配当前 `uin`/`uid` 后继续 Framework 初始化。

此修复属于**实验性适配**：要确认 QQ 原生接口的实际返回值、初始化回调时序，以及新构建产物已部署；不保证所有版本通用，不能仅靠修改 WebUI 状态冒充登录成功。

## 常见故障排查

| 现象 | 优先排查 |
|---|---|
| `Missing X server or $DISPLAY` | `DISPLAY=:1 xdpyinfo`、VNC 是否存活 |
| `vncserver -list` 显示 `(stale)` | 上次 X/VNC 进程退出，先检查实际进程，再清理失效会话 |
| `Trace/breakpoint trap` / `Segmentation fault` | QQ/Electron、VNC 或原生模块崩溃；查看对应程序日志，不要直接归咎于 JS |
| `Failed to shutdown` / `X connection error` | X11 连接是否被中断；确认 VNC 先于 QQ 启动并持续存活 |
| WebUI `200`、OneBot `000` | QQ 登录/Framework 初始化、OneBot 服务端是否启用及端口 |
| OneBot `426` | WebSocket 端口可能已监听；继续验证认证和消息收发 |
| `ECONNREFUSED 127.0.0.1:3005` | OneBot 没有在该端口接受连接；先恢复 SnowCat |
| `ERR_INVALID_CHAR` / `Authorization` | Token 含非法字符、换行或混入终端命令；重新安全输入 |
| DeepSeek 请求头值无效 | 检查 `DEEPSEEK_API_KEY` 是否被错误复制；不要打印完整密钥 |
| `[Process completed (signal 9)]` | 进程收到 `SIGKILL`；排查 Android 后台限制、内存压力和其他进程管理机制 |

### Android 13 后台进程问题

Android 12+ 的 Phantom Process Killer / 厂商省电管理**可能**清理 Termux 子进程。可根据设备情况检查：

- Termux 电池策略设置为「无限制」；允许后台运行和自启动（如果 ROM 提供）。
- 按需执行 `termux-wake-lock`，但它不能避免全部系统级进程终止。
- 必要时通过经授权的 ADB 连接检查 Android 13 的 Phantom Process 设置，例如：

```bash
adb shell settings get global settings_enable_monitor_phantom_procs
```

某些设备允许通过 ADB 修改相关配置，但**是否生效依赖 Android 版本及厂商实现**；相关修改可能影响系统资源管理和续航。`tmux` 只保留终端会话，不能对抗 Android 强制结束整个 Termux 进程组。
