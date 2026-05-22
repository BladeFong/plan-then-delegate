# Plan-Then-Delegate

English | [中文](README.md)

> A lightweight skill for smart task delegation: keep the main agent focused on plan alignment and documentation; delegate code changes + self-verified compilation to subagents, run serially. Separate Claude Code and Codex versions.

- **Lightweight implementation**: one skill + three reference docs
- **Smart delegation**: the main agent dispatches based on the plan it already aligned with you — subagents hit the ground running even on cold start
- **Keeps the main agent's context clean**: main agent doesn't write code itself or rerun compilation — its context stays free for judgment and documentation
- **Dual-platform**: `claude-code/` and `codex/` versions side by side, pick the one for your platform

## What it solves

When the main agent handles multi-step coding tasks (fixing multiple bugs, editing multiple files) end-to-end:

- The main agent's context is consumed by low-level details (read files, write code, fix imports), crowding out the bandwidth needed for plan judgment and documentation.

This skill splits the roles — **main agent**: alignment + documentation; **subagents**: code changes + self-verified compilation.

## When to use

**Good fit**:

- A single task spans multiple independent issues / files / modules
- You want the main agent to spend its context on *judgment*, not *minutiae*
- Project `CLAUDE.md` or project memory declares this workflow as default

**Not a fit**:

- Single-line typos / trivial renames (main agent edits directly is faster)
- Pure research / exploration tasks (use `explore` / `general-purpose` subagents)
- User explicitly says "just edit it yourself this time"

## Install

Symlink the platform-specific skill directory into your user-level skills folder:

```bash
# Claude Code
ln -s "$(pwd)/claude-code/plan-then-delegate" ~/.claude/skills/plan-then-delegate

# Codex
ln -s "$(pwd)/codex/plan-then-delegate" ~/.codex/skills/plan-then-delegate
```

> ⚠️ Link the whole directory (including `references/`). Linking only `SKILL.md` will leave subagents unable to read their execution rules.

## How it gets triggered

**Only when the user explicitly opts in**:

- Explicit invocation via `/plan-then-delegate` (Claude Code) or `$plan-then-delegate` (Codex)
- Mentioning the workflow by name in natural language ("go with plan-then-delegate", "main-aligns-then-subagents-implement style")
- Project `CLAUDE.md` or project memory declaring this as the default workflow

The main agent **will not** self-trigger this skill based on surface cues ("looks complex", "multiple issues").

## Workflow overview

```
User reports issues
       │
Main agent diagnoses → aligns plan with user
       │
       ▼
[serial] Impl subagent 1 lands changes + self-runs compile → returns change notes
       │
[serial] Impl subagent 2 lands changes + self-runs compile → returns change notes
       │
       ...
       ▼
(optional) Supplementary test loop: test agent V ↔ fix agent F take turns
       ▼
Main agent consolidates everything into the project's summary / long-term docs
```

Core rules:

- One subagent at a time (no parallelism)
- Foreground only (Claude Code: no `run_in_background`)
- One issue → one subagent (all files for that issue go in the same subagent)
- Subagents return only "change notes"; the main agent organizes documentation
- Impl subagents self-run compile; tests are owned by the test subagent (when the supplementary test loop is enabled)

See each platform's `SKILL.md` for full execution rules.

## What's Inside

```
repo/
├── README.md              # This file (Chinese)
├── README_en.md           # English version
├── LICENSE                # MIT
├── docs/
│   └── design.md          # Design document
│
├── claude-code/plan-then-delegate/
│   ├── SKILL.md           # Main agent: core principles, 5-stage workflow, subagent prompt template
│   └── references/
│       ├── agent-impl.md  # Impl subagent execution rules
│       ├── agent-test.md  # Test subagent execution rules
│       └── test-loop.md   # Supplementary test loop details
│
└── codex/plan-then-delegate/
    ├── SKILL.md           # Main agent (Codex version, tools: spawn_agent/send_input)
    ├── agents/
    │   └── openai.yaml    # Codex UI metadata
    └── references/
        ├── agent-impl.md  # Impl subagent execution rules
        ├── agent-test.md  # Test subagent execution rules
        └── test-loop.md   # Supplementary test loop details (uses send_input)
```

## Requirements

### Claude Code

- Claude Code **v2.1.32+** (verify with `claude --version`)
- Stage 4 "Supplementary test loop" depends on the `SendMessage` tool, which requires the Agent Teams experimental feature. Without it, stage 4 is skipped; other stages work normally.

### Codex

- Codex CLI (verify with `codex --version`)
- `send_input` is a native tool — the supplementary test loop works without extra feature flags.

### Enabling Agent Teams (Claude Code only)

Pick one:

```json
// ~/.claude/settings.json
{ "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
```

```sh
# ~/.bashrc / ~/.zshrc
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

After enabling, **restart your Claude Code session** so the new tools get injected. Verify: the main agent's tool list now includes `SendMessage`.
