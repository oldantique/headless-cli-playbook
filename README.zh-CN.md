# headless-cli-playbook — 中文快速上手

**让 Claude Code(或任何 agent harness)可靠地无头调用三个编码 CLI —— OpenAI 的 `codex`、`opencode`、Google Antigravity 的 `agy` —— 的加固手册**:stdin 死锁、静默空输出、SQLite 并发锁、输出截断,这些官方文档不会告诉你的坑,每个都配一行修法。

这不是又一个"找别的模型要第二意见"的编排器(那类工具已经很多,见英文 README 的 Prior art)。这里是没人写下来的部分:让无头调用**不挂死、不返空**的运维知识。完整内容(坑对照表、逐 CLI 配置、多轮对话与并行配方)见 [README.md](README.md)(英文)。

## 30 秒安装

```bash
git clone https://github.com/oldantique/headless-cli-playbook
mkdir -p ~/.claude/skills
cp -r headless-cli-playbook/skills/headless-cli ~/.claude/skills/
```

装完后任意 Claude Code 会话按需自动加载(或 `/headless-cli` 手动调用)。三个 CLI 按需安装,不必全装。

## 三个 CLI 一句话

| CLI | 模型来源 | 认证 | 无头入口 |
|---|---|---|---|
| `codex` | ChatGPT 订阅(GPT 系) | `codex login --device-auth` | `codex exec "..." < /dev/null` |
| `opencode` | 任意 OpenAI 兼容端点(DeepSeek、各类网关) | 环境变量 API key | `opencode run "..."` |
| `agy` | Gemini | Google OAuth | `agy -p "..." < /dev/null` |

对国内用户:`opencode` 的 provider 是通用 OpenAI 兼容模板,DeepSeek 官方 API、腾讯云等聚合网关都能直接填(README 里有完整 JSON 配置示例,key 一律走 `{env:VAR}`,不落盘)。

## 最容易踩的三个坑(全表见英文 README)

1. **`codex exec` 永久挂死** → 结尾加 `< /dev/null`(它会读 stdin 直到 EOF)。
2. **并行跑 `opencode run` 报 `database is locked`** → 每个进程给独立数据目录:`XDG_DATA_HOME=$(mktemp -d) XDG_STATE_HOME=$(mktemp -d) opencode run …`(上游 issue 至今未修,这是已验证的绕法)。
3. **`opencode run` exit 0 但 stdout 为空** → 读文件时钉死**绝对路径**;自动化场景用 `--format json` 检测 `state.status: "error"`。

## 给 AI 装

把仓库链接丢给你的 Claude Code / coding agent,让它读 [AGENTS.md](AGENTS.md) —— 安装、验证、安全规则都在里面。
