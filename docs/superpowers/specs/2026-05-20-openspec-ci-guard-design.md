# openspec-ci-guard — Design Spec

**Date:** 2026-05-20  
**Status:** Approved  
**Repo:** https://github.com/mhenke/openspec-ci-guard

---

## Problem

OpenSpec creates spec-driven development artifacts — but nothing closes the loop. Code changes without spec updates. Specs become stale. Active changes accumulate without archiving. The gap between spec and reality grows silently.

spec-kit-ci-guard solved this for Spec Kit. This project solves it for OpenSpec's OPSX workflow.

---

## Solution

A TypeScript npm package (`openspec-ci-guard`) that wraps and extends OpenSpec's existing CLI to surface six categories of drift — continuously, with low friction, without blocking merges by default.

Three delivery formats sharing a single core engine:

1. **CLI** (`openspec-ci-guard`) — programmatic, scriptable, CI-agnostic
2. **GitHub Action** — thin wrapper around the CLI for PR comments and job summaries
3. **Claude Code skills** — markdown command files for AI-native local checks

---

## Target Workflow

OPSX only (`core` profile and expanded profile). Legacy `/openspec:*` commands are structurally compatible and handled automatically since they share the same folder structure.

---

## Design Principles

See `docs/design-principles.md`. Summary:

- **Visibility over enforcement** — exit 0 by default, gates opt-in via `.openspec-ci.yml`
- **No drift without governance** — all 6 scenarios always reported, no quiet mode
- **Both drift directions** — spec→code AND code→spec are first-class
- **Maintenance responsibility implicit** — change author owns lifecycle through archive
- **Lessen friction for simple changes** — hotfixes/bugfixes get reduced artifact requirements
- **Archive is closure** — orphaned completed changes are drift, not housekeeping
- **No reviewer bottleneck** — informational by default, no approval workflow built in

---

## Architecture

### Single package, CLI-first

One npm package. The CLI is the ground truth for all checks. The GitHub Action wraps it. The Claude Code skills mirror its checks for AI-native use.

```
openspec-ci-guard/
├── src/
│   ├── checkers/
│   │   ├── change-scanner.ts       # find active changes (not in archive/)
│   │   ├── artifact-checker.ts     # wraps openspec validate + openspec status
│   │   ├── task-checker.ts         # parse tasks.md GFM checkboxes
│   │   ├── branch-drift.ts         # check #1: branch spec vs git diff
│   │   ├── sot-drift.ts            # check #2: openspec/specs/ vs codebase
│   │   ├── reverse-drift.ts        # check #3: code changes with no spec coverage
│   │   ├── orphan-checker.ts       # check #4: completed but not archived
│   │   └── conflict-checker.ts     # check #5: parallel changes touching same domain
│   ├── reporters/
│   │   ├── markdown.ts             # human-readable tables for PR comments
│   │   └── json.ts                 # machine-readable for CI pipelines
│   ├── stubs/
│   │   └── delta-spec-stub.ts      # generate minimal delta spec scaffolds
│   ├── config.ts                   # .openspec-ci.yml reader and defaults
│   ├── git.ts                      # git diff helpers (merge-base, changed files)
│   ├── openspec-cli.ts             # wrapper around openspec validate/status/show
│   ├── types.ts                    # shared interfaces
│   └── cli.ts                      # Commander.js entrypoint
├── action/
│   ├── action.yml                  # GitHub Action metadata
│   └── index.js                    # calls CLI, posts PR comment
├── skills/
│   ├── opsx.ci.check.md
│   ├── opsx.ci.report.md
│   ├── opsx.ci.badge.md
│   └── opsx.ci.gate.md
├── skill.yml                       # Claude Code extension manifest
├── package.json
├── tsconfig.json
└── vitest.config.ts
```

### Key dependency

`openspec` CLI (`@fission-ai/openspec`) must be installed in the project. The guard calls:
- `openspec validate --all --json` — structural validation (artifact completeness)
- `openspec status --change <name> --json` — artifact dependency graph per change
- `openspec list --json` — enumerate active and archived changes
- `openspec show <name> --json --deltas-only` — read delta spec content

