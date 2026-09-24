# Type: Concept
# Domain: CORE
# Title: Plan Server — Independent Design Review (pre-MVP)

**Date:** 2026-09-24
**Status:** Immutable — external review record. Never edit. Never archive.

---

# Plan Server Design Review

**Date:** 2026-09-24
**Review type:** Design and documentation review

## Executive summary

The project is a local, human-operated planning server. A person enters one planning item at a time; the server validates structured fields and writes Markdown files to a project folder initialized with Git. The intended stages are Idea, Discussion, and PRD. The design was shaped by a previous planning attempt that overloaded AI agents with classification, restructuring, and long-context work, leading to fatigue and unreliable adherence to rules.

The strongest design choice is the division of responsibility: people decide meaning and classification; the application enforces input shape and file conventions; AI assistance, if used, is reviewed by a person and is not run by the server. The staged roadmap and write-once Discussion files also support that goal.

The documents are not yet a single, implementation-ready specification. Release boundaries conflict, Idea editing is unresolved despite Ideas being the MVP data, PRD archival can leave stale copies in the active folder, and the identifier/address schemes do not define a common uniqueness model. Input validation, atomic writes, failure recovery, and localhost security also need concrete rules. Resolving the data lifecycle and scope questions first will reduce the chance of recreating the complexity and confusion seen in the sample project.

## Scope reviewed

- Read the plan-server design discussion in [CORE-Concept-plan-server-design-discussion.md](plan-server/plan-context/CORE-Concept-plan-server-design-discussion.md).
- Read [SYSTEM-PROMPT.md](plan-server/SYSTEM-PROMPT.md).
- Lightly scanned the non-xyz structure of Sample-Example-Only and read its [plan/README.md](Sample-Example-Only/plan/README.md) and [plan/objectives.md](Sample-Example-Only/plan/objectives.md).
- No contents from the xyz folder were read.
- This is a documentation review. No implementation code was reviewed, and runtime behavior is therefore not assessed.

## What the design is trying to achieve

1. Create a project folder through a one-time wizard, with objectives, domain-specific planning areas, templates, and Git.
2. Capture small Ideas with minimal friction.
3. Preserve Discussion / Concept files as raw, immutable historical context.
4. Create structured, versioned PRDs with metadata, acceptance criteria, and a changelog.
5. Keep the application local and simple: Node.js, Express, plain HTML and CSS, Markdown files, no database, and no AI execution.
6. Add richer classification, OKF metadata, seven QC views, and agent guidance only in later releases.

The sample project's README describes a much more agent-led process: regular triage, de Bono decisions, promotion, and broader document pipelines. The design discussion explicitly identifies that approach as having failed in actual use. The new system reduces the AI role and narrows capture; this is a sound response. The remaining risk is that the current specification reintroduces a large number of classifications, views, metadata rules, and agent files without sufficiently binding their cost to the later release stages.

## Findings

### High priority: settle before implementation

#### 1. The release boundaries contradict one another

The release table says MVP is the wizard, Idea capture, and Git initialization; Discussion and PRD arrive in MMP. However, the wizard output describes folders for all stages, and says .agent stubs are present at MVP even though the output tree labels .agent as MMP. The concept record also contains an earlier roadmap that places agent scaffolding in PMF, while the current prompt moves it to MMP.

There are lifecycle naming differences too. The discussion refers to MMP/MMR, the prompt uses MMP, and release-target values include Scale even though there is no Scale release scope. PMF is both a target value and a bucket for post-validation additions.

**Why it matters:** implementation scope, wizard output, and acceptance criteria cannot be determined consistently from the documents.

**Improvement:** publish one versioned release matrix. For each stage, list the routes/forms, folders, fields, validation, and files that exist. Define whether MMR is an alias or a typo, and either specify Scale or remove it from release targets. Decide whether MVP creates only its usable structure or also empty future-stage folders.

#### 2. Idea lifecycle and edit behavior are unresolved at the MVP boundary

Ideas are the only MVP item type, but the prompt defers final edit/archive behavior for Ideas to MMP. It also calls for append-at-top Idea logs, says never to delete files, and says files are archived on change. It does not say how to correct an Idea, retract a mistaken capture, merge duplicates, or record a parked/rejected decision without rewriting the log.

