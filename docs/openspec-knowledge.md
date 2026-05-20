# OpenSpec Knowledge Reference

> Compiled from the Fission-AI/OpenSpec repository documentation.  
> Source: https://github.com/Fission-AI/OpenSpec

---

## What OpenSpec Is

OpenSpec is a **spec-driven development (SDD)** tool for AI coding assistants. Its core job is adding a lightweight spec layer so teams agree on what to build before any code is written.

Philosophy:
- Fluid not rigid — no phase gates
- Iterative not waterfall — update artifacts as you learn
- Easy not complex — minimal ceremony
- Brownfield-first — designed for existing codebases, not just greenfield

---

## Directory Structure

```
openspec/
├── config.yaml                      # Project configuration
├── specs/                           # SOURCE OF TRUTH
│   ├── auth/
│   │   └── spec.md
│   ├── payments/
│   │   └── spec.md
│   └── ui/
│       └── spec.md
├── changes/                         # PROPOSED MODIFICATIONS (active)
│   ├── add-dark-mode/
│   │   ├── proposal.md
│   │   ├── design.md
│   │   ├── tasks.md
│   │   ├── .openspec.yaml           # Change metadata (schema, created date)
│   │   └── specs/                   # Delta specs
│   │       └── ui/
│   │           └── spec.md
│   └── archive/                     # COMPLETED CHANGES
│       └── 2025-01-24-add-dark-mode/
│           └── (same structure)
└── schemas/                         # Custom workflow schemas (optional)
    └── my-workflow/
        ├── schema.yaml
        └── templates/
```

**Active changes** = direct subdirectories of `openspec/changes/` that are NOT inside `archive/`.  
**Archived changes** = `openspec/changes/archive/YYYY-MM-DD-<name>/`.

---

## The Two Layers

### openspec/specs/ — Source of Truth

Describes how the system **currently** behaves. Organized by domain. Updated only when changes are archived (delta specs merge in).

**Spec file format:**
```markdown
# Auth Specification

## Purpose
Authentication and session management.

## Requirements

### Requirement: User Authentication
The system SHALL issue a JWT token upon successful login.

#### Scenario: Valid credentials
- GIVEN a user with valid credentials
- WHEN the user submits the login form
- THEN a JWT token is returned
- AND the user is redirected to the dashboard

#### Scenario: Invalid credentials
- GIVEN invalid credentials
- WHEN the user submits the login form
- THEN an error message is displayed
- AND no token is issued

### Requirement: Session Expiration
The system MUST expire sessions after 30 minutes of inactivity.
```

RFC 2119 keywords: `MUST`/`SHALL` = absolute requirement, `SHOULD` = recommended, `MAY` = optional.  
Specs are **behavior contracts**, NOT implementation plans. They should not mention class names, libraries, or execution steps.

### openspec/changes/ — Proposed Modifications

Each change is a self-contained folder. The folder is the unit of work. Multiple changes can exist in parallel.

---

## Artifacts (spec-driven schema — the default)

Artifact dependency DAG:
```
proposal
   ├──► specs    (requires: proposal)
   └──► design   (requires: proposal)
         └──► tasks (requires: specs + design)
              └──► APPLY (implementation)
```

Dependencies are **enablers**, not gates. You can skip design if not needed. State = filesystem existence.

### proposal.md
```markdown
# Proposal: Add Dark Mode

## Intent
Users have requested dark mode to reduce eye strain.

## Scope
In scope: theme toggle, system preference detection, localStorage persistence
Out of scope: custom color themes

## Approach
CSS custom properties + React context for state management.
```

### specs/ (Delta Specs)
Delta specs describe **what's changing** relative to current specs — not the full spec. Lives at `specs/<domain>/spec.md` mirroring `openspec/specs/<domain>/spec.md`.

```markdown
# Delta for UI

## ADDED Requirements

### Requirement: Theme Selection
The system MUST support light and dark themes.

#### Scenario: Default theme
- GIVEN first-time user
- WHEN page loads
- THEN system preference is detected and applied

## MODIFIED Requirements

### Requirement: Accessibility
The system MUST meet WCAG AA contrast in both themes.
(Previously: light theme only)

## REMOVED Requirements

### Requirement: High Contrast Mode
(Superseded by the theme system)
```

Archive behavior:
- `## ADDED` → appended to main spec
- `## MODIFIED` → replaces existing requirement
- `## REMOVED` → deleted from main spec

### design.md
```markdown
# Design: Add Dark Mode

## Technical Approach
Theme state via React Context. CSS custom properties for runtime switching.

## Architecture Decisions

### Decision: Context over Redux
Using React Context because: simple binary state, no complex transitions.

## Data Flow
ThemeProvider → ThemeToggle ↔ localStorage → CSS Variables (:root)

## File Changes
- `src/contexts/ThemeContext.tsx` (new)
- `src/components/ThemeToggle.tsx` (new)
- `src/styles/globals.css` (modified)
```

