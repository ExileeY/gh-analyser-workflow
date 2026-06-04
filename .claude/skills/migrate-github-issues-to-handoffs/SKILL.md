---
name: migrate-github-issues-to-handoffs
description: This skill should be used when the user asks to "analyse github issues", "analyze github issues", "fetch and analyse issues", "review open issues", "produce issue analysis documents", "create issue analysis reports", or wants to run a batch analysis of the current repository's open GitHub issues with one detailed development-plan document per issue. Fetches OPEN issues from the current repo with `gh`, then spawns one `github-issue-analyser` sub-agent per issue (clean context window each). Each sub-agent analyses the codebase in planning mode and writes `issues/issue-<N>.md` at the project root.
---

# Migrate GitHub Issues to Handoffs

Orchestrate a batch analysis of the current repository's **open** GitHub issues. Fetch them with the GitHub CLI, spawn one `github-issue-analyser` sub-agent per issue (each with a clean context window), and produce one detailed development-plan document per issue at `./issues/issue-<issue-number>.md`.

## When to Use

Trigger on requests such as:
- "Analyse the open GitHub issues in this repo"
- "Run an analysis pass over the open issues"
- "Produce dev plans for open issues"
- "Review the open issues backlog and propose development plans"

## Hard Requirements

- The current working directory must be a GitHub-connected repository (the `gh` CLI must resolve a repo).
- Fetch **OPEN** issues only by default. Adjust only if the user explicitly asks for closed or all issues.
- Each issue analysis MUST run as a spawn of the `github-issue-analyser` sub-agent. Sub-agents always start with a clean context window. Do NOT inline the analysis logic in this skill.
- Use the `Agent` tool with `subagent_type: "github-issue-analyser"`. The agent definition lives at `.claude/agents/github-issue-analyser.md`.
- One output file per issue at `./issues/issue-<issue-number>.md` (project root, folder name `issues`).

## Workflow

Follow these phases in order. Use `TaskCreate` to track progress: one task for "Fetch issues", one task per issue dispatched, and one task for "Verify outputs".

### Phase 1 — Preflight

Verify the environment before doing anything else:

```bash
gh --version
gh repo view --json nameWithOwner -q .nameWithOwner
pwd
```

If `gh` is missing or the repo is not detected, stop and report the problem to the user. Do not attempt to fetch issues without a confirmed repo.

Ensure the output directory exists at the project root:

```bash
mkdir -p ./issues
```

Verify the **project map** prerequisite is present. The pipeline pastes the repo's `CLAUDE.md` into every analyser as a shared project map (resolved in Phase 3.5), so its existence is a hard prerequisite checked here — up front, alongside the `gh` and `./issues` checks — not deep into the run:

```bash
test -f CLAUDE.md && echo "CLAUDE.md: present" || echo "CLAUDE.md: MISSING"
```

If `CLAUDE.md` is **missing**, do not proceed to fetch issues. Run the onboarding process to generate it by invoking the **`/init`** skill (via the `Skill` tool with `skill: "init"`), which writes a `CLAUDE.md` documenting the codebase. Once `/init` completes and `CLAUDE.md` exists, continue with the workflow. If the user declines onboarding, stop — the pipeline cannot run without a project map.

### Phase 2 — Fetch Open Issues

Fetch the open issues with the fields the analyser needs:

```bash
gh issue list \
  --state open \
  --limit 100 \
  --json number,title,state,author,labels,assignees,milestone,createdAt,updatedAt,url,body,comments
```

Notes:
- Default to `--state open`. If the user explicitly asks for closed or all issues, adjust the flag accordingly.
- If the user supplies filters (`label`, `assignee`, `search`), forward them to `gh`.
- If the result is empty, report "No open issues found" and stop — do not create empty analysis files.
- If the list is very large (> 30), report the count to the user and ask for confirmation before spawning agents. Suggest filtering by label if they want to narrow it.

### Phase 3 — Resolve the Absolute Output Path

Sub-agents cannot reliably resolve relative paths against the parent's cwd. Resolve the absolute project path once:

```bash
echo "$(pwd)/issues"
```

Use this absolute path in every spawn prompt below.

### Phase 3.5 — Resolve the Project Map

