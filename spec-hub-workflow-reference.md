# Spec Hub Workflow Reference

A practical reference for setting up and running multi-repo AI-assisted development using the Centralized Spec Hub architecture.

---

## Core Concept

One source of truth for specifications lives **above** all service repos. Each feature flows through five steps: design → cascade → prompt → implement → close. Each AI session is scoped to one repo and one prompt.

---

## Repository Structure

```text
{platform}-specs/                        ← Spec Hub (no source code)
├── 00-architecture-overview.md          ← System map + status table
├── CHANGELOG.md                         ← Prepended after each feature closes
├── WORKFLOW.md                          ← This file (team reference)
├── {02..N}-{service-name}.md            ← One spec per service
├── Features/
│   └── FEATURE-{name}.md               ← Active design files (deleted after close)
└── Prompts/
    └── PROMPT-{service}-{feature}.md   ← Generated, versioned, reusable

service-repos/
├── {service-a}/
│   ├── .github/copilot-instructions.md ← Stack + conventions (auto-loaded)
│   └── src/
├── {service-b}/
│   ├── .github/copilot-instructions.md
│   └── src/
└── {service-c}/
    ├── .github/copilot-instructions.md
    └── src/
```

---

## The Five-Step Workflow

### Step 1 — Design
**Where:** Spec Hub, new session  
**Output:** `Features/FEATURE-{name}.md`

Cover:
- Feature summary and motivation
- Affected services
- Data model (fields, types, constraints)
- API shape (endpoints, methods, auth)
- Open design decisions

The feature file is disposable — you can still change anything here.

---

### Step 2 — Cascade
**Where:** Spec Hub, same session  
**Output:** Updated service specs, each marked `[PENDING]`

For each affected service spec, add:
- New or modified endpoints
- Schema/state changes
- Cross-service contracts (what this service expects from others)
- Mark changes as `[PENDING: FEATURE-{name}]` until implementation is verified

Also update `00-architecture-overview.md` to show the feature as pending.

> Do not prompt until the specs are complete. The prompt is generated from the spec — a vague spec produces a vague prompt.

---

### Step 3 — Prompt
**Where:** Spec Hub, same or new session  
**Output:** `Prompts/PROMPT-{service}-{feature}.md` for each affected service

Each prompt must answer five questions without relying on prior session context:

| Question | What to include |
|---|---|
| What to study | Minimum existing files that establish the pattern |
| What to create | Exact file paths to create or modify |
| What the data looks like | Field names, types, defaults, indexes, constraints |
| What the API looks like | Endpoints, HTTP methods, guards, request/response shapes |
| What to call things | DTO names, slice names, action types, route paths, class names |

Declare dependencies between prompts explicitly:

```markdown
### Dependencies
Apply PROMPT-main-api-{feature}.md and verify it first.
The frontend expects:
- GET /{resource} → { items: Resource[], total: number }
- PATCH /{resource}/:id
```

**Principle: The prompt must be self-contained.** If the agent needs to ask clarifying questions, the prompt is incomplete.

---

### Step 4 — Implement
**Where:** Each service repo, one new session per service  
**Rule:** Apply prompts in dependency order. Do not move to the next service until the current one is verified.

Session rules:

| Situation | Rule |
|---|---|
| Designing + generating prompts | New session in Spec Hub |
| Implementing in service repo A | New session in repo A |
| Implementing in service repo B | New session in repo B |
| Bug fix on current work | Same session |
| Reconciling after implementation | New session in Spec Hub |

---

### Step 5 — Close
**Where:** Spec Hub, new session  
**Actions:**
1. Update each affected service spec with final implementation details
2. Remove all `[PENDING]` markers
3. Note any implementation deviations (schema changes, renamed fields, etc.)
4. Update `00-architecture-overview.md` (mark feature as shipped)
5. Prepend an entry to `CHANGELOG.md`
6. Delete `Features/FEATURE-{name}.md`

**Principle: Reconcile after every implementation, not later.** Deferred reconciliation causes spec drift.

> Keep the feature file only while design is unsettled. Once implementation is absorbed into the service specs, the feature file creates duplicate truth. Delete it.

---

## File Templates

### Service Spec (`{N}-{service-name}.md`)

