# Init-time merge strategy option (`merge-to-head` | `pull-request`)

`sandcastle init` now asks the user to choose a **merge strategy** — what happens to completed branches at the end of a run. Two entries: `merge-to-head` (default, existing behavior) and `pull-request` (new, pushes to `origin` and opens a GitHub PR).

## Key decisions

**Registry pattern.** `MergeStrategyEntry` mirrors `BacklogManagerEntry` — a name, label, and `templateArgs` record. The strategy selects which variant files are kept during scaffolding; it does not inject Dockerfile or env content.

**Variant-file approach.** Template directories contain side-by-side files suffixed `.merge-to-head` and `.pull-request` (e.g. `main.mts.merge-to-head`, `main.mts.pull-request`). `copyTemplateFiles` keeps the variant matching the selected strategy with the suffix stripped, and skips the other. Non-variant files are copied as-is.

**PR creation runs on the host, not in the agent.** The scaffolded `main.mts.pull-request` runs `gh pr create` on the host with `shellEscape`'d arguments. The agent only emits title and body via structured `<pr-title>` / `<pr-body>` tags. This eliminates shell-injection risk, force-push surface, and `gh` version drift inside the sandbox. The sandbox Dockerfile is unchanged by the merge strategy — `gh` is a host requirement only.

**No force-push.** Push rejection (non-fast-forward) skips the branch with a logged warning. The outer iteration loop is the retry mechanism; no retry primitives exist in the codebase.

**`Closes #N` gated to `github-issues`.** The `CLOSES_LINE` template argument is `Closes #{{ISSUE_ID}}\n\n` for `github-issues` and empty for `beads`. Substituted at scaffold time into `pr-prompt.md`; the inner `{{ISSUE_ID}}` is resolved at runtime via the pr-author agent's `promptArgs` (passed as `ISSUE_ID: issueId` by `main.mts`).

**Abandoned-PR behavior in `simple-loop`.** Lock-until-merged: each iteration checks for open Sandcastle PRs and continues working on the lowest-numbered one rather than picking a new backlog item. The user must merge or close the PR to free the lock. This prefers PR-completeness over priority-responsiveness.

## Idempotency contract

| Existing PR state                        | Action                                                    |
| ---------------------------------------- | --------------------------------------------------------- |
| None                                     | Push, run PR-author agent, `gh pr create`.                |
| `OPEN`                                   | Push only. Skip agent. Log existing URL.                  |
| `CLOSED` (not merged)                    | Skip everything. Log warning.                             |
| `MERGED`                                 | Skip everything. Log warning.                             |
| Branch on remote, no PR                  | Push, run agent, create PR.                               |
| Push rejected (non-FF)                   | Skip with stderr logged. Never force-push.                |
| Branch on remote, PR was deleted by user | Pre-check returns `null`; same as "no PR" — new PR opens. |

## Consequences

- Templates that support merging now have two variant files per merge-sensitive file (e.g. `main.mts.merge-to-head` + `main.mts.pull-request`). Duplication is expected per ADR 0009.
- PR mode requires `gh ≥ 2.4` + `gh auth login` + an `origin` remote + attached HEAD on the host. Pre-flight checks validate these before any agent is spawned.
- Cross-repo / fork PRs are not supported (`origin` == upstream assumed). Branch protection rules (signed commits, required checks) are user-side concerns.
- The `blank` template has no merge phase, so the merge strategy selector is skipped.