The guard does NOT reimplement OpenSpec's artifact graph engine, archive logic, or schema resolution.

---

## The Six Checks

### Check 1 — Branch-level drift (spec → code, per-PR)

**What:** The branch's own active change specs claim certain files will be touched. The actual `git diff` of the branch doesn't match.

**When:** Every PR via GitHub Action; `/opsx.ci.drift --type branch` in skills.

**How:**
1. Identify active changes in the branch (`openspec/changes/*/` not in `archive/`)
2. For each active change, extract claimed file scope in priority order:
   - `design.md > ## File Changes` section (explicit paths — most reliable)
   - Delta spec domain paths (`specs/auth/` → implies `src/auth/**` area)
   - `tasks.md` task text (backtick file mentions)
3. Get `git diff --name-only $(git merge-base HEAD origin/main)...HEAD`
4. Score two directions:
   - **Spec gap:** spec-claimed files not in the diff
   - **Undocumented change:** diff files not mentioned in any active change spec

**Output:**
```
## Branch Drift — add-dark-mode

Spec-claimed files not touched in this branch:
  ⚠ src/contexts/ThemeContext.tsx  (from design.md > File Changes)
  ⚠ src/styles/globals.css         (from design.md > File Changes)

Files changed with no spec coverage:
  ⚠ src/utils/color-helpers.ts
  ⚠ src/utils/contrast.ts

Drift score: 40%  (2/5 spec files untouched, 2 undocumented changes)
```

---

### Check 2 — Source-of-truth drift (spec → code, scheduled)

**What:** `openspec/specs/` describes system behavior the codebase no longer reflects.

**When:** Scheduled (weekly) or on push to main. Not per-PR — too noisy for brownfield.

**How (heuristic — no LLM required):**
1. Walk `openspec/specs/**/*.md`
2. **Also factor in un-archived delta specs:** for each orphaned active change (all tasks done, not archived), treat its `specs/` deltas as pending changes to the SoT that haven't merged yet. Report these as a separate "pending deltas" section — they inflate effective SoT drift because the real system state includes those changes but the SoT specs don't.
3. Extract file path references from specs (patterns matching `src/`, `lib/`, `tests/`, etc.)
4. Extract identifier references (backtick content, code blocks)
5. For each reference: check filesystem existence + grep codebase
6. Score: unresolvable references / total references = drift %

**Output:**
```
## Source-of-Truth Drift Report

| Domain   | References | Resolvable | Drift |
|----------|-----------|------------|-------|
| auth/    | 12        | 10         | 17%   |
| payments/| 18        | 13         | 28%   |
| ui/      | 8         | 8          | 0%    |

Overall SoT Drift: 20%

Unresolvable references:
  payments/spec.md: `src/payments/legacy-processor.ts` — file not found
  auth/spec.md: `validateSessionToken()` — not found in codebase
```

---

### Check 3 — Reverse drift / code-first (code → spec, per-PR)

**What:** Files changed in the branch have no corresponding spec coverage in either active changes or the source-of-truth specs.

**When:** Every PR. Supports brownfield teams who write code first and retrofit specs.

**How:**
1. Get changed files in branch
2. For each changed file, check spec coverage:
   - Does an active change's `specs/` or `design.md > File Changes` mention this file?
   - Does `openspec/specs/` have a domain that covers this file's path area?
3. Report uncovered files
4. With `--suggest-stubs`: generate a minimal delta spec scaffold for uncovered files

**Stub output (--suggest-stubs):**
```markdown
# Delta for [inferred-domain]

## ADDED Requirements

### Requirement: [TODO — describe behavior added in src/payments/refund.ts]
The system MUST ...

#### Scenario: [TODO]
- GIVEN ...
- WHEN ...
- THEN ...
```

---

### Check 4 — Orphaned and stale active changes

Two sub-checks with different triggers:

**4a — Orphaned (completed, not archived):** All tasks `[x]`, change still active. Delta specs never merged into source-of-truth.