```markdown
# {Service Name}

## Stack
{Runtime} / {Framework} / {Database} / {Auth method}

## Conventions
- {Convention 1}
- {Convention 2}

## Module Structure
{directory layout}

## Endpoints
| Method | Path | Guard | Description |
|---|---|---|---|
| GET | /{resource} | JWT | Paginated list |

## Schemas
### {Resource}
- _id: String (UUID v4)
- tenant: String (indexed, required)
- {field}: {type} ({constraints})

## Environment Variables
{VAR_NAME}: {description}
```

---

### Feature File (`Features/FEATURE-{name}.md`)

```markdown
# FEATURE: {Name}

## Summary
{One paragraph description}

## Affected Services
- {service-a}: {what changes}
- {service-b}: {what changes}

## Data Model
{fields, types, constraints}

## API Shape
{endpoints, methods, auth}

## Cross-Service Contracts
{what service A delivers, what service B expects}

## Open Design Decisions
- [ ] {Decision 1}
- [ ] {Decision 2}
```

---

### Prompt File (`Prompts/PROMPT-{service}-{feature}.md`)

```markdown
# PROMPT: {Service} — {Feature}

## Dependencies
{List any prompts that must be applied first, and what contracts they establish}

## Files to Study (read these first)
- src/{pattern-reference}/
- src/{pattern-reference}/

## Files to Create
- src/{feature}/{feature}.module.ts
- src/{feature}/{feature}.service.ts
- src/{feature}/{feature}.controller.ts
- src/{feature}/schemas/{feature}.schema.ts
- src/{feature}/dto/create-{feature}.dto.ts
- src/{feature}/dto/update-{feature}.dto.ts

## Schema Contract
{Resource}:
  _id: String (UUID v4, server-generated)
  tenant: String (index, required)
  {field}: {type} ({constraints})

## Endpoint Definitions
| Method | Path | Guard | Notes |
|---|---|---|---|
| GET | /{resource} | JWT | Paginated |
| GET | /{resource}/:id | JWT | Single item |
| POST | /{resource} | JWT | Create |
| PATCH | /{resource}/:id | JWT | Update |
| DELETE | /{resource}/:id | JWT | Delete |

## Naming Rules
- Service class: {Resource}Service
- Controller class: {Resource}Controller
- Module class: {Resource}Module
- Create DTO: Create{Resource}Dto
- Update DTO: Update{Resource}Dto
```

---

### Repo-Level Instruction File (`.github/copilot-instructions.md`)

```markdown
# {service-name} — Copilot Instructions

## Stack
{Runtime} / {Framework} / {Database} / {Auth}

## Conventions
- {Naming rule}
- {ID generation rule}
- {Schema defaults}
- {Auth guard placement}
- {Error handling pattern}

## Module Structure
src/{feature}/{feature}.module.ts
src/{feature}/{feature}.service.ts
src/{feature}/{feature}.controller.ts
src/{feature}/schemas/{feature}.schema.ts
src/{feature}/dto/create-{feature}.dto.ts
src/{feature}/dto/update-{feature}.dto.ts

## Environment Variables
PORT={value}
{VAR}=from env
```

> For Claude Code users, name this file `CLAUDE.md`. For OpenAI Codex, use `AGENTS.md`. The prompt files are plain Markdown and are portable across agents.

---

## The Five Principles

| # | Principle |
|---|---|
| 1 | **Prompts are self-contained.** The agent should implement without asking questions or reading files beyond those listed. |
| 2 | **One prompt per service, one session per prompt.** Never span multiple repos in one session. |
| 3 | **Session boundaries align with context boundaries.** Start fresh when switching repos or returning to the Spec Hub. |
| 4 | **Reconcile after every implementation.** Do not defer spec updates. |
| 5 | **Specs describe what is implemented; feature files describe what is not.** |

---

## Minimal Setup Checklist

- [ ] Create `{platform}-specs` repository
- [ ] Add `00-architecture-overview.md` with a status table
- [ ] Add one service spec per service (`{N}-{service-name}.md`)
- [ ] Add `.github/copilot-instructions.md` to each service repo
- [ ] Run one feature end-to-end through all five steps
- [ ] After close: confirm specs match implementation and feature file is deleted
