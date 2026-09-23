# System Prompt: Local Planning Server
**Version:** 0.2.0
**Date:** 2026-09-23
**Status:** Draft — MVP scope ready for implementation

---

## Purpose

Build a local server that provides a web form for capturing structured planning items and storing them as Markdown files using a consistent, versioned folder structure. The system is **human-driven**. The server enforces structure mechanically through form fields and file conventions. No AI autonomy over classification, renaming, or restructuring.

---

## Core Principles

1. **One item at a time.** The form handles one item per submission. No bulk operations.
2. **The tool enforces, the human decides.** Classification, priority, and stage are chosen by the human. The server only validates required fields and generates the file.
3. **Never delete.** Files are archived on change, never removed.
4. **Raw content is sacred.** Discussion/Concept files are immutable once written. Never edited, never archived.
5. **Small, atomic, bounded.** Each item is scoped to what fits in its stage. No monoliths.
6. **Git-first.** Every project folder is a git repository from creation.
7. **Active memory is bounded by design.** The server reads only the files needed for the current operation. It never loads the entire project into memory.

---

## Tech Stack

- **Server:** Node.js + Express
- **Front end:** Plain HTML + CSS (no framework)
- **Storage:** Local filesystem (markdown files)
- **Version control:** Git (auto-initialized per project)
- **Port:** 3000 (localhost only, no external access)
- **Dependencies:** Minimal. No database. No ORM. Files are the database.

---

## Release Scope

### `[MVP]` — Core loop only
- Project wizard: creates folder structure + git init
- Idea capture form: minimal fields, saves markdown file
- Server runs locally, no auth

### `[MMP]` — Full planning tool
- All three item stages: Idea, Discussion, PRD
- Stage-aware forms (fields change per stage)
- Archive on PRD edit with semantic versioning
- Changelog auto-updated on save
- Basic list view per domain

### `[MLP]` — Daily driver quality
- OKF v0.2 frontmatter enforcement
- Seven QC Tools as display/view actions on list view
- Optional classification fields (de Bono, feasibility, business value)
- References field linking items

### `[PMF]` — Post-validation additions
- Multi-project support
- Objectives and business requirements as structured documents
- `.agent/rules/` and `.agent/skills/` scaffolding
- SRS / TDD folder scaffolding

---

## Project Wizard `[MVP]`

Runs once to initialize a project. Cannot be re-run on the same folder. If the planning framework version changes, a new folder is created — the old folder is preserved exactly as-is.

### Wizard steps

| Step | Field | Required | Notes |
|---|---|---|---|
| 1 | Project name | Yes | Sanitized to lowercase kebab-case. Used as folder name. |
| 2 | Project description | Yes | One paragraph max. Written into `objectives.md`. |
| 3 | Domain names | Yes | Up to 7. Defaults to `domain-1` through `domain-7` if left blank. |
| 4 | Git remote URL | No | If provided, sets remote origin after `git init`. |

### Rules
- Wizard blocks if the target folder already exists
- Project name is auto-sanitized: lowercase, spaces to hyphens, special chars stripped
- `objectives.md` is the only freeform document generated at init time
- All domain log files and changelogs are created empty with headers only

### Output folder structure

```
/<project-name>/
  objectives.md
  /ideas/
    idea_log_<domain>.md      ← one per domain, created empty
    README.md
  /chat/                      ← Discussion/Concept files (immutable)
    README.md
  /prds/
    /<domain>/                ← one subfolder per domain
  /changelog/
    CHANGELOG_<domain>.md     ← one per domain, created empty
  /archive/
    /chat/                    ← never populated (chat files are immutable)
    /prds/                    ← archived PRD versions land here
  /templates/
    idea_template.md
    prd_template.md
    changelog_template.md
  .git/
```

---

## Item Stages `[MVP]` Idea only / `[MMP]` Discussion + PRD

Every item belongs to exactly one stage. Stages are independent — an Idea does not become a Discussion or PRD. They coexist as separate files.

| Stage | Folder | Purpose | Guardrails |
|---|---|---|---|
| **Idea** `[MVP]` | `/ideas/idea_log_<domain>.md` | Seed capture | Single statement. Core essence only. No implementation detail. Max 280 chars. |
| **Discussion** `[MMP]` | `/chat/<DOMAIN>-concept-<title>.md` | Raw dialogue | Unstructured. Immutable once saved. Never edited, never archived. Dated for time context. |
| **PRD** `[MMP]` | `/prds/<domain>/<Domain>_PRD_v<semver>_<YYYY-MM-DD>.md` | Formal requirement | Full metadata. Versioned. Archived on change. |

---

## Item Metadata Model

### Scope per stage and release

