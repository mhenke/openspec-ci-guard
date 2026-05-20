# openspec-ci-guard — Design Principles

> These principles define what the tool is for, what it is not, and why specific design choices were made. They should be referenced when evaluating features, resolving trade-offs, and deciding what to exclude.

---

## The Core Problem

OpenSpec creates a discipline for spec-driven development — but nothing closes the loop. Code changes without spec updates. Specs become stale. Active changes accumulate without archiving. The gap between spec and reality grows silently until it becomes too expensive to fix.

The openspec-ci-guard exists to close that gap — continuously, with low friction, and without becoming a bureaucratic bottleneck.

---

## Principle 1: Visibility over enforcement

**The tool surfaces drift. It does not block by default.**

Mandatory enforcement on every merge creates friction that teams will work around. A blocked PR creates pressure to skip the spec, not update it. The result is more drift, not less.

Instead, the tool makes drift impossible to ignore. Every PR shows a drift report. Every scheduled run shows the health of the source-of-truth specs. Teams see the score and own the decision.

Enforcement (non-zero exit codes, merge blocking) is opt-in, configurable per check type, and graduated by threshold. Teams turn it on when they're ready — not because the tool assumes they are.

**What this means in practice:**
- Default exit code is always `0`
- Reports are always generated, regardless of gate config
- Gate configuration is explicit, versioned, and team-owned (`.openspec-ci.yml`)

---

## Principle 2: No drift without governance

**Drift is always visible. Silent accumulation is not allowed.**

Drift that isn't surfaced is drift that compounds. The tool's job is to make the current state of spec health legible at all times:

- How many active changes have incomplete artifacts?
- How far has the source-of-truth spec drifted from the codebase?
- Which completed changes were never archived?
- Which parallel changes will conflict at archive time?

This isn't enforcement — it's accountability. The team governs the response. The tool governs the visibility.

**What this means in practice:**
- All 6 drift scenarios are always checked and always reported
- Drift scores appear in PR comments, job summaries, and badge output
- There is no "quiet mode" that hides findings

---

## Principle 3: Specs must evolve with code — in both directions

**Spec drift isn't just code outrunning specs. It's also specs outrunning code.**

There are two drift directions, both equally problematic:

**Forward drift (spec → code):** A spec describes behavior the code no longer implements. The spec is a lie.

**Reverse drift (code → spec):** Code implements behavior the spec doesn't describe. The spec is incomplete.

The tool handles both. It does not privilege one direction as "correct." When code moves first — in a hotfix, emergency deploy, or brownfield migration — the tool helps retrofit the spec, not just penalizes the gap.

Reverse drift detection includes a `--suggest-stubs` mode that generates a minimal delta spec scaffold for uncovered code changes, making it easy to write the spec after the fact.

**What this means in practice:**
- `drift --type branch` detects forward drift (spec claims files X, Y, Z; only A, B were changed)
- `drift --type reverse` detects reverse drift (files changed with no corresponding spec coverage)
- `--suggest-stubs` lowers the cost of retrofitting specs
- Neither direction triggers a hard failure by default

---

## Principle 4: Maintenance responsibility must be explicit

**A spec with no owner goes stale. The tool makes ownership implicit in the workflow.**

When a developer creates an active change (`openspec/changes/<name>/`), they own that change's lifecycle through archive. The CI guard surfaces this responsibility at the right moments:

- PR comment shows incomplete artifacts in the branch's active changes
- Orphaned change detection flags changes where all tasks are done but archive never ran
- Stale artifact detection flags active changes that have been idle past a configurable threshold

Responsibility doesn't require assignment — it emerges from the change author. The tool makes it visible without creating a formal review gate or approval workflow.

**What this means in practice:**
- Orphaned change detection is always enabled
- Archive step is surfaced in PR comments when changes look complete
- No formal "spec owner" assignment is required

---

## Principle 5: Lessen friction for simple changes

**Not every change needs full spec ceremony. The tool knows the difference.**

A hotfix that fixes a null pointer dereference does not need a proposal.md, design.md, and tasks.md. A small enhancement that adds a single UI option does not need a multi-scenario delta spec. Requiring the same ceremony for all changes produces one outcome: teams stop using specs for small changes, and those small changes collectively become the largest source of drift.

The tool recognizes change complexity and adjusts expectations:

| Change type | Expected spec | Drift check approach |
|------------|---------------|---------------------|
| Hotfix / critical bug | `tasks.md` with one or two items | Check tasks only; no artifact completeness requirement |
| Bug fix | Minimal: `proposal.md` + `tasks.md` | Check artifact existence, not completeness |
| Small enhancement | `proposal.md` + delta spec stub | Check for spec presence; stub is acceptable |
| Feature | Full: proposal, specs, design, tasks | Full artifact completeness + drift check |
| Cross-cutting / breaking change | Full + review | Same as feature, flag for review |

Change size is inferred from `git diff --stat` line count and number of changed files. Teams can also tag changes explicitly via `.openspec.yaml` `type:` field.

**What this means in practice:**
- Hotfixes and bug fixes have a fast path with reduced artifact requirements
- The `--suggest-stubs` flag generates the minimum viable delta spec for any uncovered change
- Completeness thresholds are configurable by inferred change type in `.openspec-ci.yml`

---

## Principle 6: Archive is closure — incomplete archive is drift

**An unarchived completed change is as problematic as a missing spec.**

Delta specs inside active changes are not merged into `openspec/specs/` until archive runs. A change where all tasks are done, code is merged, but `archive` was never run leaves:

- Source-of-truth specs that don't reflect current behavior
- Active changes folder cluttered with stale entries
- Misleading artifact status for future `openspec status` queries

Archive is the moment of closure. It is not optional housekeeping. The tool treats orphaned completed changes (all tasks done, not archived) as a first-class drift signal — equivalent to a spec with no code coverage.

**What this means in practice:**
- Orphaned change detection runs on every PR and on schedule
- PR comments surface archive reminders when changes appear complete
- The skills include `/opsx.ci.check` which explicitly calls out un-archived completed changes
- `drift --type sot` always reflects delta specs that were never merged, making orphaned changes visible in the source-of-truth drift score

---

## Principle 7: No reviewer bottleneck

**Spec review should run parallel to code review — never be the gate that holds up a merge.**

Requiring spec review approval before merging creates a two-stage review process. Reviewers become spec approvers. The PR queue backs up. Developers learn that writing specs means longer cycle times.

The tool avoids this by design:

- Spec compliance is reported, not enforced, by default
- Hard gates are opt-in and configurable per check type
- The tool does not create or require spec review workflows, approvals, or sign-offs
- Drift reporting is informational — the team decides what to act on

Teams that want spec review as a gate can add it independently (e.g., GitHub CODEOWNERS on `openspec/` + branch protection rules). The tool does not impose it.

**What this means in practice:**
- No spec-approval workflow is built into the tool
- PR comments are informational, not blocking
- Drift scores inform team retrospectives, not merge decisions (unless explicitly gated)
- The tool integrates with existing PR review flows without adding a new one

---

## What this tool is not

**Not a spec generator.** The tool analyzes and reports on specs. It generates delta spec stubs as a convenience, but the content of specs is always human- or AI-authored, not produced by the tool.

**Not a spec enforcer.** Enforcement is opt-in. The tool's default posture is visibility, not blocking.

**Not a replacement for OpenSpec.** The tool wraps and extends OpenSpec's existing `openspec validate` and `openspec status` CLI commands. It does not reimplement OpenSpec's workflow, archive logic, or artifact dependency graph.

**Not a reviewer.** The tool does not assess spec quality, correctness, or completeness beyond structural checks. It does not know whether a requirement is well-written or whether a scenario is realistic.

**Not an assignment system.** The tool does not assign spec ownership, create tickets, or route work. Ownership is implicit in the change author.

---

## Trade-off reference

| Tension | Resolution |
|---------|-----------|
| Close the gap vs. don't enforce | Surface every gap; let teams gate on their terms |
| Full ceremony vs. fast path for small changes | Infer change type; adjust required artifacts accordingly |
| Visibility vs. noise | Report all findings; let `.openspec-ci.yml` configure what surfaces as warnings vs. failures |
| Archive discipline vs. dev speed | Orphaned change is a warning, not a block; archive reminder in PR comment |
| Spec-first vs. code-first | Both are valid; reverse drift detection + stub suggestions support code-first |
| Governance vs. no bottleneck | Governance = visibility + accountability; not a new approval workflow |