Each `github-issue-analyser` would otherwise re-onboard to the repo from scratch (mapping layout, reading the README/manifests, learning conventions) — paying that generic cost once per issue. Instead, resolve a single shared **project map** here, sourced from the repo's `CLAUDE.md`, and paste it into every analyser.

Read `CLAUDE.md` from the project root:

```bash
cat CLAUDE.md
```

Capture its text verbatim as the **project map** and reuse it for every spawn in Phase 4. Phase 1 already guaranteed `CLAUDE.md` exists (running `/init` to onboard if it was missing), so here you only read it — no existence check or stop condition is needed at this stage.

> **Note:** the project map is *descriptive*, not authoritative. The analyser is instructed to use it as a starting map and to trust the code over the map on any conflict.

### Phase 4 — Spawn `github-issue-analyser` Sub-Agents (one per issue)

For each issue, spawn a separate `github-issue-analyser` sub-agent via the `Agent` tool. Each spawn starts with a clean context window — no parent conversation, no other issues, no prior analyses.

**Parallelism:** Issues are independent. Dispatch multiple `Agent` calls in parallel by emitting multiple tool-use blocks in the same assistant message. Cap parallelism at **5 concurrent agents** to avoid resource pressure. For larger backlogs, dispatch in sequential waves of 5.

**`Agent` tool call shape** — one call per issue:

- `description`: `"Analyse issue #<N>"`
- `subagent_type`: `"github-issue-analyser"`
- `prompt`: the template below, fully substituted
- `run_in_background`: omit (run foreground) so the wave completes before verification

**Prompt template** — keep it minimal; the agent's system prompt owns the workflow:

```
Repository: <OWNER/NAME>
Repository root (absolute): <ABSOLUTE_PROJECT_ROOT>
Issue number: <N>
Output path (absolute): <ABSOLUTE_PROJECT_ROOT>/issues/issue-<N>.md

Project map (from CLAUDE.md — shared, descriptive, NOT authoritative; do NOT re-derive generic repo facts, but trust the code over this map on any conflict):
<CLAUDE_MD_TEXT_FROM_PHASE_3.5>

Issue payload (JSON):
<COMPACT_JSON_FOR_THIS_ONE_ISSUE>
```

Embed the JSON for **exactly one** issue per spawn — never the whole list. Compact (no pretty-print) to keep the prompt tight. Paste the **same** `<CLAUDE_MD_TEXT_FROM_PHASE_3.5>` into every spawn.

Track each dispatch with `TaskUpdate` (set the per-issue task to `in_progress` before the wave, `completed` after the wave returns).

### Phase 5 — Verify Outputs

After all waves complete:

```bash
ls -1 ./issues/
```

Confirm one `issue-<N>.md` exists per dispatched issue. For any missing file, re-dispatch that one issue once. If it still fails, surface the failure to the user with the issue number and the agent's last response.

### Phase 6 — Summarise to User

Output a concise summary:
- Repository and total issue count analysed
- Output directory (`./issues/`)
- A short list of the generated files as `issues/issue-<N>.md — <issue title>` (truncate if > 20)
- Any failures, by issue number

Do not repeat the analyses themselves — they live on disk.

## Important Constraints

- Do NOT analyse issues directly in this skill's context. All analysis happens inside spawned `github-issue-analyser` sub-agents.
- Do NOT share state between sub-agents. Each one only sees its own issue payload — that is the point of the clean context window.
- Do NOT pass the full issue list to any single sub-agent. One issue per spawn.
- Do NOT modify repository files other than writing to `./issues/`. `CLAUDE.md` is read-only here — the pipeline consumes it, it does not write it.
- Do NOT delete or overwrite existing `issue-<N>.md` files without confirming with the user first. If a file already exists, ask whether to overwrite, skip, or write to `issue-<N>-v2.md`.

## Resources

- The `github-issue-analyser` agent definition (system prompt + tool allowlist) at `.claude/agents/github-issue-analyser.md`. The orchestrator only needs to invoke it correctly; the agent owns the analysis workflow, the codebase-planning step, and the Markdown template. It consumes the shared project map (from `CLAUDE.md`, resolved in Phase 3.5) instead of re-onboarding.
- The repo's `CLAUDE.md` is the source of the shared project map (Phase 3.5). The pipeline reads it as-is and assumes it exists and is current.
- `references/gh-issue-fields.md` — reference for the JSON fields returned by `gh issue list` and how the analyser consumes them.
