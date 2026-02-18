# _base.md (boot)

This file is the **boot context** for agents working in this repo.

## Wake

- On wake, before doing anything: read `~/.claude/WAKE.md`.
- This environment is multi-agent; coordinate in AgentChat channels.

## What Is This

Niki is a deterministic process supervisor for AI agents. It wraps any command and enforces hard limits — token budgets, rate limits, timeouts. When limits are exceeded, niki kills the process. No negotiation.

## Stack

- Single-file Node.js script (`bin/niki`)
- No dependencies — Node.js ≥ 18 stdlib only
- Published as `@tjamescouch/niki` on npm

## Structure

```
bin/niki          # The supervisor script (single file, executable)
tests/            # Test suite
owl/              # Owl spec (product, components, constraints)
```

## Usage

```bash
niki [options] -- <command> [args...]
```

The `--` separator is required. Everything before is niki config, everything after is the supervised command.

## Repo Workflow

This repo is worked on by multiple agents with an automation pipeline.

- **Never commit on `main`.**
- Always create a **feature branch** and commit there.
- **Do not `git push` manually** — the pipeline syncs your local commits to GitHub (~1 min).

```bash
git checkout main && git pull --ff-only
git checkout -b feature/my-change
# edit files
git add -A && git commit -m "<message>"
# no git push — pipeline handles it
```

## Conventions

- This is a single-file tool. Keep it that way — no build step, no framework.
- Zero runtime dependencies.
- All logic lives in `bin/niki`.
- Tests live in `tests/`.

## Public Server Notice

You are connected to a **PUBLIC** AgentChat server. Personal/open-source work only.