The Idea status list includes raw, triaged, promoted, parked, and rejected, while stages are said to be independent rather than transitions. The meaning of promoted is not defined if the Idea remains an Idea and the PRD is a separate item.

**Why it matters:** the first implemented workflow has no complete persistence or correction contract.

**Improvement:** decide and document one MVP policy. A simple option is an append-only event log with a stable Idea ID and explicit correction/supersession entries. If the log is edited in place instead, specify the audit and recovery behavior. Define what promoted means and how a resulting PRD links back to its source Idea.

#### 3. PRD archival can leave old versions looking current

The archive procedure says to copy the current PRD into /archive/prds/, then write a new version. It does not say to move the old PRD out of /prds/<domain>/. Since the filename includes version and date, the new file will usually have a different name; the old one can remain in the active directory beside it and also exist in the archive. The prompt has both superseded and archived statuses but does not define which file receives them or how the list view identifies the current version.

**Why it matters:** users and views can mistake an old PRD for the current specification, and duplicate copies make version history harder to trust.

**Improvement:** define one source of truth for current PRDs. Specify whether an edit moves the prior file to the archive or retains it in place with an explicit superseded status, and whether the archive is a byte-identical snapshot. Define current-version selection and how archived snapshots appear in views. Use a single atomic operation so a failed save cannot leave a partial archive/new-version pair.

#### 4. IDs and decimal addresses have overlapping, undefined numbering

Items receive stage-prefixed IDs such as IDEA-CORE-013 and PRD-CORE-001; PRDs also require a decimal address such as 1.3.1. The address has its own sequence, but the relationship between the two counters is unspecified. Stage is excluded from the address even though Ideas, Discussions, and PRDs coexist independently. The address is optional for Idea and Discussion but required for PRD, although the discussion describes addresses as unique and progressively assigned. The mapping from user-defined domains to numbers 1–7 is not specified.

**Why it matters:** IDs can collide or drift from addresses, and the same address may refer to multiple independent stage files.

**Improvement:** define the ID as the immutable identity and the decimal address as a mutable classification coordinate, or explicitly make both unique keys. Specify numbering scope, collision handling, domain-number mapping, whether reassignment is allowed, and how source/derived items are linked. Decide whether Idea/Discussion addresses are genuinely optional or part of the stated progressive-classification model.

#### 5. File creation, path validation, and failure recovery are underspecified

The wizard accepts a project name, domain names, and an optional Git remote, but only the project name has basic sanitization described. Domain values are used in log and changelog filenames, and domains also determine PRD folders. The specification does not define uniqueness, safe characters, reserved names, path containment, symlink handling, or what happens if initialization fails partway through. It also does not define concurrency behavior for simultaneous submissions, sequence allocation, or writes to a changelog.

**Why it matters:** invalid or hostile input can produce broken paths; parallel requests can produce duplicate IDs or incomplete files; a failed wizard can leave a folder that blocks a retry.

**Improvement:** treat all path components as validated slugs, keep writes inside the selected project root, serialize ID allocation, and define atomic write/rollback behavior. Define what happens when Git initialization or remote setup fails. Use a safe YAML serializer for frontmatter rather than interpolating user text into YAML.

### Medium priority: clarify before the relevant release

#### 6. The forms and metadata model do not fully agree

Metadata fields include stage, status, references, address, and timestamps, but the forms omit some of them without saying whether the server supplies defaults. The Discussion form asks for a title slug, while the general title rule says titles are auto-generated from domain, type, and description and are not editable. Discussion has no description field and its type is fixed to concept, so it needs an explicit exception. The wizard allows up to seven domains and defaults to seven placeholders, but it is unclear whether fewer than seven are valid or how blank slots map to addresses.

There is also a concrete example defect: the Idea example pairs 2026-09-23T10:45:00Z with Unix timestamp 1758627900, which converts to 2025-09-23T11:45:00Z. Examples should be validated mechanically before they become implementation references.