| Field | Idea `[MVP]` | Discussion `[MMP]` | PRD `[MMP]` | Values |
|---|---|---|---|---|
| `id` | Auto | Auto | Auto | `IDEA-<DOMAIN>-<seq>`, `CHAT-<DOMAIN>-<seq>`, `PRD-<DOMAIN>-<seq>` |
| `title` | Auto-generated | Auto-generated | Auto-generated | `<domain>-<type>-<description-slug>` |
| `stage` | Required | Required | Required | `idea` / `discussion` / `prd` |
| `domain` | Required | Required | Required | One of the 7 project domains |
| `type` | Required | Required | Required | See type list below |
| `description` | Required | Required | Required | Idea = one sentence max (280 chars) |
| `date_created` | Required | Required | Required | ISO 8601 + Unix timestamp (both stored) |
| `date_modified` | — | — | Required | Auto-updated on each save |
| `moscow` | Optional | Omitted | Required | `must` / `should` / `could` / `wont` |
| `release_target` | Optional | Omitted | Required | `MVP` / `MMP` / `MLP` / `Scale` / `PMF` |
| `origin` | Optional | — | Required | `sensed` / `derived` / `imagined` |
| `innovation_type` | Optional | — | Optional | `incremental` / `radical` / `process` |
| `references` | Optional | Optional | Optional | Array of item IDs |
| `version` | — | — | Required | Semantic version `major.minor.patch` |
| `status` | Required | — | Required | See status values below |

> **MoSCoW on Discussion:** Omitted entirely. Discussion items are raw, unstructured dialogue — forcing a priority classification contradicts the immutable, unedited nature of Concept files.

> **`innovation_type`:** Only shown/valid when `origin = imagined`.

### Type values

`strategy` / `story` / `rule` / `constraint` / `composite-workflow` / `gap` / `glossary` / `architecture` / `concept`

- **`concept`** is the only valid type for Discussion stage
- All other types are valid for Idea and PRD stages

### Status values

**Idea:** `raw` → `triaged` → `promoted` → `parked` → `rejected`

**PRD:** `draft` → `active` → `superseded` → `archived`

### Origin semantics (Cartesian)

- **`sensed`** *(Adventitious)*: From empirical observation, real-world feedback, or direct experience
- **`derived`** *(Derivative)*: Logically deduced from an existing rule, feature, or constraint
- **`imagined`** *(Factitious)*: Novel, speculative, or blue-sky. Unlocks `innovation_type`

### Innovation type (only when origin = imagined)

- **`incremental`**: Refinement within an existing paradigm
- **`radical`**: Paradigm shift, structural pivot, new model
- **`process`**: Operational workflow, governance pipeline, or certification gate

---

## Title Auto-Generation `[MVP]`

Title is derived from three form fields — never manually editable:

```
<domain>-<type>-<description-slug>
```

- `description-slug`: description text → lowercase kebab-case, truncated to 60 characters
- Generated live as the human types (real-time preview, read-only field)

**Example:**
domain = `core`, type = `rule`, description = `Members must verify email before accessing analysis`
→ `core-rule-members-must-verify-email-before-accessing-analysis`

---

## File Output Format

### Idea entry `[MVP]`

Appended to the **top** of `/ideas/idea_log_<domain>.md`:

```markdown
---
id: IDEA-CORE-013
title: "core-rule-members-must-verify-email-before-accessing-analysis"
stage: idea
domain: core
type: rule
origin: derived
status: raw
date_created: 2026-09-23T10:45:00Z
date_unix: 1758627900
references: []
---

Members must verify their email address before they can access any analysis feature.

---
```

### Discussion file `[MMP]`

One file per Discussion. Written once, **never modified**:

```markdown
---
id: CHAT-CORE-001
title: "core-concept-email-verification-discussion"
stage: discussion
domain: core
type: concept
date_created: 2026-09-23T10:45:00Z
date_unix: 1758627900
references: []
---

# CORE — Email Verification Discussion

<raw content exactly as entered — never modified after save>
```

### PRD file `[MMP]`

Filename: `<Domain>_PRD_v<semver>_<YYYY-MM-DD>.md`
Stored at: `/prds/<domain>/`

```markdown
---
id: PRD-CORE-001
title: "core-rule-members-must-verify-email-before-accessing-analysis"
stage: prd
domain: core
type: rule
version: 0.1.0
status: draft
moscow: must
release_target: MVP
origin: derived
date_created: 2026-09-23
date_modified: 2026-09-23
date_unix: 1758627900
references: []
---

# CORE — Members Must Verify Email PRD v0.1.0

## Problem statement

## Goals

## Non-goals

## Users / personas

## User stories

## Requirements

### Must-have (this version)
### Should-have (near-term)
### Won't-have (this version)

## Acceptance criteria

_Measurable, testable conditions that confirm this requirement is satisfied.
No vague language — quantify everything (e.g., "email verified within 24h" not "email verified quickly")._

## Success metrics

## Open questions

## Changelog

- v0.1.0 (2026-09-23) — initial draft
```

