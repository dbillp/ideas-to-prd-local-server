# System Prompt: Local Planning Server
**Version:** 0.3.0
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
- `.agent/rules/` and `.agent/skills/` scaffolded by wizard
- EARS format enforced on acceptance criteria field

### `[MLP]` — Daily driver quality
- OKF v0.2 frontmatter enforcement
- Seven QC Tools as display/view actions on list view
- Optional classification fields (de Bono, feasibility, business value)
- References field linking items

### `[PMF]` — Post-validation additions
- Multi-project support
- Objectives and business requirements as structured documents
- SRS / TDD folder scaffolding
- Extended `.agent/skills/` library (triage, classify, draft-prd)

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
  /.agent/                    ← [MMP] scaffolded at init, populated at MMP build
    /rules/
      domain-definitions.md   ← project domain names and their purpose
      type-definitions.md     ← what each type (Rule, Strategy, etc.) means
      decimal-addressing.md   ← how the x.x.x addressing system works
      ears-format.md          ← EARS patterns and when to use each
      item-stages.md          ← Idea / Discussion / PRD rules and guardrails
    /skills/
      convert-to-ears.md      ← rewrite human text into EARS format for review
      triage-idea.md          ← [MLP] de Bono triage playbook
      draft-prd.md            ← [MLP] scaffold a PRD from a promoted Idea
      classify-item.md        ← [MLP] assign decimal address to an item
  .git/
```

> **`.agent/` folder at MVP:** The folder and file stubs are created by the wizard at init time so the structure is in place. Content is written during the MMP build. Stub files contain a header and a `# TODO` placeholder only.

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
| `address` | Optional | Optional | Required | Decimal address `x.x.x` — `0` at any level = undefined |
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
address: "1.3.1"
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

_Each criterion must be written in EARS format. One statement per line.
Agent can assist conversion — human reviews and approves before entry._

- WHEN a member attempts to access the analysis feature, IF their email is not verified, THEN the server SHALL redirect them to the email verification screen and display 'Please verify your email to continue'.
- WHEN a member submits email verification, IF the token is valid, THEN the server SHALL grant access to the analysis feature within 2 seconds.
- IF the verification token has expired, THEN the server SHALL display 'Verification link expired' and offer to resend.

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

## Decimal Addressing System `[MMP]`

Every item has a unique three-level decimal address: `[domain].[type].[sequence]`

### Level definitions

| Level | Position | Range | Meaning |
|---|---|---|---|
| Domain | `x` | 0–7 | Which of the 7 domains. 0 = unassigned |
| Type | `x.x` | 0–9 | What kind of item. 0 = untyped |
| Sequence | `x.x.x` | 0–n | Item number within that coordinate. 0 = draft/unsequenced |

### The `0` rule

`0` at any level means **undefined at that level** — a progressive placeholder. Once a real number is assigned, the `0` is replaced. A fully classified item has no zeros.

```
0.0.0   → completely undefined (raw capture, nothing decided)
1.0.0   → domain-1, type not yet decided
1.3.0   → domain-1, Rule type, not yet sequenced
1.3.7   → domain-1, Rule type, item 7 (fully addressed)
```

### Type number map (second level)

```
0 = undefined
1 = Strategy
2 = Story
3 = Rule
4 = Constraint
5 = Composite Workflow
6 = Gap
7 = Glossary
8 = Architecture
9 = Concept
```

### Why this is AI-readable

An AI agent can filter by coordinate without parsing any prose:

- `*.3.*` → all Rule-type items across all domains
- `1.*.*` → everything in domain-1
- `*.0.*` → everything not yet typed — a triage queue
- `*.*.0` → all drafts not yet formally sequenced

### Address storage

The decimal address is stored as a dedicated frontmatter field:

```yaml
address: "1.3.7"
```

Stage is **not** part of the decimal — it lives in frontmatter as `stage: prd`. The decimal encodes domain and type only. Sequence is assigned by the server at save time when the item is formally placed.

---

## EARS — Requirements Syntax `[MMP]`

All acceptance criteria in PRD items must be written in EARS (Easy Approach to Requirements Syntax) format. EARS constrains free-form natural language into five testable patterns that both humans and AI can parse unambiguously.

### The five patterns

