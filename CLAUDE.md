# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

`gh-analyser-workflow` is **not** a conventional application — it is a **Claude Code automation pipeline**. The entire "source" is the set of Markdown skill/agent definitions under `.claude/`. There is no compiled code, no package manifest, and no test suite. Editing this repo means editing prompts (Markdown with YAML frontmatter) that Claude Code itself executes at runtime.

The pipeline batch-analyses a repository's **open GitHub issues**: it fetches each open issue via the `gh` CLI, spawns one isolated sub-agent per issue, and writes a codebase-grounded development plan to `issues/issue-<N>.md` — one document per issue.

## How it runs

There are no build/lint/test commands. You exercise the pipeline by invoking it through Claude Code:

- **Trigger the pipeline** — from inside a target repo, ask Claude Code to "analyse the open GitHub issues" (or similar). This loads the orchestrator skill.
- **Preflight the environment** the skill depends on:
  ```bash
  gh --version
  gh repo view --json nameWithOwner -q .nameWithOwner   # must resolve a repo
  ```
- **Inspect the issue payload** the pipeline consumes:
  ```bash
  gh issue list --state open --limit 100 \
    --json number,title,state,author,labels,assignees,milestone,createdAt,updatedAt,url,body,comments
  ```
- **Outputs** land in `./issues/issue-<N>.md` and are the deliverable; they are regenerated each run.

## Architecture: a two-component orchestration

The big-picture design is an **orchestrator + worker** split, and the key invariant is **context isolation** between them.

1. **Orchestrator skill** — `.claude/skills/migrate-github-issues-to-handoffs/SKILL.md`
   Runs in the main conversation. It does preflight, fetches issues with `gh`, resolves an **absolute** output path, reads the repo's `CLAUDE.md` as a shared **project map**, then dispatches one worker per issue and verifies the results. **It never analyses issues itself** — all analysis is delegated.

2. **Issue-analyser agent** — `.claude/agents/github-issue-analyser.md`
   Spawned **once per issue** via the `Agent` tool (`subagent_type: github-issue-analyser`), each in a **clean context window**, up to **5 in parallel** (larger backlogs run in sequential waves of 5). Each worker sees only its own single-issue JSON payload plus the shared project map, explores the codebase **read-only in planning mode**, and writes **exactly one** Markdown file.

3. **Field reference** — `.claude/skills/migrate-github-issues-to-handoffs/references/gh-issue-fields.md`
   Documents the `gh issue list` JSON shape and how filter args are forwarded.

### Load-bearing invariants (don't break these when editing)

- **One issue per spawn.** Never pass the full issue list to a single worker — the clean-context, one-issue-per-agent design is the whole point. Workers must not call `gh` to fetch other issues.
- **The orchestrator delegates; it does not analyse.** Keep analysis logic inside the agent definition, not in `SKILL.md`.
- **Absolute paths across the boundary.** Sub-agents can't resolve the parent's cwd reliably, so the orchestrator resolves and passes an absolute output path (`<root>/issues/issue-<N>.md`).
- **`CLAUDE.md` is the shared project map** — read-only input to the pipeline. The orchestrator reads it once per batch and pastes it into every worker so they skip generic onboarding. The pipeline **consumes** this file; it must not rewrite it. The map is **descriptive, not authoritative**: a worker that finds the code contradicting the map trusts the code and notes the discrepancy.
- **Default scope is OPEN issues.** Only widen to closed/all or add label/assignee/search filters when the user explicitly asks; forward those to `gh`.
- **Output safety.** Don't overwrite an existing `issue-<N>.md` without confirming with the user (overwrite / skip / write `-v2`). Don't touch repo files outside `./issues/`.

## Conventions for editing the definitions

- Skill and agent files are Markdown with **YAML frontmatter**. The agent frontmatter declares its `name`, `description`, `tools` allowlist (`Read, Glob, Grep, Write, Bash, ExitPlanMode`), and `model`. Changing the tool allowlist changes what the worker can do at runtime — edit deliberately.
- The agent enforces a fixed **output document template** (frontmatter + Goals / Acceptance Criteria / Codebase Analysis / Development Plan / Risks / Open Questions / Effort Estimate). Keep generated docs matching that structure.
- The skill's phased workflow (Preflight → Fetch → Resolve path → Resolve project map → Spawn → Verify → Summarise) is the contract between the two components. If you change a phase, keep the orchestrator and agent in sync — and update the README and this file when the contract changes.