---

## OKF v0.2 Frontmatter `[MLP]`

When MLP scope is reached, all files gain OKF v0.2 compliant frontmatter fields:

```yaml
okf_version: "0.2"
trust_tier: unverified        # unverified | machine-confirmed | human-reviewed
generated: false              # true if AI-assisted content was used
verified: false               # true only after human explicitly marks reviewed
status_okf: draft             # draft | research | active | deprecated
stale_after: ""               # optional ISO date — after this date, verified resets to false
```

### OKF trust tier definitions

| Tier | Meaning |
|---|---|
| `unverified` | Initial capture — not yet reviewed |
| `machine-confirmed` | AI-synthesized or cross-referenced — awaiting human review |
| `human-reviewed` | Explicitly validated by a human — canonical |

### OKF knowledge lifecycle (5 stages)

| Stage | `status_okf` | `trust_tier` | `verified` |
|---|---|---|---|
| Research & Observation | `research` | `unverified` or `machine-confirmed` | `false` |
| Addendum (Pending Validation) | `draft` | `human-reviewed` | `false` |
| Canonical Theory (Active) | `active` | `human-reviewed` | `true` |
| Practical Overlay | `active` | `human-reviewed` | `true` |
| Deprecated | `deprecated` | any | `false` |

### OKF hard rules

- AI cannot set `verified: true` or `trust_tier: human-reviewed` on content it generated
- If `generated: true` and `verified: true` appear together — this is a constraint violation
- Editing an `active` file automatically resets `verified: false` — forcing human re-review
- `stale_after` is optional. If set, `verified` resets to `false` automatically after that date
- Default on creation: `trust_tier: unverified`, `generated: false`, `verified: false`

---

## Archiving Rules `[MMP]`

### What triggers archiving
Only PRD file edits trigger archiving. Idea and Discussion files are never archived.

### Archive process
1. Before writing the updated PRD, the server copies the current file to `/archive/prds/`
2. Archive filename: `<original-filename>_archived_<unix-timestamp>.md`
3. The new file is written with the incremented version
4. The changelog is auto-updated

### Version bump — human selects one on the edit form

| Bump | When |
|---|---|
| **Patch** `0.0.x` | Clarification, typo fix, wording — no behavior change |
| **Minor** `0.x.0` | New section, new requirement, scope expansion |
| **Major** `x.0.0` | Breaking change, scope redefinition, fundamental restructure |

### Archive folder rules
- `/archive/prds/` — all superseded PRD versions
- `/archive/chat/` — exists but is never populated (Discussion files are immutable, not archived)
- No file in `/archive/` is ever deleted

---

## Semantic Versioning Rules `[MMP]`

Two independent version concepts — never conflated:

| Concept | What it tracks | Example |
|---|---|---|
| **Document version** (`semver`) | Stability of the PRD document itself | `v0.1.0` = early draft, `v1.0.0` = first stable spec |
| **Release target** (`release_target`) | Product maturity stage | `MVP`, `MMP`, `MLP`, `Scale`, `PMF` |

A PRD can be `v1.3.0` and still target `MVP`. The version number describes the document. The release target describes the product.

**Document versioning convention:**
- `0.x.x` — draft / pre-MVP, still shifting
- `1.0.0` — first stable spec, ships as MVP
- Minor bump — new section or requirement added
- Patch bump — clarification, no behavior change

---

## Changelog Rules `[MMP]`

