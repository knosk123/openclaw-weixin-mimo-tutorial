# OpenClaw 微信接入 Claude Code / MiMo 教程

这是一份脱敏后的参考方案，用于把 OpenClaw 接入微信，并配置 MiMo 或其他 OpenAI 兼容模型服务。教程中的 API Key、服务器地址、微信 ID 都使用占位符，请替换成自己的信息。

## 适用场景

- 想在微信里和 AI 对话
- 想让 OpenClaw 使用 MiMo、GPT-5.5 代理或其他 OpenAI 兼容接口
- 想把机器人部署到服务器上，让本地电脑关机后仍然能回复

推荐架构：

```text
微信
  ↓
OpenClaw Weixin 插件
  ↓
服务器或本机上的 OpenClaw Gateway
  ↓
MiMo / OpenAI 兼容模型服务
```

## 一、准备工作

需要准备：

```text
1. 一台 Windows 电脑，或者一台长期在线的 Linux 服务器
2. Node.js 22.16 或更高版本，推荐 Node.js 24
3. OpenClaw
4. MiMo 或其他 OpenAI 兼容服务的 API Key
5. 微信 ClawBot / openclaw-weixin 插件
```

示例占位：

```text
模型服务 Base URL: https://api.example.com/v1
API Key: YOUR_API_KEY
模型名: mimo-v2.5-pro 或 gpt-5.5
```

不要把真实 API Key、服务器密码、微信 ID、二维码登录链接发到公开仓库。

## 二、安装 Node.js

先确认 Node.js 版本：

```bash
node -v
npm -v
```

OpenClaw 建议使用：

```text
Node.js >= 22.16
```

如果服务器自带 Node.js 版本太低，可以安装新版 Node.js 到用户目录，例如：

```bash
mkdir -p ~/.local
# 按你的系统架构下载 Node.js 22/24 的 Linux 包后解压到 ~/.local/node24
```

然后把 Node.js 加入 PATH：

```bash
export PATH="$HOME/.local/node24/bin:$HOME/.local/bin:$PATH"
```

## 三、安装 OpenClaw

```bash
npm install -g openclaw
```

检查安装：

```bash
openclaw --version
```

## 四、配置模型服务

OpenClaw 配置文件通常在：

Linux：

```text
~/.openclaw/openclaw.json
```

Windows：

```text
C:\Users\<你的用户名>\.openclaw\openclaw.json
```

下面是 OpenAI 兼容模型服务的示例配置：

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "openai": {
        "baseUrl": "https://api.example.com/v1",
        "apiKey": "YOUR_API_KEY",
        "auth": "api-key",
        "api": "openai-completions",
        "models": [
          {
            "id": "mimo-v2.5-pro",
            "name": "mimo-v2.5-pro",
            "contextWindow": 200000,
            "maxTokens": 32768,
            "input": ["text"]
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "openai/mimo-v2.5-pro"
      },
      "thinkingDefault": "medium"
    }
  }
}
```

如果你的服务商模型名是 `gpt-5.5`，就把模型 ID 改成：

```json
"id": "gpt-5.5",
"name": "gpt-5.5"
```

并把默认模型改成：

```json
"primary": "openai/gpt-5.5"
```

验证模型配置：

```bash
openclaw models list
openclaw models status
```

## 五、安装微信插件

安装 openclaw-weixin：

```bash
npx -y @tencent-weixin/openclaw-weixin-cli@latest install
```

检查插件：

```bash
openclaw plugins list
```

确保配置里启用了插件：

```json
{
  "plugins": {
    "enabled": true,
    "allow": [
      "openclaw-weixin",
      "openai"
    ],
    "entries": {
      "openclaw-weixin": {
        "enabled": true
      },
      "openai": {
        "enabled": true
      }
    }
  }
}
```

## 六、启动 OpenClaw Gateway

启动网关：

```bash
openclaw gateway --port 18789
```

检查状态：

```bash
openclaw gateway status
```

正常时会看到类似：

```text
Runtime: running
Connectivity probe: ok
Listening: 127.0.0.1:18789
```

## 七、微信扫码登录

运行：

```bash
openclaw channels login --channel openclaw-weixin --verbose
```

终端会显示二维码或登录链接。用微信扫码授权。

扫码完成后检查：

```bash
openclaw channels status --deep
```

正常状态类似：

```text
openclaw-weixin configured: true
account running: true
lastError: null
```

这时就可以在微信里给 ClawBot 发消息测试。

## 八、电脑关机后仍然回复

如果 OpenClaw 只运行在自己的电脑上，电脑关机后微信机器人就不会回复。

要让它一直在线，建议部署到 Linux 服务器。服务器上完成：

```text
1. 安装 Node.js 22.16+
2. 安装 OpenClaw
3. 配置模型 API
4. 安装 openclaw-weixin 插件
5. 扫码登录微信
6. 配置 OpenClaw Gateway 开机自启
```

systemd 用户服务示例：

```ini
[Unit]
Description=OpenClaw Gateway
After=network.target