**4b — Stale (incomplete, idle):** Tasks still incomplete but the change folder hasn't been modified in `max_idle_days` (default: 30). Detected via `git log -- openspec/changes/<name>/`. A change nobody has touched in a month is accumulating maintenance debt.

**When:** Every PR + scheduled.

**How:**
1. For each active change, parse `tasks.md` checkboxes
2. Get last git modification date for the change folder
3. If all tasks `[x]` and not archived → **orphaned**
4. If any tasks `[ ]` and last modification > `max_idle_days` → **stale**

**Output:**
```
## Change Maintenance

Orphaned (completed but not archived):
  ⚠ add-dark-mode  — 8/8 tasks complete, not archived for 12 days
    → Run: openspec archive add-dark-mode

Stale (incomplete, idle):
  ⚠ fix-payments   — 2/5 tasks done, no activity for 34 days
    → Resume, reassign, or close this change
```

---

### Check 5 — Inter-change conflict

**What:** Two active changes both have delta specs for the same domain. They will conflict when archived.

**When:** Every PR + on push to main.

**How:**
1. For each active change, collect delta spec domain paths (e.g., `specs/auth/`, `specs/ui/`)
2. Find pairs of changes that share one or more domains
3. Check if they modify the same `### Requirement:` names
4. Report conflicts with severity (same requirement = ERROR, same domain only = WARN)

**Output:**
```
## Inter-Change Conflicts

  ⚠ WARN: add-dark-mode and update-footer both touch specs/ui/
    → No requirement-level conflict detected — safe to archive in order

  ✗ ERROR: add-2fa and update-session-timeout both modify
    "Requirement: Session Expiration" in specs/auth/
    → Will conflict at archive time — resolve before merging
```

---

### Check 6 — Artifact completeness

**What:** Active changes with missing or blocked artifacts based on their schema.

**When:** Every PR.

**How:** Calls `openspec validate --all --json` and `openspec status --change <name> --json` for each active change. Surfaces missing artifacts, structural warnings, and blocked dependencies.

**Change type classification** — used to determine required artifacts:

| Source | How it sets type |
|--------|-----------------|
| `.openspec.yaml type:` field | Explicit — highest precedence |
| `git diff --stat` size | Inferred fallback (< 50 lines, < 5 files = `simple`) |

| Type | Required artifacts | Recommended |
|------|--------------------|-------------|
| `hotfix` | `tasks.md` only | `proposal.md` |
| `bugfix` | `proposal.md` + `tasks.md` | `design.md` |
| `simple` (inferred) | `tasks.md` only | `proposal.md` |
| `enhancement` / default | `proposal.md` + `specs/` + `tasks.md` | `design.md` |
| `feature` | All four artifacts required | — |

Setting `type: hotfix` in a change's `.openspec.yaml` overrides inferred size entirely.

**Output:**
```
## Artifact Completeness

  add-dark-mode       ✅ All artifacts present (proposal, specs, design, tasks)
  fix-auth-bug        ⚠ design.md missing "Technical Approach" section
  update-footer       ℹ tasks.md missing (simple change — not required)
```

---

## CLI Interface

```
openspec-ci-guard check              # full per-PR suite: branch drift, reverse drift,
                                     # artifact completeness, orphaned changes,
                                     # inter-change conflicts
openspec-ci-guard check --full       # adds source-of-truth drift (slower; use on schedule)
openspec-ci-guard report             # full traceability across all checks
openspec-ci-guard badge              # generate spec compliance badge for README
openspec-ci-guard gate               # write/update .openspec-ci.yml
```

`check` is the primary command. It runs all checks appropriate for a PR context.  
`check --full` adds SoT drift for scheduled/main-push runs. No separate `drift` subcommand — callers do not need to know which internal checks map to which contexts.

**Global flags:**
- `--format markdown|json` (default: markdown)
- `--suggest-stubs` (generate delta spec scaffolds for uncovered code changes)
- `--root <path>` (default: cwd, for monorepo support)
- `--base <branch>` (default: origin/main, for merge-base calculation)