**Improvement:** define a per-stage schema with required, optional, generated, and fixed fields; state the title exception for Discussions; define domain count and mapping; and verify all sample timestamps, IDs, YAML, filenames, and version transitions.

#### 7. EARS is required, but its enforcement contract is undecided

The prompt says acceptance criteria must use EARS and the form behavior says the format is enforced. The deferred-items table still asks whether validation is server-enforced or advisory. The five patterns are described, but there is no precise grammar or rule for combined conditions, multiple criteria on one line, or a criterion that does not fit EARS. The PRD template has no dedicated non-functional requirements field even though the EARS section says NFRs should use one.

**Why it matters:** a form cannot reliably enforce a format that has no agreed validation rules, and vague regex checks could reject good criteria or accept bad ones.

**Improvement:** choose advisory guidance or strict validation before MMP. Define the accepted grammar and examples, preserve the original human wording when conversion is external, and add a structured NFR field or explicitly locate NFRs elsewhere.

#### 8. OKF v0.2 needs an authoritative schema definition

The prompt specifies status_okf, while the sample-study summary describes an OKF field named status. The initial discussion explicitly left the meaning of “Open Knowledge Format” unresolved, but the current prompt labels its fields as an OKF v0.2 specification without naming an authoritative schema or defining required serialization/validation behavior.

**Improvement:** identify the source standard or call this a project-specific profile. Publish a schema and validation examples, including the relationship between the stage-specific status and status_okf, and the rules for generated, verified, trust_tier, and stale_after.

#### 9. The active-memory safeguard targets the wrong layer

The failure described in the concept file was AI context fatigue and unreliable rule-following. The system prompt says the server will not load the entire project into memory, but the server is not the AI agent. The prompt also says the server never runs AI agents over stored files, which is a useful boundary, but it does not control what an external agent loads. Always-loaded rule files, broad list views, and QC views could still encourage large-context workflows.

**Improvement:** keep AI out of the application workflow unless explicitly added later. For any future agent-assisted work, require user-selected files, bounded summaries, and explicit review; make the selected context visible. Do not describe application RAM limits as an AI-memory guarantee.

#### 10. The seven QC views need richer relationship data

The views are read-only, which fits the human-control principle. Their inputs are not fully modeled, though. A generic references array does not indicate whether a link means cause, dependency, duplicate, conflict, or background context. It cannot reliably support an interrelationship diagram or activity network. A type of gap alone does not give a PDPC view risks, mitigations, or branches. The prioritization matrix depends on feasibility and business-value fields that are deferred to MLP.

**Improvement:** defer views until the required data exists, or define typed, directed relationships and the minimum data each view consumes. Keep view outputs read-only and avoid implying that inferred links are human-confirmed facts.

#### 11. Git-first needs a save and sync policy

Git is initialized per project and an optional remote can be configured, but the prompt does not say whether each save is committed, whether users commit manually, whether the server pushes, or how merge conflicts and uncommitted changes are handled. Git history and the separate archive mechanism may record overlapping histories.

**Improvement:** state whether Git is only initialized or is part of each save workflow. Define the remote setup boundary, commit ownership, and relationship between Git history and /archive/. Keep remote credentials out of generated files and logs.

#### 12. Local-only access still needs a concrete security boundary

“Localhost only” and “no auth” are stated, but the bind address is not. A local web service should not accidentally listen on all network interfaces. Raw Markdown may later be rendered in a list view, and form input is used to generate paths and YAML.

**Improvement:** specify loopback-only binding, safe Markdown rendering, request size limits, origin/host checks appropriate for a local form, and path validation. This is especially important if untrusted planning files can be opened or rendered.

### Lower priority: protect simplicity and maintainability

#### 13. The design is growing beyond the original low-friction goal

Decimal addressing, MoSCoW, release targets, origin, innovation type, EARS, OKF, seven QC views, optional business classifications, and agent rules/skills all add conceptual load. Release tags are present, but the wizard structure and metadata tables make it easy to treat later features as immediate requirements.

**Improvement:** keep MVP to the smallest usable loop: create a project, capture one valid Idea, receive a clear save confirmation, and preserve it. Require a clear user need and usable source data before adding each later classification or view. Avoid making empty scaffolding look like a finished feature.

