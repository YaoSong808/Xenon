# Xenon

[![CI](https://github.com/xianyu-sheng/Xenon/actions/workflows/ci.yml/badge.svg)](https://github.com/xianyu-sheng/Xenon/actions/workflows/ci.yml)
[![codecov](https://codecov.io/gh/xianyu-sheng/Xenon/branch/main/graph/badge.svg)](https://codecov.io/gh/xianyu-sheng/Xenon)
[![Release](https://img.shields.io/github/v/release/xianyu-sheng/Xenon)](https://github.com/xianyu-sheng/Xenon/releases)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://www.python.org/downloads/)

[English](README.en.md) | 简体中文

<p align="center">
  <img src="docs/logo-animated.gif" alt="Xenon — 像素跑者沿中线奔跑，逐渐消散重组为方块 XENON 标识" width="666">
</p>

**一个把"机器到底验证了什么"摊开给你看的终端 AI 编码代理。**

别的 agent 说"完成了"，你只能选择相信它。Xenon 会把结果拆成两半说清楚：哪些是机器确认过的，哪些需要你自己看一眼。

```text
❯ 修复 config.py 里的超时默认值

  🔧 edit_file  config.py                     ✓
  🔧 run        pytest                        ✗ 2 failed
  🏁 交付（降级）— 以下 2 项机器无法核实，请人工确认：
     · "已覆盖所有调用点"（实际只确认 1/3 处引用）
     · 未运行集成测试
  🧾 证据链：41 条事件，完整连续 · /evidence 可随时复核
```

<sub>简化示意；真实界面会展示每次工具调用的完整记录。</sub>

```bash
# 30 秒安装（无需 PyPI）
pip install https://github.com/xianyu-sheng/Xenon/releases/download/v0.9.4/xenon_harness-0.9.4-py3-none-any.whl
xenon   # 首次运行会引导配置（API Key、模型、推理引擎）
```

> **Windows 提示：** 如果提示 `'xenon' 不是内部或外部命令`，说明 Scripts 目录不在 PATH——pip 安装时通常会打印 `WARNING: The script xenon.exe is installed in '...' which is not on PATH`，按那个路径运行，或把它加进 PATH；也可以直接用 `python -m xenon`（v0.9.4+）。混用 conda 和独立 Python 时，建议用专用 venv：`python -m venv .venv`。

## 它具体做什么

- **每一次工具调用都进证据链。** 会话事件写入 append-only 哈希链，每条带内容哈希与前驱哈希；`/evidence` 随时重算自证，能发现损坏、单点篡改和重排。
- **只有一个阻断类校验。** "空洞回答"（没干活、答非所问）是确定性的，命中就同回合反馈重试、同因熔断；语义类问题不阻断交付，而是整理成随答案输出的「待人工核验清单」。
- **每个回合是一棵树上的节点。** `/rewind` 回退、`/fork` 分叉；中断或熔断后，裸敲"继续"会自动从那个节点接上，不用重述上下文。
- **越权操作要走审批。** 工作区内写文件不打扰你；越界写和命令会弹统一面板（带 diff 预览），授权可以按目标和时长限定，子代理默认只读。
- **大文件截断、上下文超限都如实标注，** 不伪装完整。

## 一个回合怎么走

```text
你的输入
  → 意图判定（LLM 分类为主，失败自动降级正则）
  → 引擎执行（预算检查点 / 循环检测 / 断路器）
  → 回合校验（唯一判定点）
      pass    → 正常交付
      remind  → 交付 + 待人工核验清单（不阻断）
      fail    → 同回合重试，预算用尽则降级交付
      fuse    → 同因结构失败熔断，按未完成交付
  → 会话树落盘 + 哈希链追加 + 回合归档
```

## 机器验证了什么，没验证什么

| 层 | 测的是什么 | 结果怎么呈现 |
|---|---|---|
| **L0 证据层** | 过程是否真实、可复核 | 哈希链事件日志；`/evidence` 随时重算 |
| **L1 结构层** | 回答是不是"没干活" | 唯一的阻断类校验：反馈重试 / 同因熔断 |
| **L2 语义层** | 宣称、准则、文件声称是否站得住 | 不阻断；命中即附「待人工核验清单」，重试耗尽则降级交付 |

语义判断全仓只有一处（`xenon/turn/gate.py`），引擎内部不做语义判定；工具成败只由工具回执决定，不靠关键词猜。

`/evidence` 的输出（结构化回合摘要 + 一行链自证）：

<img src="docs/evidence-chain.png" alt="/evidence 输出示例" width="720">

## 安装

```bash
# 方式一：从 GitHub Releases 安装 wheel（推荐）
pip install https://github.com/xianyu-sheng/Xenon/releases/download/v0.9.4/xenon_harness-0.9.4-py3-none-any.whl

# 方式二：源码安装（editable，浅克隆省流量）
git clone --depth 1 https://github.com/xianyu-sheng/Xenon.git
cd Xenon && python -m venv .venv
.venv/Scripts/python -m pip install -e .        # Windows
source .venv/bin/activate && pip install -e .   # Linux / macOS
```

也可以用 `uv tool install <wheel URL>` 或 `pipx install <wheel URL>`，入口会自动处理。

## 常用命令

```
/evidence   查看证据链（当前 / 全局 / 指定会话）      /plan   计划模式
/goal       设定本轮目标                             /review 审阅改动（含 .bak 回退）
/rewind     回退到某个回合                           /fork   从某个回合分叉
/sessions   会话列表                                 /compact 压缩长对话
/permissions 查看当前授权                           /tools  工具清单
```

启动时先选会话：直接敲 `xenon` 会弹会话选择器（数字选历史会话或新对话），`xenon resume` 恢复最近一次。恢复后会回放对话历史。

## 环境变量

| 变量 | 作用 | 默认 |
|---|---|---|
| `XENON_APPROVAL_TTL` | 会话级 `a` 授权规则的有效期（秒） | 1800 |
| `XENON_VERIFY_RETRIES` | 结构层失败后的同回合重试次数 | 2 |
| `XENON_RETRY_BUDGET_MIN/MAX` | 重试轮的步数预算区间 | 10 / 40 |
| `XENON_GATE_FUSE_THRESHOLD` | 同因结构失败的熔断阈值 | 2 |
| `XENON_MAX_ITERATIONS` | 单次运行的步数上限 | 40 |
| `XENON_MAX_TOKENS` | 单次模型调用的输出上限 | 8192 |
| `XENON_MAX_CONTINUATIONS` | 截断后的自动续写次数 | 3 |
| `NO_COLOR` | 标准环境变量；关闭输出与输入提示的颜色 | 未设置 |
| `XENON_READ_ONLY` | 只读模式（禁用一切写/执行工具） | 关 |
| `XENON_COMMAND_ALLOWLIST` | 命令首词白名单（逗号分隔） | 空 = 不限 |
| `XENON_SESSION_EVENTS(_DIR)` | 事件日志开关 / 目录 | 开 |
| `XENON_HOOKS` | 本地 hooks 开关 | 开 |
| `GITHUB_TOKEN` / `GH_TOKEN` | GitHub 工具认证（提升 API 限额） | 空 |

## Windows

- Windows Terminal (ConPTY) + PowerShell 全功能支持（粘贴、中文输入法、方向键）。
- `XENON_TERMINAL_ANIMATION=0` 关闭动画。

## 诚实边界

- **机器不验证语义。** L2 命中时只提醒不阻断——内容真伪请对照「待人工核验清单」自行确认。
- **哈希链证明的是过程，不是结论。** 它保证事件流未被篡改，不保证模型答对了。
- **链本身是无密钥锚点的完整性检查。** 能发现偶然损坏、单点篡改和重排，防不住"有写权限的人整链重算"；需要强抗篡改请把事件目录放到受控存储。（可选 HMAC 锚点见 #106）
- **沙箱是检测 + 软件边界，不是真隔离。** 不可信代码请在 WSL / Docker 容器里运行。
- **工具输出的注入检测是启发式提醒，不是安全边界。** Xenon 会标记来源并提示常见注入模式，但不会阻断或改写内容；范围与限制见 [工具输出威胁模型](docs/THREAT_MODEL.md)。
- `/review` 的回退只覆盖有 `.bak` 备份的编辑；版本管理仍请交给 git。
- 意图分类器默认走最快可用的模型，约 1 秒延迟；不可用时会降级为正则（偏保守）。

## 收录

<!-- 按榜单影响力排序；新收录在合并后插入到对应位置 -->

- [awesome-opensource-ai](https://github.com/alvinreal/awesome-opensource-ai) — 开源 AI 项目清单（Agentic AI & Multi-Agent Systems → Autonomous Coding Agents）
- [awesome-mcp-lists](https://github.com/collabnix/awesome-mcp-lists) — Collabnix 社区的 MCP 客户端 / 工具清单（MCP Clients → Command Line Tools）

## License

MIT