| Pattern | Keyword | Template |
|---|---|---|
| Ubiquitous | *(none)* | `The <system> SHALL <response>` |
| Event-driven | WHEN | `WHEN <trigger>, the <system> SHALL <response>` |
| State-driven | WHILE | `WHILE <precondition>, the <system> SHALL <response>` |
| Optional feature | WHERE | `WHERE <feature is included>, the <system> SHALL <response>` |
| Unwanted behaviour | IF / THEN | `IF <trigger>, THEN the <system> SHALL <response>` |

**Combined (complex):**
```
WHILE <precondition>, WHEN <trigger>, the <system> SHALL <response>
```

### Examples

| Human text (vague) | EARS equivalent (testable) |
|---|---|
| "The system should handle errors gracefully" | `IF an invalid domain is selected, THEN the server SHALL display 'invalid domain' and block submission` |
| "MoSCoW must be required on PRDs" | `WHEN the PRD form is submitted, IF the MoSCoW field is empty, THEN the server SHALL block submission and display 'MoSCoW classification is required'` |
| "Files should never be deleted" | `The server SHALL NOT delete any file from the project folder or archive` |
| "Acceptance criteria must be EARS" | `WHILE a PRD form is active, WHEN the user submits, IF any acceptance criterion is not in EARS format, THEN the server SHALL highlight the field and display the relevant EARS pattern` |

### EARS limitations

EARS is not used for:
- Non-functional requirements expressed as metrics (e.g. `page load ≤ 2.5s`) — use a dedicated NFR field
- Architectural constraints — use `type: constraint` items
- Requirements needing decision tables or state diagrams

### Agent-assisted EARS conversion `[MMP]`

The `.agent/skills/convert-to-ears.md` skill enables the following workflow:

1. Human enters rough requirement text in plain language
2. Human invokes the agent with the text
3. Agent loads `convert-to-ears` skill, identifies the correct EARS pattern, rewrites the text
4. Human reviews the EARS output, edits if needed, approves
5. Approved EARS text is pasted into the acceptance criteria field by the human
6. The server stores it exactly as entered — no further transformation

> The agent proposes. The human decides. The server stores.

### .agent rules files — always loaded `[MMP]`

Short, invariant, project-scoped. Loaded at the start of every agent session.

| File | Contents |
|---|---|
| `domain-definitions.md` | The 7 domain names for this project and what each covers |
| `type-definitions.md` | What each type (Rule, Strategy, Concept, etc.) means in this project |
| `decimal-addressing.md` | The x.x.x system: level definitions, type number map, the 0 rule |
| `ears-format.md` | All five EARS patterns with examples and limitations |
| `item-stages.md` | Idea / Discussion / PRD stage rules, guardrails, and field requirements |

### .agent skills files — loaded on demand `[MMP/MLP]`

Loaded only when the agent determines the task matches the skill description.

| File | Scope | Purpose |
|---|---|---|
| `convert-to-ears.md` | `[MMP]` | Rewrite human requirement text into EARS format for human review |
| `triage-idea.md` | `[MLP]` | Walk an Idea through de Bono 8-category triage |
| `draft-prd.md` | `[MLP]` | Scaffold a PRD skeleton from a promoted Idea |
| `classify-item.md` | `[MLP]` | Assign or suggest a decimal address for an unaddressed item |

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
| 17 | Acceptance criteria | Textarea | Yes | One EARS statement per line. Agent can assist conversion. Human approves before entry. |
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
| SRS / TDD folder scaffolding | PMF |
| Business requirements as a structured document | PMF |
| Objectives as a structured (non-freeform) document | PMF |
| `release_target` spanning multiple stages | MMP review |
| EARS validation in form — enforce pattern server-side or advisory only | MMP decision |
| Decimal address auto-assignment vs. human-assigned | MMP decision |

---

## Changelog

- v0.3.0 (2026-09-23) — Added decimal addressing system (x.x.x); added EARS requirements syntax section with all 5 patterns, examples, and limitations; added .agent rules and skills file specifications; moved .agent scaffolding from PMF to MMP; updated acceptance criteria field to require EARS format; added `address` field to metadata model; updated PRD file example with EARS acceptance criteria
- v0.2.0 (2026-09-23) — OKF v0.2 full spec added; MoSCoW ruling on Discussion stage; acceptance criteria added as required PRD field; archive subfolders differentiated; all deferred items tagged by release stage
- v0.1.0 (2026-09-23) — initial draft
