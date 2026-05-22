# Plan-Then-Delegate

[English](README_en.md) | 中文

> 轻量的智能任务分发 Skill：让主代理专注方案对齐与文档整理，代码落盘 + 自跑编译派给子代理串行承担。分成 Claude Code 和 Codex 版本。

- **轻量实现**：一个 skill + 三份 references
- **智能分发**：主代理基于已对齐方案精准派发，子代理冷启动也直奔目标
- **保持主代理上下文干净**：主代理不下场写代码、不重复跑编译，把上下文留给方案判断与文档整理
- **双平台**：`claude-code/` 和 `codex/` 两个版本平级，按平台选用

## 解决什么

多步骤编码任务（修多个 bug、改多个文件）由主代理一手包办时：

- 主代理上下文耗在具体改动细节上，挤掉对方案的判断与文档整理

本技能拆分角色——主代理负责"对齐 + 整理"，子代理负责"落盘 + 自跑编译"。

## 适用场景

**适合**：

- 一次任务涉及多个独立问题 / 多文件 / 多模块改动
- 你希望主代理把上下文留给"判断"而不是"细节"
- 项目 CLAUDE.md / 项目记忆中声明默认使用本工作流

**不适合**：

- 单行 typo / 简单变量重命名（主代理直接改更快）
- 纯研究 / 探索任务（用 explore / general-purpose 子代理）
- 用户明确说"这次你直接改"

## 安装

把对应平台的 skill 目录软链到用户级 skills 目录：

```bash
# Claude Code
ln -s "$(pwd)/claude-code/plan-then-delegate" ~/.claude/skills/plan-then-delegate

# Codex
ln -s "$(pwd)/codex/plan-then-delegate" ~/.codex/skills/plan-then-delegate
```

> ⚠️ 必须链整个目录（含 `references/`）。仅链 `SKILL.md` 单文件会导致子代理读不到执行规则。

## 触发

**仅在用户显式要求时启用**：

- 显式调用 `/plan-then-delegate`（Claude Code）/ `$plan-then-delegate`（Codex）
- 明文提到"走 plan-then-delegate 工作流"等
- 项目 CLAUDE.md / 项目记忆中声明默认采用

主代理**不会**根据任务表象（"看起来复杂"、"多个问题"）自行启用。

## 工作流概览

```
用户报问题
       │
主代理排查 → 与用户对齐方案
       │
       ▼
[串行] 实现子代理 1 落盘 + 自跑编译 → 返回修改要点
       │
[串行] 实现子代理 2 落盘 + 自跑编译 → 返回修改要点
       │
       ...
       ▼
（可选）补充测试循环：测试代理 V ↔ 修复代理 F 轮流跑
       ▼
主代理统一整理项目约定的汇总 / 长期文档
```

核心规则：

- 一次一个子代理（不并行）
- 前台运行（Claude Code 不 `run_in_background`）
- 一个问题对应一个子代理（同问题多文件全部在同一子代理里完成）
- 子代理只反馈"修改要点"；文档由主代理统一组织
- 实现子代理自跑编译；测试由测试子代理负责（如启用补充测试循环）

详细执行规则见各平台 `SKILL.md`。

## What's Inside

```
repo/
├── README.md              # 本文件（中文）
├── README_en.md           # 英文版
├── LICENSE                # MIT
├── docs/
│   └── design.md          # 设计文档
│
├── claude-code/plan-then-delegate/
│   ├── SKILL.md           # 主代理读：核心原则、5 阶段工作流、子代理 prompt 模板
│   └── references/
│       ├── agent-impl.md  # 实现子代理执行规则
│       ├── agent-test.md  # 测试子代理执行规则
│       └── test-loop.md   # 补充测试循环详细规则
│
└── codex/plan-then-delegate/
    ├── SKILL.md           # 主代理读（Codex 版，工具名为 spawn_agent/send_input）
    ├── agents/
    │   └── openai.yaml    # Codex UI 元数据
    └── references/
        ├── agent-impl.md  # 实现子代理执行规则
        ├── agent-test.md  # 测试子代理执行规则
        └── test-loop.md   # 补充测试循环详细规则（使用 send_input）
```

## 环境要求

### Claude Code

- Claude Code **v2.1.32+**（`claude --version` 确认）
- 第 4 节「补充测试循环」依赖 `SendMessage` 工具，需启用 Agent Teams 实验功能；不启用则跳过该节，其他节正常工作

### Codex

- Codex CLI（`codex --version` 确认）
- `send_input` 为原生工具，补充测试循环无需额外功能开关

### 启用 Agent Teams（仅 Claude Code）

任选其一：

```json
// ~/.claude/settings.json
{ "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
```

```sh
# ~/.bashrc / ~/.zshrc
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

启用后**重启 Claude Code 会话**让新工具注入。验证：主代理工具列表包含 `SendMessage`。
