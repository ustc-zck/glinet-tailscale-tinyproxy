pyenv: cannot rehash: /Users/bytedance/.pyenv/shims isn't writable
# GL.iNet Tailscale Tinyproxy Skill

一个可复用的 Codex Skill，用于把 GL.iNet / OpenWrt 路由器配置成仅通过 Tailscale 访问的 HTTP/HTTPS 代理，让 Claude Code 等指定应用使用远端路由器的公网出口，同时不改变设备的其他网络流量。

## 能力

- 在 GL.iNet 路由器上启用并绑定 Tailscale
- 安装 Tinyproxy、LuCI 及相关界面包
- 用 Tinyproxy ACL 和 OpenWrt 防火墙限制访问范围
- 保持 Tailscale 出口节点、子网路由和 WAN 端口转发关闭
- 为 Claude Code 提供单进程代理命令
- 排查 LuCI、`rpcd`、Tinyproxy 启动和 tailnet 连通问题

## 安装

克隆到 Codex 个人技能目录：

```bash
git clone https://github.com/ustc-zck/glinet-tailscale-tinyproxy.git \
  ~/.codex/skills/glinet-tailscale-tinyproxy
```

也可以下载仓库后，将整个目录复制到 `~/.codex/skills/glinet-tailscale-tinyproxy`。

## 使用

在 Codex 中调用：

```text
$glinet-tailscale-tinyproxy
```

示例请求：

```text
使用 $glinet-tailscale-tinyproxy 帮我把这台 GL.iNet 路由器配置成仅限 Tailscale 使用的 Claude Code HTTP 代理。
```

## 安全模型

- 不在公网开放 Tinyproxy 端口
- 默认以 tailnet 成员身份和 Tailscale ACL 作为认证边界
- Tinyproxy 仅允许 Tailscale 地址或明确指定的 peer
- 不默认启用 exit node、子网通告或 IP 伪装
- 客户端使用进程级 `HTTP_PROXY` / `HTTPS_PROXY`，避免影响其他应用

## 文件

- `SKILL.md`：技能入口、约束和决策流程
- `references/router-workflow.md`：路由器 UI 操作、客户端命令和故障排查
- `agents/openai.yaml`：Codex 技能展示信息

## 验证

使用 Codex 内置的 skill validator：

```bash
python3 ~/.codex/skills/.system/skill-creator/scripts/quick_validate.py .
```