**Exit codes:**
- `0` — always; reports are generated regardless of gate configuration. Gates only change exit code, never suppress output.
- `1` — only when a gate is configured with `fail: true` and its threshold is exceeded

---

## Configuration (.openspec-ci.yml)

Should be committed to the repository so gate configuration is versioned and team-owned.

```yaml
version: "1"
mode: report            # report | gate
                        # report: always exits 0, reports all findings
                        # gate: exits 1 when a check with fail:true exceeds its threshold
                        # either mode always generates full reports — mode only affects exit code

change_size:
  simple_max_lines: 50  # changes under this are "simple" (reduced artifact reqs)
  simple_max_files: 5

# Override required artifacts per change type.
# Defaults shown. Teams can loosen or tighten per type.
artifact_requirements:
  hotfix:   [tasks]
  bugfix:   [proposal, tasks]
  simple:   [tasks]
  enhancement: [proposal, specs, tasks]
  feature:  [proposal, specs, design, tasks]

gates:
  artifact_completeness:
    fail: false
    warn: true

  task_completion:
    fail: false
    warn: true
    threshold: 80         # warn if < 80% complete

  branch_drift:
    fail: false
    warn: true
    threshold: 30         # warn if drift score > 30%

  sot_drift:
    fail: false
    warn: true
    threshold: 25

  reverse_drift:
    fail: false
    warn: true
    suggest_stubs: true   # auto-suggest delta spec stubs in PR comment

  orphaned_changes:
    fail: false
    warn: true
    max_completed_days: 7   # warn if ALL tasks done but not archived for > 7 days

  stale_changes:
    fail: false
    warn: true
    max_idle_days: 30       # warn if ANY tasks incomplete and change untouched for > 30 days
                            # uses git log to detect last modification to the change folder

  inter_change_conflicts:
    fail: false             # requirement-level conflicts → fail
    warn: true              # domain-level overlaps → warn
```

---

## GitHub Action

Single job. The action detects its own execution context (PR vs schedule vs push to main) and runs the appropriate check set automatically. Consumers do not configure which checks to run — the tool decides based on context.

| Context | Checks run | Output |
|---------|-----------|--------|
| `pull_request` | `check` (branch drift, reverse drift, artifact completeness, orphaned, conflicts) | PR comment |
| `push` to default branch | `check --full` (adds SoT drift) | Job summary |
| `schedule` | `check --full` | Job summary |
| Manual dispatch | `check --full` | Job summary |

**Note on SoT drift and the "always visible" principle:** SoT drift does not run per-PR because it scans the full codebase and is too noisy for brownfield projects on every commit. Instead it runs on push to main and on schedule — ensuring it is always visible on a regular cadence, not silently skipped. This is consistent with Principle 2 ("drift is always visible") because teams see it at a defined, governed frequency rather than never.

```yaml
# .github/workflows/openspec-ci.yml (example for consumers)
name: OpenSpec CI Guard

on:
  pull_request:
  push:
    branches: [main]
  schedule:
    - cron: '0 9 * * 1'  # weekly Monday 9am

jobs:
  openspec-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: mhenke/openspec-ci-guard@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          suggest-stubs: true
```

**action.yml inputs:**

| Input | Default | Description |
|-------|---------|-------------|
| `token` | required | GitHub token for PR comments and job summaries |
| `format` | `markdown` | Report format |
| `suggest-stubs` | `false` | Generate delta spec stubs for reverse drift |
| `base-branch` | `main` | Branch to diff against |
| `config-file` | `.openspec-ci.yml` | Config file path |

---

## Badge

`openspec-ci-guard badge` generates a spec health badge for README display, driven by the last `check` run's results.

```
openspec-ci-guard badge              # print badge markdown for README
openspec-ci-guard badge --format url # print shield.io URL only
```

**Output (markdown):**
```markdown
![Spec Health](https://img.shields.io/badge/spec_health-92%25-brightgreen?style=flat-square)
![Tasks](https://img.shields.io/badge/tasks-14%2F14-brightgreen?style=flat-square)
![Drift](https://img.shields.io/badge/drift-8%25-green?style=flat-square)
```