#### 14. Objectives and business requirements are not connected to item behavior

The discussion places objectives and business requirements above PRDs, but the wizard only creates a freeform objectives.md. Structured business requirements are deferred to PMF, and no relationship or traceability rule links an Idea/PRD to an objective.

**Improvement:** decide whether objectives are descriptive context only for MVP or whether items must reference them. Keep business requirements explicitly out of early releases until there is a concrete workflow and a way to maintain links.

## Recommended decision order

1. Freeze the release matrix and confirm MVP contents.
2. Define Idea correction/status behavior and PRD version/archive/current-file behavior.
3. Define stable IDs, domain-number mapping, and the address sequence rules.
4. Write per-stage schemas and validation rules, including title, timestamp, YAML, and reference handling.
5. Specify safe, atomic filesystem operations and wizard recovery behavior.
6. Decide EARS enforcement and the project-specific OKF schema before MMP/MLP work.
7. Defer agent files and QC views until their actual loading and data requirements are clear.

## Overall assessment

The direction is coherent: a small local tool should reduce dependence on agent memory by making capture deterministic and keeping the human in control. The concept discussion also records the failed sample project's lessons clearly. The main weakness is the gap between those principles and operational detail. A short, authoritative MVP contract plus explicit file lifecycle rules would make the design much safer to implement and easier to evolve without repeating the sample project's failure modes.

## Independent suggestions

This section is advisory. These are my design suggestions based on general software and information-management practice; they are not decisions already made in the system prompt or concept discussion. Each is an option to consider, adapt, or reject.

### 1. Make stable identity separate from classification

**What could be done:** Give every item one immutable ID at creation. Treat the decimal address as a classification coordinate that can be assigned or changed independently. Use references to connect related Idea, Discussion, and PRD records.

**How it could work:** Generate the ID once and never derive it from title, domain, stage, or address. Keep a project-level domain-number map so a domain keeps the same number for the life of that planning folder. If an item is reclassified, update its address and record the change rather than changing its identity.

**Why this may fit better:** Domain names, titles, types, and classifications can change. An immutable identity keeps links and history stable. It also avoids treating two numbering systems as competing identifiers.

### 2. Choose an Idea storage model that matches the correction rules

**What could be done:** Consider one file per Idea instead of a growing per-domain log, or keep the log but make it truly append-only.

**How it could work:** With one file per Idea, use the stable ID in the filename and keep current fields in that file; record corrections as new versions or explicit superseding records. With a log, append entries at the end and never rewrite earlier entries. A list view can combine the records without changing them.

**Why this may fit better:** A log that inserts at the top must rewrite existing bytes and grows without bound. Per-item files are easier to read in bounded chunks, link to, archive, and inspect in Git. A true append-only log has a simpler audit story. The choice should be made based on expected volume and the desired correction model, not as an incidental filename decision.

### 3. Keep a small project manifest as the source of project configuration

**What could be done:** Add a machine-readable project manifest with only durable project-level facts.

**How it could work:** Store the planning framework version, project slug, domain names and their numeric mapping, and perhaps schema version in a small file such as project.yml. Generate it once in the wizard, validate it on startup, and avoid re-deriving domain identity from directory names.

**Why this may fit better:** It gives the server one explicit configuration source and makes the seven-domain mapping stable. It can also help identify which framework rules apply to an older project without scanning every document.

### 4. Define three different file operations: create, revise, and supersede

**What could be done:** Describe file lifecycle as a small state model rather than one broad “never delete / archive on change” rule.

**How it could work:** Creation writes a new item. Revision writes a new version and moves or snapshots the prior version according to one rule. Superseding records a decision that an item is no longer current while preserving its history. Use temporary files and atomic rename for writes, and make the active-folder/current-version rule explicit.

**Why this may fit better:** “Never delete” protects history but does not say where the current copy lives. Separate operations make it clear what users see, what Git records, and what the archive contains.

### 5. Keep capture small, and move optional classification to review

**What could be done:** Make the initial Idea form deliberately minimal, then offer a separate review step for optional metadata.

