# SpecHub: How I Ship Features Across Three Repos With an AI Agent (and Stay Sane)

*A spec-first workflow that keeps an AI agent precise even when one feature touches a frontend, a backend, and a CDN service — built on the discipline of context engineering.*

---

## The problem nobody warns you about

AI-assisted coding looks magical in a demo: one repo, one prompt, working feature. Then you try it on a real platform — the kind where a single feature spans a public frontend, an admin web app, a backend API, and a static-file CDN service — and the magic quietly leaks away.

The agent still writes code that compiles. But backend assumptions bleed into frontend work. Naming conventions drift between services. Every new chat session starts cold, so you re-explain the same design decisions for the third time. And after a week of "successful" implementations, no single document tells you what the system actually does anymore.

The instinct is to fix this with *better prompts*. That's the wrong layer. The real problem is **context**: what the model sees, how explicit it is, and what you deliberately keep out. Too much source and it wanders. Too little structure and it invents. Mixed signals from three repos and it blends conventions that were never meant to touch.

SpecHub is the workflow I landed on after running into every one of these failures on a real multi-service content platform. It's not a tool you install — it's a way of organizing specs, prompts, and sessions so the agent is always working from clean, deliberate inputs.

This article walks through how it works, shows real prompts and feature files, and explains why each rule exists.

