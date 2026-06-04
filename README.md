# gh-analyser-workflow

A Claude Code pipeline for batch-analysing a repository's **open GitHub issues**. It fetches each open issue with the `gh` CLI, spawns one isolated sub-agent per issue, and writes a detailed, codebase-grounded development plan to `issues/issue-<N>.md` — one Markdown document per issue, ready to hand off to a developer.

## Prerequisites

- **[GitHub CLI](https://cli.github.com/) (`gh`)** — installed and authenticated (`gh auth login`). The current working directory must be a GitHub-connected repository (`gh repo view` must resolve it).
- **[Claude Code](https://claude.com/claude-code)** with sub-agent support — the pipeline spawns sub-agents via the `Agent` tool, each running in a clean context window.
- **A current `CLAUDE.md`** at the repo root — the pipeline reads it as the shared project map for every analyser. Its presence is checked during preflight; if it is missing, the pipeline runs `/init` to onboard and generate one before continuing.

## Usage

From inside the target repository, ask Claude Code to analyse the open issues. Any of these phrases triggers the pipeline:

- "Analyse the open GitHub issues in this repo"
- "Run an analysis pass over the open issues"
- "Produce dev plans for open issues"
- "Review the open issues backlog and propose development plans"

By default only **open** issues are analysed. If you want closed or all issues — or want to filter by label, assignee, or search — say so, and the request is forwarded to `gh`.

## How It Works

The pipeline is a two-component orchestration that lives entirely under `.claude/`:

1. **Orchestrator skill** — [`.claude/skills/migrate-github-issues-to-handoffs/SKILL.md`](.claude/skills/migrate-github-issues-to-handoffs/SKILL.md)
   Verifies the environment, fetches open issues with `gh`, reads the repo's `CLAUDE.md` as the shared **project map**, then dispatches the analysers below and verifies the output. It never analyses issues itself.

2. **Issue-analyser agent** — [`.claude/agents/github-issue-analyser.md`](.claude/agents/github-issue-analyser.md)
   Spawned **once per issue**, each in its own clean context window (up to 5 in parallel). It receives the shared project map (so it skips generic onboarding), reads a single issue payload, explores the codebase in read-only planning mode to ground the plan in real files, and writes exactly one Markdown document. It treats the map as descriptive — if the code conflicts with it, the code wins.

The shared project map is the repo's **`CLAUDE.md`**, read as-is. The pipeline assumes it exists and is current, and never modifies it. This replaces the earlier auto-generated "repo digest" step.

## Output

Each analysed issue produces one Markdown file at:

```
./issues/issue-<N>.md
```

where `<N>` is the issue number. Every document includes YAML frontmatter (number, title, state, author, labels, classification, effort) followed by the issue summary, goals, acceptance criteria, a codebase analysis citing real files, a step-by-step development plan, risks, open questions, and an effort estimate.

## Repository Structure

```
.
├── .claude/
│   ├── agents/
│   │   └── github-issue-analyser.md       # per-issue analysis agent
│   └── skills/
│       └── migrate-github-issues-to-handoffs/
│           ├── SKILL.md                    # orchestrator entry point
│           └── references/
│               └── gh-issue-fields.md      # gh issue JSON field reference
├── issues/                                 # generated issue-<N>.md output
└── README.md
```