**How it could work:** Capture only the domain, short statement, and automatically generated ID/time. Add origin, innovation type, address, feasibility, business value, or MoSCoW only when a human opens an enrichment or triage action. Always let the user save a valid Idea without completing optional classification.

**Why this may fit better:** It follows the stated low-friction goal and avoids turning an early hypothesis into an apparently precise classification. It also reduces required decisions at the moment of capture.

### 6. Treat the Markdown schema as a versioned contract

**What could be done:** Define each stage's frontmatter in a concise, versioned schema and keep a few canonical examples.

**How it could work:** Specify field type, requiredness, default, allowed values, generated-by-server behavior, and whether the value can change after creation. Validate generated documents against that schema before writing. Include examples for ordinary text, quotes, Unicode, empty optional fields, and references.

**Why this may fit better:** Markdown is easy for people to edit but flexible enough to become inconsistent. A small schema catches malformed YAML and field drift while keeping the files readable and portable.

### 7. Treat relationships as typed links only when a real use case needs them

**What could be done:** Keep simple references for traceability, but do not assume every reference means causality or dependency.

**How it could work:** Initially store a list of stable item IDs. If a view later needs richer semantics, add an explicit relation type such as supports, depends-on, conflicts-with, or derived-from, and define whether each relation is directional.

**Why this may fit better:** It preserves a simple data model for early releases while preventing visualizations from presenting a generic link as a confirmed cause or dependency.

### 8. Keep agent assistance outside the save path

**What could be done:** Continue to treat AI output as a human-reviewed draft and the server as a deterministic recorder.

**How it could work:** If EARS conversion is used, have a person request it in a separate agent workflow, review the proposed text, and paste the approved text into the form. The server stores exactly what was submitted and records no claim that AI-generated text is verified. Avoid automatically loading the whole planning folder into an agent context.

**Why this may fit better:** It preserves the human-driven boundary that addresses the sample failure. It also makes it possible to use or replace an agent without coupling saved project data to a particular model or prompt.

### 9. Make the local security boundary explicit

**What could be done:** Specify safe defaults for a local web server before implementation.

**How it could work:** Bind only to the loopback interface; never accept a project path from an untrusted request; validate all path segments; escape user text when rendering a list; and impose sensible request-size limits. Keep remote setup opt-in and make its effects visible.

**Why this may fit better:** “Local” describes where the user intends to connect, but a server can still accidentally expose a port or render unsafe content. Explicit rules reduce accidental exposure without adding account-management complexity.

### 10. Let Git preserve history without hiding file behavior

**What could be done:** Treat Git and the application archive as complementary, with distinct jobs.

**How it could work:** Use Git for project-wide history and user-controlled sync. Use the archive only for the product's explicit PRD-version workflow, if that workflow still needs a separate folder. State whether the server commits anything; a conservative default is for it to write files and leave commits and pushes to the user.

**Why this may fit better:** Silent commits or pushes can surprise users and complicate recovery. Clear ownership lets Git remain transparent and avoids confusing the archive with the repository's full history.

### 11. Use release gates to keep future features optional

**What could be done:** Require evidence of a working earlier stage before adding the next stage's concepts.

**How it could work:** Complete and use the Idea capture loop first. Before adding Discussion/PRD, confirm the real edit/archive workflow. Before adding EARS enforcement, decide the exact validation contract. Before adding QC views, confirm the required relationship and classification data exists.

**Why this may fit better:** The sample demonstrates that theoretical completeness can create more overhead than value. Small release gates make it easier to learn from actual use and to stop features that do not improve the workflow.

### 12. Define a few failure scenarios alongside the happy path

**What could be done:** For each write operation, document the expected result if it fails at each step.

**How it could work:** Describe what happens when the destination already exists, a disk write fails, Git initialization fails, the browser retries a submission, or two requests request the next ID at once. Decide which outcomes are retriable and how the user can see whether a save succeeded.

**Why this may fit better:** File-based systems are simple when everything succeeds, but partial writes and retries can create duplicates or orphaned archives. Clear failure behavior protects trust in the Markdown files without requiring a database.