**The `## File Changes` section** is the most machine-readable, concrete artifact for drift detection.

### tasks.md
```markdown
# Tasks

## 1. Theme Infrastructure
- [ ] 1.1 Create ThemeContext with light/dark state
- [x] 1.2 Add CSS custom properties for colors
- [ ] 1.3 Implement localStorage persistence

## 2. UI Components
- [ ] 2.1 Create ThemeToggle component
```

Task completion is tracked by GFM checkboxes: `- [x]` = done, `- [ ]` = not done.

### .openspec.yaml
Change metadata file:
```yaml
schema: spec-driven
created: 2025-01-20
```

---

## Archive Process

When `/opsx:archive` runs:
1. Delta specs merge into `openspec/specs/` (ADDED/MODIFIED/REMOVED applied)
2. Change folder moves to `openspec/changes/archive/YYYY-MM-DD-<name>/`
3. All artifacts preserved for audit trail

After archive, `openspec/specs/` reflects the updated system state.

---

## Project Configuration (openspec/config.yaml)

```yaml
schema: spec-driven         # Default schema for new changes

context: |                  # Injected into ALL artifact prompts
  Tech stack: TypeScript, React, Node.js
  Testing: Vitest

rules:                      # Per-artifact rules
  proposal:
    - Include rollback plan
  specs:
    - Use Given/When/Then format
```

Schema resolution order: CLI flag > change's `.openspec.yaml` > `config.yaml` > default (`spec-driven`).

---

## OPSX Commands (the standard workflow)

### Core profile (default install)

| Command | What it does |
|---------|-------------|
| `/opsx:propose` | Create change folder + all planning artifacts in one step |
| `/opsx:explore` | Think through ideas before committing |
| `/opsx:apply` | Implement tasks, checking them off |
| `/opsx:sync` | Merge delta specs into main specs (optional, archive handles it) |
| `/opsx:archive` | Complete change, merge deltas, move to archive |

### Expanded profile (opt-in)

| Command | What it does |
|---------|-------------|
| `/opsx:new` | Scaffold change folder only, no artifacts |
| `/opsx:continue` | Create one artifact at a time (respects dependencies) |
| `/opsx:ff` | Fast-forward: create all planning artifacts at once |
| `/opsx:verify` | Validate implementation matches artifacts (completeness/correctness/coherence) |
| `/opsx:bulk-archive` | Archive multiple completed changes, resolves spec conflicts |
| `/opsx:onboard` | Guided tutorial through complete workflow |

### `/opsx:verify` — what it checks

Runs without making changes. Key checks:

| Dimension | What it validates |
|-----------|------------------|
| Completeness | All tasks done, all spec requirements have code evidence |
| Correctness | Implementation matches spec intent, edge cases handled |
| Coherence | Design decisions reflected in code, patterns consistent |

Reports CRITICAL / WARNING / SUGGESTION. Does NOT block archive.

---

## OpenSpec CLI (programmatic use)

The `openspec` CLI has JSON output for scripting/CI. All agent-compatible commands accept `--json`.

```bash
# List all changes (active + archived)
openspec list --json

# Artifact dependency graph status for a change
openspec status --change add-dark-mode --json
# Returns: { changeName, schemaName, isComplete, artifacts: [{id, outputPath, status, missingDeps}] }

# Structural validation
openspec validate --all --json
# Returns: { results: { changes: [{name, valid, warnings}] }, summary }

# Read change content
openspec show add-dark-mode --json
openspec show add-dark-mode --json --deltas-only    # delta specs only

# Read spec content  
openspec show auth --type spec --json
openspec show auth --type spec --json --requirements  # requirements only, no scenarios
```

**`openspec validate` checks structural issues** (missing sections, malformed format). It does NOT check implementation drift.

---

## Drift Scenarios

There are multiple distinct drift scenarios in an OpenSpec project. A CI guard needs to handle them differently.

---

### 1. Branch-level drift (per-PR, spec→code)

**What it is:** Active changes in the PR branch specify what files should be modified (`design.md > File Changes`, delta spec domains, task mentions). The actual git diff of the branch doesn't match.

**Direction:** Spec claims to change files X, Y, Z — but branch only changes A, B.

**How to detect:**
- Parse `design.md > ## File Changes` for explicit paths
- Parse delta spec domain paths (e.g., `specs/auth/` → implies `src/auth/` related code)
- Parse `tasks.md` items for file mentions
- Compare against `git diff --name-only <merge-base>...HEAD`

**Signal:** Files in specs not touched in branch (spec gap), files in branch not mentioned in any spec (undocumented change).

