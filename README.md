# RLM: Recursive Language Models for Claude Code (Windows Fork)

A Claude Code skill that implements **Recursive Language Models (RLMs)** — enabling Claude to process arbitrarily long inputs by treating them as external data and recursively delegating analysis to sub-agents.

Forked from [Tenobrus/claude-rlm](https://github.com/Tenobrus/claude-rlm) with modifications for **cross-platform compatibility** (Windows/Git Bash/MINGW support).

Based on the paper [Recursive Language Models](https://arxiv.org/abs/2512.24601) (Zhang, Kraska, Khattab; MIT CSAIL, 2026).

> **Note: Unverified / Experimental.** This fork is context-hungry and has only been tested with literary texts (Shakespeare complete works, Frankenstein, War and Peace). It has not been validated for real-world use cases such as large codebases, legal documents, or structured data. The recursive fan-out can consume significant API quota quickly — a single analysis run used ~12% of a weekly Claude Code quota. Further testing and optimization needed before production use.

> **Note: No tmux/psmux support.** The upstream repo uses `tmux` for sub-agent session management. This fork replaces it entirely with background processes and PID files, since `tmux` (and Windows alternatives like `psmux`) do not work reliably in Git Bash/MINGW environments.

## Changes from upstream

- **Replaced `tmux` with background processes** — uses PID files for tracking instead of tmux sessions, which aren't available on Windows
- **Fixed `sed -i` compatibility** — removed macOS-only `''` argument
- **Piped prompts via stdin** — `cat prompt | claude -p` instead of argument passing (avoids shell arg length limits)
- **Added dead process detection** — sub-agents that crash are detected and reported
- **Added `unset CLAUDECODE`** — prevents env leakage into sub-agent sessions

## Installation

Copy the skill directory into your Claude Code skills folder:

```bash
# Clone the repo
git clone https://github.com/Linaqruf/claude-rlm-windows.git

# Copy skill into Claude Code
cp -r claude-rlm-windows/rlm-skill ~/.claude/skills/rlm
```

Once installed, Claude Code activates the RLM skill automatically when processing large contexts, or you can trigger it explicitly with `/rlm`.

### Uninstall

```bash
rm -rf ~/.claude/skills/rlm
```

## How it works

LLMs have limited context windows, and performance degrades as prompts get longer (**context rot**). RLMs solve this by treating the input as an external object in a programming environment, letting the LLM programmatically interact with it through code — including recursively calling itself on slices of the input.

**Two key ideas:**

1. **Bash is the programming environment.** The filesystem is the variable store. File paths are variable names. `cat` is dereference. `>` is assignment. `split` is destructuring. The LLM writes bash that manipulates file-symbols — the actual content stays in files and never enters the context window.

2. **Sub-agents do the understanding.** When semantic work is needed (classification, summarization, extraction), the LLM writes a prompt file and hands it to a sub-agent via `rlm-query` or `rlm-batch`. Each sub-agent is a full Claude Code instance that can itself spawn sub-agents if needed.

## Architecture

```
User
  |
  v
Claude Code (interactive session, skill loaded)
  |
  |-- reads rlm-agent.md for instructions
  |-- examines context metadata (wc, head)
  |-- splits context with bash (split, sed)
  |-- writes prompt files, one per chunk
  |-- calls rlm-batch to fan out
  |       |
  |       |-- rlm-query chunk1 --> bash bg --> claude -p --> result1.out
  |       |-- rlm-query chunk2 --> bash bg --> claude -p --> result2.out
  |       |-- rlm-query chunk3 --> bash bg --> claude -p --> result3.out
  |       |       |
  |       |       └── (can itself call rlm-query/rlm-batch recursively)
  |       |
  |       └── waits for all, collects results
  |
  |-- reads result files, aggregates with code
  |-- returns final answer
```

## Components

| File | Purpose |
|------|---------|
| `rlm-skill/SKILL.md` | Skill trigger — points Claude to `rlm-agent.md` |
| `rlm-skill/prompts/rlm-agent.md` | Shared RLM instructions for all agents (examples, tools, config) |
| `rlm-skill/scripts/rlm-query` | Spawns a Claude Code sub-agent as a background process with concurrency limiting, timeout, and depth tracking |
| `rlm-skill/scripts/rlm-batch` | Parallel fan-out — runs `rlm-query` on every `.md` file in a directory concurrently |

## Commands

```bash
# Single sub-agent call
rlm-query <prompt_file> <output_file> [--context <file>] [--task NAME] [--model MODEL] [--max-depth N]

# Parallel fan-out (preferred)
rlm-batch <prompts_dir> <results_dir> [rlm-query options...]
```

## Configuration

All settings propagate through the recursion tree via environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `RLM_MAX_DEPTH` | 3 | Maximum recursion depth |
| `RLM_MAX_PARALLEL` | 15 | Max concurrent Claude instances |
| `RLM_TIMEOUT` | 1200 | Timeout per sub-query (seconds) |
| `RLM_MODEL` | opus | Model for sub-agents |
| `RLM_TASK` | auto | Task name (auto-generated if not set) |

**Cost tip:** Use `--model sonnet` for leaf tasks (classification, extraction, summarization) and reserve `opus` for final aggregation/synthesis. This significantly reduces token usage.

## Monitoring

Sub-agents run as background processes. Monitor via PID files and logs:

```bash
# List active sub-agents
for f in .rlm/<task>/sessions/*/pid; do
    pid=$(cat "$f" 2>/dev/null)
    kill -0 "$pid" 2>/dev/null && echo "RUNNING: $(dirname "$f" | xargs basename) (pid $pid)"
done

# Check progress
echo "$(find .rlm/<task>/sessions -name '.done' | wc -l) done / $(ls .rlm/<task>/sessions/ | wc -l) total"

# Tail a sub-agent's log
tail -f .rlm/<task>/sessions/rlm-<task>-d1-…/stderr.log
```

## References

- [Recursive Language Models](https://arxiv.org/abs/2512.24601) — Zhang, Kraska, Khattab (MIT CSAIL, 2026)
- [Original implementation](https://github.com/alexzhang13/rlm) — Official Python implementation
- [Original Claude Code skill](https://github.com/Tenobrus/claude-rlm) — Upstream repo this fork is based on
