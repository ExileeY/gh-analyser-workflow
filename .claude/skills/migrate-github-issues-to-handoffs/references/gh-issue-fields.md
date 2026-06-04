# `gh issue list` JSON fields used by github-issue-analyser

The orchestrator fetches issues with:

```bash
gh issue list --state open --limit 100 \
  --json number,title,state,author,labels,assignees,milestone,createdAt,updatedAt,url,body,comments
```

Each item in the resulting JSON array has this shape:

| Field         | Type                                | Notes                                              |
| ------------- | ----------------------------------- | -------------------------------------------------- |
| `number`      | int                                 | Used as the filename: `issues/issue-<N>.md`.       |
| `title`       | string                              | Title — used as the document heading.              |
| `state`       | `"OPEN"` \| `"CLOSED"`              | Should be `OPEN` for the default fetch.            |
| `author`      | `{login: string}`                   | Reporter identity.                                 |
| `labels`      | `[{name: string, description: ...}]`| Signals issue type (bug/feature/etc).              |
| `assignees`   | `[{login: string}]`                 | Current owners.                                    |
| `milestone`   | `{title: string, ...}` \| null      | Sprint/release context.                            |
| `createdAt`   | ISO-8601 timestamp                  | Age of the issue.                                  |
| `updatedAt`   | ISO-8601 timestamp                  | Staleness.                                         |
| `url`         | string                              | Link back to the issue.                            |
| `body`        | string (Markdown)                   | Primary content to analyse.                        |
| `comments`    | `[{author, body, createdAt}]`       | Discussion context. Often contains the real spec.  |

## Per-issue payload passed to each sub-agent

The orchestrator extracts a single item from the list above and embeds it as compact JSON in the sub-agent's prompt. Alongside the JSON, each spawn also carries a **shared project map sourced from the repo's `CLAUDE.md`** — resolved once per batch (see SKILL.md Phase 3.5) — so the analyser does not re-onboard to the repo for every issue. The map is descriptive, not authoritative; the pipeline assumes `CLAUDE.md` exists and is current. The sub-agent (`github-issue-analyser`) parses the JSON, uses the map as its starting point (trusting the code over the map on any conflict), reads the issue-specific codebase locally, plans, and writes the analysis Markdown. The sub-agent must NOT call `gh` to fetch additional issues — its scope is one issue only.

## Filter forwarding

If the user supplies arguments like:
- `state=closed` → `--state closed`
- `state=all` → `--state all`
- `label=bug` → `--label bug`
- `assignee=@me` → `--assignee @me`
- `limit=50` → `--limit 50`

forward them on the `gh issue list` call. Default is `--state open --limit 1000`.

## Empty / large-list handling

- Empty list → stop, report "No issues match the filter".
- > 30 issues → confirm with the user before dispatching agents.
- Pagination: `--limit 100` is sufficient for almost any repo. For larger backlogs, paginate via `--search "created:<date-range>"` segments.