- One changelog per domain: `/changelog/CHANGELOG_<domain>.md`
- Format: [Keep a Changelog](https://keepachangelog.com) — sections: `Added / Changed / Fixed / Deprecated`
- Auto-updated by the server on every PRD save
- Each entry records: version, date, what changed, why (human-entered on edit form)
- The "why" field is required on all PRD edits — blocks submission if empty

---

## Form Behavior

### Idea form `[MVP]`

| # | Field | Type | Required | Notes |
|---|---|---|---|---|
| 1 | Domain | Select | Yes | From project domain list |
| 2 | Type | Select | Yes | Full type list |
| 3 | Description | Text | Yes | 280 char max. One sentence. |
| 4 | Origin | Select | No | `sensed` / `derived` / `imagined` |
| 5 | Innovation type | Select | No | Shown only when origin = `imagined` |
| 6 | Title preview | Read-only | — | Auto-generated live |

### Discussion form `[MMP]`

| # | Field | Type | Required | Notes |
|---|---|---|---|---|
| 1 | Domain | Select | Yes | From project domain list |
| 2 | Title slug | Text | Yes | Used in filename. Kebab-case enforced. |
| 3 | Content | Textarea | Yes | Raw. No formatting enforced. |
| 4 | References | Text | No | Comma-separated item IDs |
| 5 | Title preview | Read-only | — | Auto-generated |

> Once submitted, a Discussion file is locked. No edit form exists for Discussion items.

### PRD form `[MMP]`

| # | Field | Type | Required | Notes |
|---|---|---|---|---|
| 1 | Domain | Select | Yes | |
| 2 | Type | Select | Yes | |
| 3 | Description | Text | Yes | |
| 4 | MoSCoW | Select | Yes | `must` / `should` / `could` / `wont` |
| 5 | Release target | Select | Yes | `MVP` / `MMP` / `MLP` / `Scale` / `PMF` |
| 6 | Origin | Select | Yes | `sensed` / `derived` / `imagined` |
| 7 | Innovation type | Select | No | Shown only when origin = `imagined` |
| 8 | Status | Select | Yes | `draft` / `active` / `superseded` / `archived` |
| 9 | References | Text | No | |
| 10 | Title preview | Read-only | — | Auto-generated |
| 11 | Problem statement | Textarea | Yes | |
| 12 | Goals | Textarea | Yes | |
| 13 | Non-goals | Textarea | No | |
| 14 | Users / personas | Textarea | No | |
| 15 | User stories | Textarea | No | |
| 16 | Requirements | Textarea | Yes | |
| 17 | Acceptance criteria | Textarea | Yes | Measurable, testable. No vague language. |
| 18 | Success metrics | Textarea | No | |
| 19 | Open questions | Textarea | No | |

**Edit-only fields (shown only when editing an existing PRD):**

| # | Field | Type | Required | Notes |
|---|---|---|---|---|
| 20 | Version bump | Select | Yes | `patch` / `minor` / `major` |
| 21 | Why this change | Text | Yes | Feeds into changelog entry |

### Form rules (all stages)
- Required fields validated before submission — blocked if empty
- No AI assistance on any form field — all input is human-entered
- On successful save, server returns: file path written, generated title, version (if PRD)
- Form resets after successful submission

---

## Seven QC Tools as View Actions `[MLP]`

Read-only display modes on the list view. Do not modify any file.

| Tool | Display behaviour |
|---|---|
| Affinity Diagram | Group items by shared domain and type into thematic clusters |
| Interrelationship Diagram | Show cause-effect links using the `references` field |
| Tree Diagram | Display items in domain → type → item hierarchy |
| Matrix Diagram | Table: items vs. MoSCoW × release_target |
| Prioritization Matrix | Rank items by feasibility × business value (when tagged) |
| Process Decision Program Chart | Surface items with `type: gap` as risk nodes |
| Activity Network Diagram | Sequence items by release_target and reference dependencies |

---

## What the System Does Not Do

- Does not generate, rewrite, summarize, or reformat content
- Does not make classification decisions
- Does not rename files autonomously
- Does not promote items between stages automatically
- Does not run AI agents over stored files
- Does not connect to external services `[MVP/MMP]`
- Does not support multiple projects simultaneously `[MVP/MMP]`
- Does not load the entire project into memory at any time

---

## Open Items (Confirmed Deferred)

| Item | Deferred to |
|---|---|
| Idea log: append-only per domain vs. one file per idea | Decide before MMP build |
| Full OKF format details beyond v0.2 spec | MLP |
| Markdown structure preference for display (table vs. list) | MLP |
| Edit vs. archive behavior finalization for Ideas | MMP |
| Multiple optional classifications on a single item | MLP |
| Context-driven form fields for optional classification frameworks | MLP |
| Conflicting classification flagging (e.g. de Bono vs. MoSCoW) | MLP |
| User-extensible classification frameworks | PMF |
| Multi-project support | PMF |
| Item relationships beyond the references field | PMF |
| Wizard re-run / migration behavior | PMF |
| `.agent/rules/` and `.agent/skills/` scaffolding | PMF |
| SRS / TDD folder scaffolding | PMF |
| Business requirements as a structured document | PMF |
| Objectives as a structured (non-freeform) document | PMF |
| `release_target` spanning multiple stages | MMP review |

---

## Changelog

- v0.2.0 (2026-09-23) — OKF v0.2 full spec added; MoSCoW ruling on Discussion stage; acceptance criteria added as required PRD field; archive subfolders differentiated; all deferred items tagged by release stage
- v0.1.0 (2026-09-23) — initial draft