[Service]
Type=simple
ExecStart=/home/<用户名>/.local/node24/bin/openclaw gateway --port 18789
Restart=always
RestartSec=5
Environment=PATH=/home/<用户名>/.local/bin:/home/<用户名>/.local/node24/bin:/usr/local/bin:/usr/bin:/bin

[Install]
WantedBy=default.target
```

保存到：

```text
~/.config/systemd/user/openclaw-gateway.service
```

启用服务：

```bash
systemctl --user daemon-reload
systemctl --user enable openclaw-gateway.service
systemctl --user start openclaw-gateway.service
```

如果没有权限启用 linger，可以用 crontab 兜底：

```bash
crontab -e
```

加入：

```cron
@reboot /home/<用户名>/.local/bin/start-openclaw-gateway.sh
* * * * * /home/<用户名>/.local/bin/start-openclaw-gateway.sh
```

`start-openclaw-gateway.sh` 示例：

```bash
#!/usr/bin/env bash
set -e
export PATH="$HOME/.local/bin:$HOME/.local/node24/bin:$PATH"

if ! ss -ltn | grep -q ':18789 '; then
  nohup openclaw gateway --port 18789 > "$HOME/.openclaw/gateway-nohup.log" 2>&1 &
fi
```

## 九、微信常用命令

查看当前模型：

```text
/model
```

设置思考深度：

```text
/think medium
```

常见选项：

```text
low
medium
high
xhigh
```

如果已开启服务器命令权限，可以在微信里执行：

```text
/bash whoami && hostname && df -h
```

也可以直接说：

```text
帮我查看服务器磁盘空间
```

## 十、可选：开启微信控制服务器

如果希望从微信里让 AI 控制服务器，可以开启命令和工具权限。但这有明显风险，建议只允许自己的微信 ID。

示例配置：

```json
{
  "commands": {
    "native": true,
    "nativeSkills": true,
    "text": true,
    "bash": true,
    "config": true,
    "mcp": true,
    "plugins": true,
    "debug": true,
    "restart": true,
    "ownerAllowFrom": [
      "你的微信用户ID"
    ],
    "allowFrom": {
      "openclaw-weixin": [
        "你的微信用户ID"
      ]
    }
  },
  "tools": {
    "allow": ["*"],
    "elevated": {
      "enabled": true,
      "allowFrom": {
        "openclaw-weixin": [
          "你的微信用户ID"
        ]
      }
    },
    "exec": {
      "host": "gateway",
      "security": "full",
      "ask": "off"
    },
    "fs": {
      "workspaceOnly": false
    }
  },
  "agents": {
    "defaults": {
      "elevatedDefault": "full",
      "thinkingDefault": "medium"
    }
  }
}
```

重要说明：

- 这通常只是服务器当前系统用户的权限，不是 root 权限
- 不建议给机器人配置免密 root / sudo
- 不要把机器人拉进陌生群
- 不要允许陌生微信 ID 使用 `/bash`

## 十一、常用排查命令

检查网关：

```bash
openclaw gateway status
```

检查微信插件：

```bash
openclaw channels status --deep
```

检查整体健康状态：

```bash
openclaw health
```

查看日志：

```bash
tail -f /tmp/openclaw/openclaw-$(date +%F).log
```

如果微信不回复，重点检查：

```text
1. Gateway 是否 running
2. openclaw-weixin 是否 configured
3. account 是否 running
4. 模型 API Key 是否有效
5. Base URL 是否写成 /v1
6. 模型名是否和服务商一致
7. 微信扫码登录是否过期或掉线
```

## 十二、Claude Code 接入 MiMo 的补充说明

如果是 Claude Code 直接接入 MiMo，需要按服务商文档配置 Claude Code 的环境变量或 settings 文件。

Windows 常见配置位置：

```text
C:\Users\<你的用户名>\.claude\settings.json
```

示例：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.example.com",
    "ANTHROPIC_AUTH_TOKEN": "你的API_KEY",
    "ANTHROPIC_MODEL": "mimo-v2.5-pro"
  }
}
```

测试：

```bash
claude --print "你好"
```

如果仍然报：

```text
Unable to connect to Anthropic services
Failed to connect to api.anthropic.com
```

通常说明 Claude Code 仍在请求 Anthropic 官方地址，配置没有生效，或者 Base URL 写错。

## 十三、安全清单

公开分享前确认不要包含：

```text
真实 API Key
服务器 IP / 域名
SSH 账号和密码
微信用户 ID
二维码登录链接
OpenClaw gateway token
本地配置文件备份
日志文件
```

如果 API Key 曾经发到聊天、截图或公开仓库，建议立即到服务商后台重置。
