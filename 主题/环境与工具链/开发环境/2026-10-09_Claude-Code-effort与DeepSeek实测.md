---
title: Claude Code 推理强度（effort）调整与 DeepSeek 是否支持实测
date: 2026-10-09
updated: 2026-10-09
project: 开发环境
type: 方案选型
themes: [环境与工具链]
tags: [claude-code, deepseek]
status: 已解决
---

# Claude Code 推理强度（effort）调整与 DeepSeek 是否支持实测

> 整理日期：2026年10月9日 20:23

## 环境信息

- 操作系统：Windows 11 Home China（10.0.26200），Shell 为 Git Bash
- Claude Code：v2.1.201（`C:\Users\luza\AppData\Roaming\npm\node_modules\@anthropic-ai\claude-code`）
- 模型端点：DeepSeek 的 Anthropic 兼容端点 `https://api.deepseek.com/anthropic`，模型 `deepseek-flash[1m]`
- 实测脚本环境：Python 3.8.15 + requests 2.28.1
- 当时的推理强度：环境变量 `CLAUDE_EFFORT=high`（只读输出变量，非设置项）

## 背景与问题

1. 起因是询问「当前推理强度是多少」，进而问「如何根据任务难度调整」。
2. 关键疑点：本地走的是第三方端点（DeepSeek），Claude Code 发出的 effort 参数**服务端认不认**无法凭猜测断定，需要实测。
3. 二进制字符串反查得到 Claude Code 侧的机制：
   - 档位：`low / medium / high / xhigh / max`，另有 `auto`、`ultracode`（= xhigh + 动态 workflow，仅本会话）
   - 设置入口：`/effort` 命令、`--effort <level>` 参数、`settings.json` 的 `effortLevel`、环境变量 `CLAUDE_CODE_EFFORT_LEVEL`
   - `CLAUDE_EFFORT` 是**输出**变量（供 hook / Bash 读取当前生效档位），`CLAUDE_CODE_EFFORT_LEVEL` 才是**输入**覆盖变量 —— 两者方向相反，别搞混
   - 子 agent 可单独设：`.claude/agents/*.md` frontmatter 的 `effort` 字段

## 报错记录

| 现象 | 原因 | 解决 |
|---|---|---|
| `ANTHROPIC_BASE_URL=http://127.0.0.1:9999 claude -p ...` 指向**死端口也照样成功** | `~/.claude.json` 顶层 `env` 块强制覆盖了进程环境变量，前缀写法无效 | 改用 `claude --settings '{"env":{"ANTHROPIC_BASE_URL":"..."}}'`，实测生效 |
| 代理日志单次请求达 185KB | 日志脚本只排除了 `messages`，把 `system`+`tools`（26 个工具，约 22k token）也 dump 了 | 过滤集合改为 `{messages, system, tools}`，只记 model / max_tokens / thinking / output_config |
| 重启代理后仍是旧代码在跑 | 旧进程未被杀掉（`kill %1` 无效，后台任务是独立进程），新进程绑定 8899 失败后静默退出 | 用 `netstat -ano \| grep :8899` 取 PID，再 `taskkill //F //PID` |
| `422 Failed to deserialize ... unknown variant` | **刻意触发的探针**，非故障 | 见下方「修改方案」 |

## 涉及文件

- `C:\Users\luza\.claude\settings.json` —— 仅含 permissions 与模型名 env，**无** effort 相关配置
- `C:\Users\luza\.claude.json` —— 顶层 `env` 块（`ANTHROPIC_BASE_URL` / `ANTHROPIC_AUTH_TOKEN` / `API_TIMEOUT_MS`），本次确认其**优先级高于进程环境变量**
- `C:\Users\luza\.claude\skills\对话整理\` —— 本次整理所用 skill
- 临时实测脚本（`_effort_proxy.py` / `_effort_test.py` / `_effort_effect.py` 及 log/json）—— 已验证完毕，**已删除**

## 修改方案（实测过程）

### 1. 客户端侧：抓包确认发了什么

写了一个转发到 `api.deepseek.com` 的本地 HTTP 日志代理（监听 8899），用 `--settings` 把端点指过去，分别以 `--effort low` 和 `--effort xhigh` 各跑一次 `claude -p`。

**结果**：两次请求唯一差异就是这个字段 ——

```json
"max_tokens": 32000,
"thinking": {"type": "adaptive", "display": "omitted"},
"output_config": {"effort": "low"}      // xhigh 那次这里是 "xhigh"
```

即 Claude Code **发 `output_config.effort`，不发 `budget_tokens`**。另：请求里 model 是 `deepseek-flash`，`[1m]` 只是客户端本地后缀，不上线。

### 2. 服务端侧：探针法验证是否解析

核心思路：**被忽略的未知字段永远不会被类型检查**。因此故意发 schema 违规的值：

- `output_config` 传字符串 → `422 invalid type: string, expected struct OutputConfig`
- `effort` 传非法值 → `422 unknown variant ... expected one of none, minimal, low, medium, high, xhigh, ultra, max`
- `output_config` 内多塞一个 `budget_tokens` → 200（多字段被容忍）

报错原文直接暴露了 DeepSeek 自己的枚举，证明该字段被反序列化成 struct，**不是被丢弃**。

### 3. 行为侧：确认取值真的改变行为

用一道必须真算的题（`C(100,50)` 的十进制值及其数字 7 的个数，答案可独立验证），测量响应中 thinking 块的长度。

## 结果与结论

**结论：DeepSeek 完整支持并严格执行 `output_config.effort`。**

1. **枚举不同但兼容**。DeepSeek 侧真实枚举为
   `none | minimal | low | medium | high | xhigh | ultra | max`
   （比 Claude Code 多出 `none` / `minimal` / `ultra` 三档）。Claude Code 的五档全部落在合法集合内，**不存在档位对不上的问题**。
2. **取值真实生效的铁证**：`effort: "none"` 时 thinking 长度 **6/6 次精确为 0**；不传该字段时稳定有 500~2500 字符推理。被忽略的字段不可能有这种确定性表现。

   | effort | thinking 长度（字符） | 次数 |
   |---|---|---|
   | 不传 | 551 / 1863 / 2571 | 3 |
   | none | 0 / 0 / 0 / 0 / 0 / 0 | 6 |
   | low | 135 / 600 / 996 / 1365 | 4 |
   | high | 2178 / 2322 | 2 |
   | ultra | 2002 / 4862 | 2 |
   | max | 2514 / 4281 | 2 |

3. **Claude Code 够不到最省的两档**。它最低只能发 `low`，而 DeepSeek 另有 `none` / `minimal`。想在 Claude Code 里彻底关闭推理做不到 —— `effort: low` 仍然有推理。
4. **中高档之间方差极大**，ultra 两次跑出 2002 与 4862 字符（差 2.4 倍）。**不能拿单次观测下结论**说某档更强，需多次取均值。
5. 修正了此前的错误记录：`/models` 端点返回的 "effort: low/high/max" 是不完整信息，真实枚举有 8 档。
6. 附带确认：`~/.claude.json` 的顶层 `env` 块会强制覆盖进程环境变量，临时改端点必须用 `--settings`。

## 变更记录

- 2026-10-09 20:23 首次整理