*If you'd rather read files than prose, there's a complete example Spec Hub on GitHub ([GenerosoCantu/SpecHub-Example](https://github.com/GenerosoCantu/SpecHub-Example)) — I walk through it at the end.*

---

## First, the distinction that makes the rest make sense

Andrej Karpathy put it precisely:

> *"Context engineering is the delicate art and science of filling the context window with just the right information for the next step."*

That distinction from *prompt* engineering is the whole game. Prompt engineering is about crafting better sentences. Context engineering is about **what information is present in the model's window** — its shape, its relevance, and crucially, what is *not* there.

Every rule in SpecHub exists to control that window. The centralized spec layer, the stateless prompt format, the session boundaries — none of them are about writing cleverer instructions. They're about making sure that when the agent starts a task, it sees exactly the right slice of the system and nothing else. Keep that lens in mind and every rule below stops looking like bureaucracy and starts looking like the same idea applied at different layers.

---

## The core idea: a Spec Hub that sits *above* the repos

The first decision is structural and almost boringly simple: **the specifications live in their own workspace, separate from every service repo.**

That sounds obvious until a feature touches three services. Without a shared spec layer, the same design decision gets copied into three codebases and immediately starts drifting. With one, there's exactly one source of truth — and the service repos are never allowed to become alternates.

A SpecHub workspace contains specs, an architecture overview, feature design files, and generated prompts. It contains **no application source code**. Here's the actual layout from my platform:

```text
platform-specs/
│
├── 00-architecture-overview.md     ← system-wide truth + feature status
├── WORKFLOW.md                     ← the process itself, authoritative
├── CHANGELOG.md                    ← implementation history
│
├── 01-public-frontend.md           ← one spec per service
├── 02-webapp-frontend.md
├── 03-main-api.md
├── 04-tenant-service.md
├── 05-cdn-service.md
├── 06-banner-service.md
├── 07-control-center-frontend.md
│
├── Features/                       ← pending design docs (scratchpad)
│   └── Implemented/                ← archived design docs for shipped features
├── Prompts/                        ← active implementation prompts
│   └── Implemented/                ← prompts for completed work
```

The service repositories, by contrast, contain **only** source code and a lean `.github/copilot-instructions.md` file (or `CLAUDE.md` for Claude Code, `AGENTS.md` for Codex). That file is small but crucial: it tells the agent *which spec to read before doing anything*, so even a fresh session routes itself to the right context. The relationship is inverted from what most teams expect — the Spec Hub is the authority; the service repos are the implementation targets. And because that file holds only stable, repo-wide conventions — stack, naming rules, module layout — and never feature-level detail, there is very little in it that *can* drift from the hub. Everything that changes from one feature to the next lives in the hub and reaches the repo through a prompt, not through the instructions file.

It's worth being explicit about each kind of file — not by *who types it* (in practice an agent drafts all three) but by *who owns its correctness* and *how durable it is*, because that's what actually separates them:

- **Service specs (`##-service.md`)** — *AI-drafted, human-curated source of truth.* You guide the agent to write and revise them in the hub, but you own them: they are the durable contract every prompt is generated from. The agent never edits them autonomously from inside a service repo.
- **Feature files (`Features/FEATURE-*.md`)** — *AI-drafted from your intent, human-refined scratchpad.* You describe what you want; the agent reads the relevant service specs and drafts the feature file against them; you correct it. Disposable — archived once the feature ships.
- **Prompts (`Prompts/PROMPT-*.md`)** — *AI-generated, human-reviewed scratchpad.* The agent drafts them from the spec, a person checks them once. Disposable — archived once the work ships.

The pattern is a gradient of *ownership*, not authorship: heaviest for specs (the source of truth), lighter for feature files (your intent plus a review pass), lightest for prompts (a glance before applying). What stays invariant is that the agent never edits a spec *as part of building a feature* — every spec change is a deliberate, reviewed step, Steps 2 and 5 below.

---

## Why spec files are context compression, not just documentation

Here is the insight that most spec-driven-development writing misses, and it's worth pausing on before walking through the loop.

A properly written service spec is not primarily a planning document. It is a **compressed representation of a codebase**, designed to fit inside an AI's context window without requiring the model to read the actual source.

Consider what happens when you open a project repo and ask an agent directly: *"Add a photo galleries module."* The agent must first understand the existing codebase. It reads services, schemas, route handlers, DTOs, and test files to learn naming patterns, error-handling conventions, how the auth guard is applied, what the schema conventions look like. For a backend of any real size, that's thousands of lines of source pulled into context *before any useful work begins*.

Now consider asking the agent to implement from a well-written spec prompt:

> *"Study `galleries.service.ts` (pattern reference). Create `print-editions.module.ts`, `print-editions.service.ts`, `print-editions.controller.ts`, and `print-editions.schema.ts`. The schema has these exact fields with these types. The API has these endpoints with these guards. The DTO is named `CreatePrintEditionDto`."*

The agent reads three files. It implements eight. It never needs to scan the other 200 files in the project.

The spec compresses what would take thousands of lines of source to communicate into a 150–250 line document. It works because a spec encodes *conventions and contracts* rather than *implementations* — and conventions are what the agent actually needs. This is context engineering applied deliberately: curate what goes in the window, and keep everything else out.

So what does the artifact the prompt is generated *from* actually look like? Here's the `Authors` section of `03-main-api.md` — the same feature whose prompt appears later in this article. Notice it describes *contracts*, not code:

```markdown
### Authors

Manages author profiles. Authors can be linked to stories for byline
attribution. Integrates with the CDN for photo storage.

#### Schema
| Field  | Type   | Default                | Description               |
|--------|--------|------------------------|---------------------------|
| _id    | String | UUID v4 (randomUUID()) | Unique identifier         |
| name   | String | —                      | Author display name (req) |
| photo  | String | —                      | Photo filename on the CDN |
| status | String | "Active"               | "Active" or "Inactive"    |

#### Endpoints
| Method | Path         | Guard | Description                      |
|--------|--------------|-------|----------------------------------|
| GET    | /authors     | JWT   | List authors (paginated)         |
| POST   | /authors     | JWT   | Create; moves photo from tmp     |
| PATCH  | /authors/:id | JWT   | Update; manages CDN JSON + photo |
| DELETE | /authors/:id | JWT   | Delete; removes CDN JSON + photo |
```

That's the entire surface the agent needs in order to work on authors: the fields with their types and defaults, and the endpoints with their guards. The prompt in Step 3 is generated straight from a section like this — which is why it can be precise without the agent ever opening the existing source. The spec is human-authored and updated by people; the agent only ever *reads* it.

The cost difference is stark and roughly ordered like this. An ad-hoc request with no spec drags 2,000–5,000 lines into context per task and carries a high risk of the agent inventing conventions. A per-repo SDD tool cuts that to a few hundred lines and works well *inside one repo*. A centralized hub with stateless prompts holds it at 200–400 lines per service. And the worst case isn't the single-repo ad-hoc request — it's a cross-service feature handled in one session, where two or three codebases compete in the same window, attention dilutes, and the agent produces code that compiles but quietly violates one repo's conventions. That last failure mode is exactly what the rest of this workflow is built to prevent.

---

## The five-step loop

Every feature — large or small — moves through the same pipeline:

```text
1. DESIGN    →  Create Features/FEATURE-{name}.md
2. CASCADE   →  Write the decisions into the affected service spec(s)
3. PROMPT    →  Generate Prompts/PROMPT-{service}-{feature}.md (one per service)
4. IMPLEMENT →  Apply each prompt inside its service repo (new session per service)
5. CLOSE     →  Reconcile specs + architecture + changelog, archive the feature file
```

Each step is a prerequisite for the next. The discipline isn't bureaucracy — it's what keeps the agent's context clean at every handoff. Let's walk through each one with real artifacts.

---

### Step 1 — Design: the feature file is a thinking space

Before touching a spec or writing a line of code, the feature gets documented in `Features/FEATURE-{name}.md`. This is the scratchpad where every design decision is made *before* it becomes a commitment.

Why a separate file? Because the service specs describe the system as it *is*. If you mix half-formed design ideas into them, you pollute the source of truth and the prompts you later generate from it become unreliable. The feature file is the safe place to think; once the decisions are settled, they graduate into the spec, and the feature file is deleted.

Here's a real one — a feature to import editorial stories from a legacy site that publishes each article as an [NINJS](https://www.iptc.org/standards/ninjs/) (IPTC News-in-JSON) document. Note how much of it is *decisions*, not code:

```markdown
# FEATURE — Story Import (NINJS)

**Status:** Pending (design)
**Scope:** Main API only (new ImportModule). Driven by a thin external
script/cron — no Web App work in v1.

## Description
Import editorial stories from a legacy site that exposes each story as an
NINJS document. The Main API gains an ImportModule that fetches an NINJS
payload, maps it onto the platform's Story model, resolving and (where needed)
creating the related section, subsection, topics, and authors, optionally
downloading the feature image, then publishes through the existing pipeline.

## Why an API module, not a standalone script
A script would re-implement auth, CDN orchestration, slug generation, and
author/topic dedupe — all of which already live in the Main API. Workspace
rules forbid service logic becoming an alternate source of truth, so the heavy
lifting belongs in an endpoint.

## Confirmed design decisions
1. Delivery surface — Main API endpoint only, driven by a thin cron.
2. Unknown section/subsection — configurable mapping table + default fallback.
   Sections are never auto-created (they carry config/covers).
3. Import status — imported stories are created as Active (publish immediately).

## Open design decisions
- Author identity: match by case-insensitive name only, or add a source/alias
  field to avoid collisions between distinct people with the same name?
- Re-import policy: overwrite editor changes, or skip if a human has edited?

## Outstanding work (checklist)
- [ ] Add `source` object + partial-unique index to Story schema.
- [ ] Cascade ImportModule into 03-main-api.md as Pending.
- [ ] Add status-table row to 00-architecture-overview.md §12.
- [ ] Generate Prompts/PROMPT-main-api-story-import.md.
```

The most valuable sections here are **Confirmed design decisions** and **Open design decisions**. They force every ambiguity to the surface *before* the agent ever sees the task. The agent never has to guess "should re-imports overwrite human edits?" — by the time it gets a prompt, that question has a documented answer.

---

### Step 2 — Cascade: write the spec as if it already shipped

Once the design is confirmed, you write it into the relevant service spec(s) **in the exact format the rest of that spec uses** — module, endpoints, schema, data model, the works. You write it as though the feature is already built.

This feels backwards the first time. *Why document something that doesn't exist yet?* Because **the implementation prompt is generated from the spec.** If the spec is vague, the prompt is vague, and the agent fills the gaps by inventing conventions. Writing the spec first forces every field name, validation rule, and access guard to be explicit — which is precisely what the agent needs to get it right on the first pass.

For the Story Import feature, this meant adding a `source` provenance object to the documented Story schema, the two new endpoints (`POST /import/stories` and `POST /import/stories/batch`), the NINJS→Story field mapping, and the env config — all in `03-main-api.md`. You also add a row to the architecture overview's feature status table, marked **Pending**, with a simple HTML comment like `<!-- PENDING: FEATURE-story-import -->` against the new sections so any reader — human or agent — knows they're designed but not yet built. Step 5 removes those markers once the work is verified.

---

### Step 3 — Prompt: one self-contained artifact per service

Now you open a **fresh session in the SpecHub workspace** and ask the agent to generate an implementation prompt. The prompt is the heart of the whole system, so it has a strict shape. A good prompt tells the agent:

- **Files to study** — existing files in the service repo to read for patterns
- **Files to create** — exact paths, no ambiguity
- **Schema contract** — field names, types, defaults, indexes
- **Endpoint definitions** — method, path, guard, request/response shape
- **Pattern references** — "follow the same structure as `galleries.service.ts`"
- **Naming rules** — DTO names, Redux slice, route strings, verbatim from the spec

Here's the opening of a real prompt — the one that added author profiles and bylines to the Main API:

```markdown
# Task: Implement the AuthorsModule and Add Author Attribution to Stories

## Context
This adds author profile management to the Main API and links authors to
stories. Two parts:
1. New AuthorsModule — Full CRUD for author profiles in MongoDB. When an
   author is "Active", the service writes a static JSON file to the CDN so the
   Public Frontend can render profile pages without querying the database.
2. `authors` field on the Story entity — the CDN JSON for a published story
   embeds a denormalised snapshot of each author's _id, name, and photo.

## How the CDN integration works (background)
The CDN Service exposes three endpoints used here:
  PATCH /files/:tenant/json        → writes the static JSON
  PATCH /files/:tenant/moveimages  → moves photos from tmp to permanent path
  DELETE /files/:tenant            → deletes a single file by path
The prefix for authors is "author". All CDN calls forward the original Bearer
token via getToken(req.headers) from src/utils/file-upload.utils.ts.

## Relevant Existing Files to Study Before Implementing
src/
├── menus/menus.service.ts     ← Reference: Mongoose CRUD pattern
├── menus/menus.controller.ts  ← Reference: controller + JwtAuthGuard
├── stories/stories.schema.ts  ← UPDATE: add authors field
├── fronts/fronts.service.ts   ← Reference: CDN JSON write + moveimages
└── utils/file-upload.utils.ts ← getToken() — already used by fronts/stories

## New Files to Create
src/authors/
├── authors.module.ts
├── authors.controller.ts
├── authors.service.ts
└── authors.schema.ts
```

Notice what this does to the agent's context. When this prompt lands in a 200-file repo, the agent reads **three** pattern-reference files, then creates and edits exactly the ones named. It never scans the other 197. That precision — the agent working from a tight, deliberate slice of the codebase — is the entire payoff of the workflow.

Two rules make prompts reliable:

- **Stateless and self-contained.** A prompt assumes no prior conversation. You should be able to paste it into a brand-new session and get correct output. If the agent would need to ask a clarifying question or read a file you didn't name, the prompt is incomplete — fix it before implementing.
- **One prompt per service.** A cross-service feature gets a separate prompt for each repo. Frontend prompts explicitly declare the backend prompt as a prerequisite. You never ask one session to context-switch between two entirely different codebases.

---

### Step 4 — Implement: a fresh session per service repo

You open a **new session inside the target service repo** and apply the prompt. Backend first; the frontend prompt only after the backend is built and tested.

The session boundaries are deliberate and align with service boundaries:

- A fresh session per service — never carry one across repos.
- If the agent produces a bug, fix it *in the same session* — it has the full implementation in context and can correct precisely.
- A different feature gets a different session.

Verification is a human gate, not an automated one. A person runs the build, exercises the new endpoints or UI, and reads the diff before the work counts as done. The workflow makes the agent's output predictable and convention-compliant; it does not make it self-certifying. Nothing graduates to Step 5 until a human has confirmed it actually works.

Why so strict about fresh sessions? Because context from a previous conversation — even a *successful* one — is noise. Every ongoing session accumulates invisible state: prior responses, intermediate reasoning, assumptions about file contents. The model may pattern-match on the wrong prior example or assume a file is in a state it no longer is. A clean session with a self-contained prompt produces cleaner, more predictable output. The prompt carries the context; the session doesn't have to.

---

### Step 5 — Close the loop: reconcile, then archive

After implementation is verified, you do four things **in a single pass**, back in the SpecHub workspace:

1. **Update the service spec** to match what was actually built — correct any drift between plan and reality, remove the `PENDING` markers, bump the "Last updated" date.
2. **Update the architecture overview** — flip the feature's status row from Pending to complete.
3. **Update the changelog** — one line: `| date | feature | description |`.
4. **Archive the feature design file** — move it to `Features/Implemented/` and add a banner marking it as a frozen historical record. The spec now holds the *contracts*, but the feature file's *design rationale* — the "why", the confirmed and open decisions, the alternatives weighed — lives nowhere else, so it's worth keeping. The banner (`ARCHIVED — NOT a source of truth`) is what keeps it from ever being mistaken for a live document, which is what neutralizes the old risk of two diverging copies. (The applied prompts move to `Prompts/Implemented/` in the same pass.)

These four are all consequences of the same event, so doing them together guarantees they stay consistent. If you update the spec but forget to archive the feature file, the next person opening `Features/` can't tell whether that work is pending or done. The loop is only closed when the SpecHub once again reflects reality exactly.

---

## A real cross-service feature, end to end

The loop above is abstract until you watch one feature move through it, so here is a real one from the platform: **author profiles and bylines.** It touched three repos — the Main API, the Web App Frontend, and the Public Frontend — with the CDN Service acting as the static delivery layer in between. This is not a hypothetical; the prompts, schema, and reconciled deviation below are taken from the workspace.

**Design.** The feature file settled the shape before any code: an author is a profile (`name`, `bio`, `photo`, `email`, `link`, `status`), and a story gains an `authors` field. One decision mattered more than the rest — when an author is `Active`, the Main API writes a static `author.json` to the CDN so the Public Frontend can render profile data without ever querying the database, and a published story's `story.json` embeds a denormalised snapshot of each author (`_id`, `name`, `photo`) so bylines render with no extra lookup.

**Cascade.** Those contracts were written into `03-main-api.md` (the `AuthorsModule`, the five CRUD endpoints, the CDN JSON-write rules) and into the two frontend specs, each new section marked pending.

**Prompt — one per service.** Three separate prompt files, generated from the specs. `PROMPT-main-api-authors.md` opened like this:

```markdown
# Task: Implement the AuthorsModule and Add Author Attribution to Stories

## Context
1. New AuthorsModule — Full CRUD for author profiles in MongoDB. When an
   author is "Active", the service writes a static JSON file to the CDN so
   the Public Frontend can render profile pages without querying the database.
2. `authors` field on the Story entity — the CDN JSON for a published story
   embeds a denormalised snapshot of each author's _id, name, and photo.

## Relevant Existing Files to Study Before Implementing
src/menus/menus.service.ts     ← Reference: Mongoose CRUD pattern
src/menus/menus.controller.ts  ← Reference: controller + JwtAuthGuard
src/stories/stories.schema.ts  ← UPDATE: add authors field
src/fronts/fronts.service.ts   ← Reference: CDN JSON write + moveimages
```

The Web App prompt (`PROMPT-webapp-frontend-authors.md`) declared the backend prompt as its prerequisite; the Public Frontend byline prompt came later as its own follow-on.

**Implement — a session per repo.** The Main API prompt was applied first, in its own session, and the endpoints were verified by hand. Only then did the Web App prompt get applied, in a separate session in a different repo. The Public Frontend byline work followed the same way. No session ever held two codebases at once.

**Close.** The specs were reconciled — including one real deviation worth noting: the byline's original "render the date as provided" decision was *superseded* during implementation by reusing the platform's date-format configuration (UTC converted to the tenant timezone and formatted per `websiteConfig.dateFormats`). That drift between plan and reality is exactly what Step 5 exists to absorb: the spec was corrected to describe what shipped, the pending markers came out, the architecture status row flipped to complete, one line went into the changelog, and the feature file was archived.

Three repos, three prompts, three isolated sessions, one reconciled deviation. That's the whole method running on a real feature.

---

## What's inside the architecture overview

A few people asked me what the top-level `00-architecture-overview.md` actually contains, since it's the document the agent reads first for any system-wide question. It's the map of the whole platform. The section structure looks like this:

```text
1.  Product Summary
2.  System Components       (client apps, backend services, data layer)
3.  High-Level Architecture Diagram
4.  Multi-Tenancy Strategy  (incl. static JSON content delivery)
5.  Authentication and Authorization
6.  Service Communication
7.  Hosting and Deployment  (which service runs where)
8.  Shared Conventions      (API naming, domain entities, CDN storage,
                             env vars, logging, code organization)
9.  Spec Document Index
10. Known Gaps and Technical Debt
11. Open Questions Log
12. Pending Features         ← status table, one row per in-flight feature
```

Sections 8 and 12 do the heavy lifting day to day. **Section 8 (Shared Conventions)** is what stops convention drift — it's where "this is how we name APIs / store CDN files / structure env vars" lives, so every prompt inherits the same rules. **Section 12 (Pending Features)** is the live status board: a table where each feature has a row showing which services are done and which are still pending. That table is what gets flipped to "complete" in Step 5.

The overview answers *system-wide* questions; the per-service specs answer *service-specific* ones. The agent's routing file tells it which to open for a given task, so it never reads all of them at once.

---

## "Isn't this just Spec Kit?"

It's the natural objection, and the answer sharpens what SpecHub is for. Tools like GitHub Spec Kit and AWS Kiro popularized spec-driven development, and they're good at what they do — but they're **single-repo by design.** They answer *"how do I give my agent enough context to build a feature in this project?"* They don't answer *"how do I coordinate one feature across five services and three repos without duplicating context or tangling sessions?"*

That's the layer SpecHub adds, and the two coexist naturally: Spec Kit can handle per-feature implementation phases *inside* a repo, while the Spec Hub handles the cross-repo coordination above it. On a single-service task they perform about the same. The difference only shows up on cross-service features — and that's precisely where unaided AI-assisted development tends to fail silently.

---

## A note on skills

Another fair question is whether agent *skills* — reusable, named procedures like Claude Code Skills — overlap with any of this. They're complementary, not competing, because they operate on different things. A spec encodes a *contract*: what the system is. A skill encodes a *repeatable procedure*: how to perform an activity. The recurring routines in this workflow — generating a prompt from a spec, running the close-the-loop reconciliation, archiving a feature file — are exactly the kind of multi-step motions worth packaging as a skill. Doing so keeps the procedure out of the context window until it's actually needed, and makes every run identical instead of re-improvised. Put plainly: specs say *what*, skills say *how to operate on it*. A mature setup uses both — specs as the source of truth, skills as the repeatable motions over that truth.

---

## Why it works: it's context engineering, not prompt engineering

Step back and the pattern is clear. Every rule in this workflow exists to control what enters the model's working set:

- **Specs outside the repos** → one source of truth, no cross-repo drift.
- **Specs as compression** → the agent reads a 200-line contract instead of 2,000 lines of source.
- **Feature file as scratchpad** → design noise never pollutes the specs.
- **Spec-before-code** → prompts are generated from explicit contracts, not guesses.
- **One self-contained prompt per service** → the agent reads a tight slice, not the whole repo.
- **Fresh session per service** → no stale context bleeding across boundaries.
- **Close-the-loop reconciliation** → the hub always matches reality, so the *next* feature starts from truth.

None of these ingredients is novel on its own. The value is in the combination, and in the discipline of always keeping the spec layer authoritative. The goal was never to give the agent *more* context. It was to give it exactly the right context, and nothing else.

If you're doing AI-assisted work across more than one repo and it feels like it's fraying, the fix probably isn't a cleverer prompt. It's a place for your specs to live, a loop that keeps them honest, and session boundaries that match your service boundaries.

---

## See the full example

Reading about a spec layer only gets you so far; it's easier to judge once you can open the files. I put a complete, runnable-on-paper example Spec Hub on GitHub:

**[github.com/GenerosoCantu/SpecHub-Example](https://github.com/GenerosoCantu/SpecHub-Example)**

It's a small fictional multi-tenant storefront platform ("Plaza") — five services, a shared database — modeling the exact structure described here. Inside you'll find the architecture overview, one full service spec and four lean ones, the spec-writing guidelines, the workflow, the fill-in templates, and one cross-service feature ("gift cards") traced end to end: the feature file, the spec changes it cascaded into, and the prompt generated from those specs. It's the same skeleton I use, with all real product details stripped out — fork it and replace the contents with your own.

---

*Written from a real multi-service content platform with six services, two frontend applications, and a shared MongoDB Atlas cluster. The feature file, prompt, and architecture structure shown above are lightly trimmed but otherwise verbatim from the workspace they describe.*

---

### Publishing notes (delete this section before importing to Medium)

- Author here in Markdown, push to GitHub, then use Medium's **Import a story** with the file URL — it pulls headings, lists, links, and inline code cleanly.
- This article deliberately uses **no tables** (Medium can't render them); everything is prose, lists, or code blocks, so the import needs no manual conversion.
- After import, verify the fenced ```text``` and ```markdown``` blocks rendered as Medium code blocks. If any flattened to plain text, select them and apply Medium's Code Block formatting.
- Replace the title/subtitle with Medium's native title + subtitle fields rather than leaving them as body headings.