Badges are generated from local check results — no external service required. Color thresholds match the configured gate thresholds in `.openspec-ci.yml`.

Claude Code skill: `/opsx.ci.badge` — generates badge markdown and instructions for adding to README.

---

## Claude Code Skills

Four skills installed to `.claude/skills/` (via `skill.yml`):

| Skill | Purpose |
|-------|---------|
| `/opsx.ci.check` | Run all checks appropriate for the current context. Interprets results, suggests next actions (archive, retrofit spec, resolve conflict), and explains what each finding means — not a pass-through to the CLI |
| `/opsx.ci.report` | Full traceability report with AI-assisted interpretation: which findings are highest priority, what the drift pattern indicates, suggested remediation order |
| `/opsx.ci.badge` | Generate spec health badge markdown for README |
| `/opsx.ci.gate` | View or configure `.openspec-ci.yml` — explains trade-offs of each gate threshold |

Skills call the CLI when available (`npx openspec-ci-guard`). When the CLI is not installed, skills perform equivalent checks by reading OpenSpec files directly. Either way, skills add genuine AI-native value — interpretation, prioritization, and actionable next steps — not just formatted CLI output.

---

## Data Flow

```
Git repo
  │
  ├── openspec/changes/*/           → change-scanner.ts
  │     ├── proposal.md
  │     ├── design.md               → branch-drift.ts (File Changes section)
  │     ├── tasks.md                → task-checker.ts (checkboxes)
  │     └── specs/**/*.md           → branch-drift.ts (domain paths)
  │                                 → conflict-checker.ts (requirement names)
  │
  ├── openspec/specs/**/*.md        → sot-drift.ts (reference extraction)
  │
  └── git diff (merge-base...HEAD)  → branch-drift.ts (check 1)
                                    → reverse-drift.ts (check 3)

openspec CLI
  ├── validate --all --json         → artifact-checker.ts (check 6)
  └── status --change <n> --json   → artifact-checker.ts (blocked deps)

All checkers → reporters → CLI stdout / GitHub PR comment / job summary
```

---

## Error Handling

- If `openspec` CLI is not installed: artifact completeness check is skipped; all other checks use direct file parsing. Warning emitted.
- If `openspec/` directory doesn't exist: all checks pass silently with a notice (not an error — project may not use OpenSpec yet).
- If git is not available: branch-level and reverse drift checks are skipped with a warning.
- Malformed `tasks.md` (no checkboxes): task completion = unknown, reported as such.
- Malformed `design.md` (no File Changes section): branch drift falls back to domain-path heuristic.

---

## Testing Strategy

- Unit tests for each checker (vitest) using fixture OpenSpec directories
- Fixture set covers: complete change, partial artifacts, all tasks done, conflicting domains, no openspec/ dir
- Integration test: full CLI run against a real fixture repo with known drift scores
- No snapshot tests — output format is intentionally stable and human-readable

---

## Technology

| Concern | Choice | Reason |
|---------|--------|--------|
| Language | TypeScript | Matches OpenSpec's own codebase |
| CLI framework | Commander.js | Lightweight, no opinion on config |
| Test runner | Vitest | Fast, matches OpenSpec's own test setup |
| Build | tsup | Zero-config ESM + CJS dual output |
| GitHub Action | JS action (not Docker) | Faster startup, no container pull |
| Runtime | Node.js 20+ | Matches OpenSpec's requirement |

No runtime dependencies for core checkers beyond Node.js built-ins. `commander` for CLI. `@actions/core` + `@actions/github` in action bundle only.

---

## Out of Scope (v1)

- Spec quality assessment (well-written requirements, good scenarios) — requires LLM
- Automatic spec merging or archiving — tool is read-only by default
- Formal spec review workflow or approval gates — by design principle
- Workspace support — OpenSpec workspaces are in beta and explicitly unsupported for automation
- Custom schema support beyond `spec-driven` — v2 when schema diversity exists in the wild