**Context:** The specs being compared are the **branch's version** of `openspec/changes/`, not main.

---

### 2. Source-of-truth drift (scheduled, spec→code)

**What it is:** `openspec/specs/` describes system behavior, but the codebase has diverged over time. Requirements describe behavior that no longer exists or works differently.

**Direction:** Main specs say the system does X — but codebase no longer does X.

**How to detect (heuristic):**
- Extract file path references from spec text
- Extract identifier references (backtick content, code blocks)
- Check if those files/identifiers still exist in the codebase
- Extract MUST/SHALL requirements, check if related test files have corresponding test coverage

**Signal:** Stale specs pointing to deleted/renamed files or removed behavior.

**Context:** Best run on schedule or on push to main, not per-PR. High false-positive rate in brownfield.

---

### 3. Reverse drift / code-first (code→spec)

**What it is:** Code has been written but the corresponding specs haven't been updated. The developer implemented something that isn't captured in `openspec/specs/`. Useful when teams want to use code as the source and backfill specs.

**Direction:** Code has behavior X — but no spec describes X.

**How to detect:**
- Identify code patterns (API routes, exported functions, components) that have no spec coverage
- Look at files changed in a PR — do any changed modules have corresponding spec entries?
- Flag large code additions with no matching spec artifact

**Use case:** Team wants to write code first, then auto-update or suggest spec updates. The CI guard reports what code needs spec coverage, and can suggest or generate delta spec stubs.

---

### 4. Orphaned active changes

**What it is:** An active change folder exists but the code has already been merged/deployed without archiving. The change is "done in practice" but still shows as active.

**How to detect:**
- All tasks in `tasks.md` are checked `[x]`
- The `design.md > ## File Changes` files all exist and have been modified recently
- But the change is not in `archive/`

**Signal:** Stale active changes that should be archived but weren't.

---

### 5. Inter-change conflict

**What it is:** Two active changes have overlapping delta specs for the same domain. They may conflict with each other when archived.

**How to detect:**
- Find active changes that both have `specs/<same-domain>/spec.md`
- Check if they touch the same `## Requirement:` names
- Report potential merge conflict before archive

**Signal:** Parallel changes that will conflict at archive time.

---

### 6. Artifact completeness gaps

**What it is:** An active change is missing expected artifacts based on its schema, OR artifacts exist but are effectively empty/incomplete.

**How to detect:**
- Use `openspec status --change <name> --json` to get artifact graph
- Check for artifacts with `status: blocked` or missing entirely
- Check file sizes / section headers for completeness

---

## Key Implications for a CI Guard

1. **The openspec CLI already handles structural validation** (`openspec validate --all --json`). A CI guard should call this rather than re-implement it.

2. **`openspec status --change <name> --json`** provides the artifact dependency graph — who's blocked, what's done, what's missing.

3. **`design.md ## File Changes`** is the most reliable, explicit source for what code a change is supposed to touch. Always check this first for branch-level drift.

4. **Delta specs use ADDED/MODIFIED/REMOVED sections** — parsing them gives the semantic intent of the change.

5. **Tasks are the only executable checklist** — `tasks.md` checkboxes are the ground truth for implementation progress.

6. **Branch context matters** — for per-PR checks, read specs from the branch, not from main.

7. **Reverse drift (code→spec) is a valid maintenance scenario** — teams may want to push code changes and have specs suggested or auto-generated, not just blocked.

8. **`/opsx:verify` is the AI-native equivalent** of what a CI guard does programmatically. The CI guard should produce compatible output format.

---

## Schemas

Schemas define the artifact types and their dependency graph. Custom schemas live in `openspec/schemas/<name>/` (project-local, versioned) or `~/.local/share/openspec/schemas/` (user global).

```yaml
# openspec/schemas/spec-driven/schema.yaml
name: spec-driven
artifacts:
  - id: proposal
    generates: proposal.md
    requires: []
  - id: specs
    generates: specs/**/*.md
    requires: [proposal]
  - id: design
    generates: design.md
    requires: [proposal]
  - id: tasks
    generates: tasks.md
    requires: [specs, design]
```

Custom schemas can add artifacts (e.g., `research.md` before `proposal.md`), remove them, or reorder dependencies. A CI guard targeting custom schemas should read `openspec status --json` to know what artifacts are expected rather than hardcoding `spec-driven` assumptions.

---

## Legacy Commands (still work, not recommended)

| Command | What it does |
|---------|-------------|
| `/openspec:proposal` | Create ALL artifacts at once (no dependency graph) |
| `/openspec:apply` | Implement the change |
| `/openspec:archive` | Archive the change |

Legacy changes are compatible with OPSX commands and have the same folder structure.
