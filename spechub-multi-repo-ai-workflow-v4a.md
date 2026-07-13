# SpecHub, Part 1: How I Ship Features Across Three Repos With an AI Agent (and Stay Sane)

*A spec-first workflow that keeps an AI agent precise even when one feature touches a frontend, a backend, and a CDN service, built on the discipline of context engineering. Part 1 covers the core workflow; Part 2 covers what fifty features of real use taught me about scaling it.*

---

## The problem nobody warns you about

AI-assisted coding looks magical in a demo: one repo, one prompt, working feature. Then you try it on a real platform, the kind where a single feature spans a public frontend, an admin web app, a backend API, and a static-file CDN service, and the magic leaks away.

The agent still writes code that compiles. But backend assumptions bleed into frontend work. Naming conventions drift between services. Every new chat session starts cold, so you re-explain the same design decisions for the third time. And after a week of "successful" implementations, no single document tells you what the system actually does anymore.

The instinct is to fix this with better prompts. That's the wrong layer. The real problem is context: what the model sees, how explicit it is, and what you deliberately keep out. Give the model too much source and it wanders; give it too little structure and it invents. Feed it mixed signals from three repos and it blends conventions that were never meant to touch.

SpecHub is the workflow I landed on after hitting every one of these failures on a real multi-service content platform. There's nothing to install: it's a way of organizing specs, prompts, and sessions so the agent always works from clean, deliberate inputs. This first part covers the core: the hub, the reason specs work at all, and the five-step loop every feature moves through.

*If you'd rather read files than prose, there's a complete example Spec Hub on GitHub ([GenerosoCantu/SpecHub-Example](https://github.com/GenerosoCantu/SpecHub-Example)); I come back to it at the end.*

---

## The distinction that makes the rest make sense

Prompt engineering is about crafting better sentences. Context engineering is about what information is present in the model's window: its shape, its relevance, and what is left out of it. Every rule in SpecHub exists to control that window. Keep that lens and the rules below stop looking like bureaucracy and start looking like the same idea applied at different layers.

---

## The core idea: a Spec Hub that sits *above* the repos

The first decision is structural and almost boringly simple: **the specifications live in their own workspace, separate from every service repo.**

That sounds obvious until a feature touches three services. Without a shared spec layer, the same design decision gets copied into three codebases and immediately starts drifting. With one, there's exactly one source of truth, and the service repos are never allowed to become alternates.

A SpecHub workspace contains specs, conventions, a status board, feature design files, generated prompts — and no application source code. The actual layout from my platform:

```text
platform-specs/
├── 00-architecture-overview.md   ← system map + shared decisions
├── CONVENTIONS.md                ← naming, entities, env vars, logging, code org
├── STATUS.md                     ← live feature status board
├── WORKFLOW.md                   ← the process itself, authoritative
├── CHANGELOG.md                  ← implementation history (the ONLY dated log)
├── 01-public-frontend.md         ← one spec per service — the two largest are
├── 02-webapp-frontend.md + /     ←   INDEX files over per-module directories
├── 03-main-api.md + /            ←   (a Part 2 story)
├── 04…07-*.md                    ← tenant, CDN, banner, control-center specs
├── Features/  (+ Implemented/)   ← pending design docs; archived once shipped
├── Prompts/   (+ Implemented/)   ← implementation prompts (a dispatch board)
├── repo-instructions/            ← canonical copies of each repo's instruction file
├── templates/                    ← fill-in templates (spec, feature, prompt)
└── .claude/skills/               ← the workflow's recurring motions, as skills
```

The service repositories, by contrast, contain only source code and a lean instruction file (`CLAUDE.md`, `.github/copilot-instructions.md`, or `AGENTS.md`, depending on your agent). It holds nothing but stable, repo-wide conventions (stack, naming rules, module layout) and tells the agent which spec to read before doing anything, so even a fresh session routes itself to the right context. Everything feature-specific reaches the repo through a prompt, never through the instruction file. The relationship is inverted from what most teams expect: the Spec Hub is the authority; the service repos are the implementation targets.

The hub's three artifact kinds differ by who owns their correctness, not who types them; in practice an agent drafts all three:

- **Service specs (`##-service.md`)** — *AI-drafted, human-curated source of truth.* The durable contract every prompt is generated from. The agent never edits them autonomously from inside a service repo.
- **Feature files (`Features/FEATURE-*.md`)** — *AI-drafted from your intent, human-refined scratchpad.* You describe what you want; the agent drafts it against the specs; you correct it.
- **Prompts (`Prompts/PROMPT-*.md`)** — *AI-generated, human-reviewed.* A person checks them once before applying.

The pattern is a gradient of ownership: heaviest for specs, lightest for prompts. What stays invariant is that the agent never edits a spec as part of building a feature; every spec change is a deliberate, reviewed step (Steps 2 and 5 below).

---

## Spec files are context compression, not documentation

This is the part most writing about spec-driven development misses, and it's worth pausing on before the loop. A properly written service spec is more than a planning document: it's a compressed representation of a codebase, designed to fit inside an AI's context window without the model reading the actual source.

Ask an agent directly, inside a repo, to "add a photo galleries module," and it must first understand the codebase: services, schemas, DTOs, guards, test files. That means thousands of lines pulled into context before any useful work begins. Now ask it to implement from a well-written spec prompt:

> *"Study `galleries.service.ts` (pattern reference). Create `print-editions.module.ts`, `print-editions.service.ts`, `print-editions.controller.ts`, and `print-editions.schema.ts`. The schema has these exact fields with these types. The API has these endpoints with these guards. The DTO is named `CreatePrintEditionDto`."*

The agent reads three files, implements eight, and never scans the other 200.

The spec compresses thousands of lines of source into a 150–250 line document, because it encodes conventions and contracts rather than implementations, and conventions are what the agent actually needs. Here's the `Authors` section of my Main API spec; notice it describes contracts, not code:

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

That's the entire surface the agent needs to work on authors: fields with types and defaults, endpoints with guards. The prompt in Step 3 is generated straight from a section like this, which is why it can be precise without the agent ever opening the existing source.

---

## The five-step loop

Every feature, large or small, moves through the same pipeline:

```text
1. DESIGN    →  Create Features/FEATURE-{name}.md
2. CASCADE   →  Write the decisions into the affected service spec(s) + STATUS.md
3. PROMPT    →  Generate Prompts/PROMPT-{service}-{feature}.md (one per service)
4. IMPLEMENT →  Apply each prompt inside its service repo (new session per service)
5. CLOSE     →  Verify specs against the code, reconcile, log, archive
```

Each step is a prerequisite for the next; the discipline is what keeps the agent's context clean at every handoff. Let's walk through each one with real artifacts.

### Step 1 — Design: the feature file is a thinking space

Before touching a spec or writing code, the feature gets documented in `Features/FEATURE-{name}.md`. Why a separate file? Because the service specs describe the system as it *is*. Mix half-formed ideas into them and you pollute the source of truth, and the prompts you later generate become unreliable. The feature file is the safe place to think; settled decisions graduate into the spec, and after the feature ships the file is archived as a frozen design record.

Here's a real one: importing editorial stories from a legacy site that publishes each article as an [NINJS](https://www.iptc.org/standards/ninjs/) (IPTC News-in-JSON) document. Note how much of it is decisions, not code:

```markdown
# FEATURE — Story Import (NINJS)

**Status:** Pending (design)
**Scope:** Main API only (new ImportModule). Driven by a thin external
script/cron — no Web App work in v1.

## Description
Import editorial stories from a legacy site that exposes each story as an
NINJS document. The Main API gains an ImportModule that fetches an NINJS
payload, maps it onto the platform's Story model, resolving and (where needed)
creating the related section, subsection, topics, and authors, then publishes
through the existing pipeline.

## Confirmed design decisions
1. Delivery surface — Main API endpoint only, driven by a thin cron.
2. Unknown section/subsection — configurable mapping table + default fallback.
   Sections are never auto-created (they carry config/covers).
3. Import status — imported stories are created as Active (publish immediately).

## Open design decisions
- Author identity: match by case-insensitive name only, or add a source/alias
  field to avoid collisions between distinct people with the same name?
- Re-import policy: overwrite editor changes, or skip if a human has edited?

## Implementation order
main-api only (v1).

## Recommended Claude model (per service)
- main-api: Haiku 4.5 — backend-only mapping + CRUD against existing patterns.
```

The two decision sections are the most valuable part: they force every ambiguity to the surface before the agent sees the task. It never has to guess whether re-imports should overwrite human edits; that question has a documented answer, and prompt generation refuses to run while an open decision remains. That gate has caught more half-baked designs than any review.

The footer lines earn their place too. A cross-service feature declares its implementation order once, here, and every generated prompt inherits it, so sequencing is never implicit. The model recommendation decides at design time which tier each implementation session deserves: the cheapest that fits. (Why that's one of the biggest cost levers in the system is a Part 2 topic.)

This feature is also an honesty check on the workflow itself: design review showed the API already exposed everything the import needed, so the ImportModule was rejected in favor of a standalone script driving existing endpoints. The feature shipped with zero service-code changes.

### Step 2 — Cascade: write the spec as if it already shipped

Once the design is confirmed, you write it into the affected service spec(s) in the exact format the rest of that spec uses (module, endpoints, schema, the works), as though the feature were already built.

This feels backwards the first time. Why document something that doesn't exist yet? Because the implementation prompt is generated from the spec. A vague spec makes a vague prompt, and the agent fills gaps by inventing conventions. Writing the spec first forces every field name, validation rule, and access guard to be explicit, which is precisely what the agent needs to get it right on the first pass.

Two bookkeeping moves complete the cascade: a Pending row in `STATUS.md`, and a `<!-- PENDING: FEATURE-story-import -->` marker on each not-yet-built spec section, so any reader, human or agent, knows it's designed but not yet real. Step 5 removes the markers once the work is verified.

### Step 3 — Prompt: one self-contained artifact per service

In a fresh session in the hub, you generate the implementation prompts. Every prompt opens with a *dispatch header*:

```markdown
> **Target repo:** application-main-api
> **Branch:** feature/authors
> **Prerequisites:** none
> **Status:** Generated   <!-- Generated → Applied → Verified -->
> **Recommended model:** Haiku 4.5 — backend-only, straightforward CRUD
```

The header turns `Prompts/` into a dispatch board: what's waiting to be applied (`Generated`), what's implemented but unchecked (`Applied`), what's cleared for close-out (`Verified`). The `Prerequisites` line is how a frontend prompt refuses to run before its backend prompt is verified; the target-repo line is what lets close-out find the actual code later.

The body names the files to study (three to five existing files, for patterns), the files to create (exact paths), the schema contract, the endpoint definitions, and the naming rules, all verbatim from the spec. Here's the opening of the real prompt that added author profiles to the Main API:

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

When this prompt lands in a 200-file repo, the agent reads three pattern-reference files, then creates and edits exactly the ones named. It never scans the other 197. That precision is the entire payoff of the workflow: the agent works from a tight, deliberate slice of the codebase and nothing more.

Two rules make prompts reliable. **Stateless and self-contained:** a prompt assumes no prior conversation and no access to the hub, because the implementing session runs in a different workspace and can't follow a reference back to a spec file. If the agent would need to ask a question or read an unnamed file, the prompt is incomplete; fix it before implementing. **One prompt per service:** a cross-service feature gets a separate prompt per repo, sequenced by the declared implementation order. You never ask one session to context-switch between codebases.

### Step 4 — Implement: a fresh session per service repo

You open a new session inside the target repo, on the model tier the prompt recommends, and apply the prompt. Backend first; the frontend prompt only after the backend is built and verified.

The session boundary tracks the context boundary: a fresh session per service, never carried across repos; a different feature gets a different session; but a bug fix stays in the same session, which has the full implementation in context and can correct precisely. Why so strict? Because context from a previous conversation, even a successful one, is noise: prior responses, intermediate reasoning, assumptions about file contents the model may wrongly pattern-match against. The prompt carries the context; the session doesn't have to.

Verification is a human gate, and the dispatch header records it: after applying, flip `Status:` to `Applied`; only after a person runs the build, exercises the new endpoints or UI, and reads the diff does it become `Verified`. Nothing reaches Step 5 until every prompt in the set says `Verified`. The workflow makes the agent's output predictable and convention-compliant; it does not make it self-certifying.

### Step 5 — Close the loop: verify, reconcile, log, archive

After verification you return to the hub, in a fresh session again, and close out. This is the step everyone is tempted to skip, and it's where the whole system either stays honest or rots. Five moves, one pass:

1. **Verify the spec against the code, not against memory.** The hub's one real failure mode is drift: the spec claiming a contract the implementation silently changed. Open the actual service repo (the dispatch header says where), read the files the prompt named, and confirm the spec's claimed field names, endpoint shapes, and component names match what was built. Where they don't, reality wins: the spec is corrected to describe what shipped. Capture commit SHAs or PR links while you're there.
2. **Update the service spec(s):** correct the drift, remove the `PENDING` markers.
3. **Flip the status board:** the `STATUS.md` row moves to shipped with a one-line note, never more.
4. **Write one dated changelog entry:** what shipped, what deviated, the commit refs, e.g. `(main-api@a1b2c3d, webapp-frontend#124)`. The changelog is the only hub document that accumulates dated history; every other file describes current state.
5. **Archive the feature file and prompts** into their `Implemented/` folders, the feature file with a banner: `ARCHIVED — historical design record. NOT a source of truth.` The spec holds the contracts, but the design rationale (the why, the alternatives weighed) lives nowhere else, so it's worth keeping.

These are all consequences of the same event, so doing them together keeps them consistent. If you update the spec but forget to archive the feature file, the next person opening `Features/` can't tell whether that work is pending or done. The loop is closed only when the hub reflects verified reality, not remembered reality.

---

## A real cross-service feature, end to end

The loop is abstract until you watch one feature move through it: author profiles and bylines, touching the Main API, the Web App, and the Public Frontend, with the CDN service as the static delivery layer in between.

**Design.** The feature file settled the shape before any code: an author is a profile (`name`, `bio`, `photo`, `email`, `link`, `status`), and a story gains an `authors` field. One decision mattered most: an `Active` author gets a static `author.json` written to the CDN so the Public Frontend never queries the database, and a published story's JSON embeds a denormalised author snapshot so bylines render with no extra lookup.

**Cascade.** Those contracts went into the Main API spec and the two frontend specs, each new section marked pending, plus a Pending status row.

**Prompt.** Three prompt files; the Main API one is excerpted in Step 3 above. The Web App prompt declared the backend prompt as its prerequisite in its dispatch header.

**Implement.** Main API first, in its own session, verified by hand, flipped to `Verified`. Only then the Web App prompt, in a separate session in a different repo. No session ever held two codebases at once.

**Close.** The check against the code surfaced one real deviation: the byline's original "render the date as provided" decision had been superseded during implementation by reusing the platform's date-format configuration. Exactly what Step 5 exists to absorb: the spec was corrected to describe what shipped, the markers came out, the status row flipped with a one-line note, one dated entry went into the changelog, and the feature file was archived.

That was three repos, three prompts, three isolated sessions, and one reconciled deviation: the whole method running on a real feature.

---

## Five principles to carry away

1. **The prompt is self-contained.** If a session would need to ask a question or read an unnamed file, fix the prompt before implementing.
2. **One prompt per service, one session per prompt.** Cross-service features fan out in a declared order, never one tangled session.
3. **Session boundaries align with context boundaries.** New repo or new feature, new session; bug fixes stay put.
4. **Reconcile after every implementation, against the code.** The hub is updated from what was actually built, with the commits to prove it.
5. **Specs describe what is implemented; feature files describe what is not.** And exactly one document accumulates history.

(There's a sixth principle, about model tiers and token budgets. It earned its place after real billing surprises, and it's where Part 2 begins.)

None of these ingredients is novel on its own. The value is in the combination, and in keeping the spec layer authoritative. The goal was never to give the agent more context; it was to give it exactly the right context, and nothing else.

---

## Getting started, and what's in Part 2

Start with a skeleton and let it grow:

1. **Create the hub.** A separate `{platform}-specs` workspace with an architecture overview, a status board, a conventions file, and one service spec for your primary API.
2. **Add repo-level instruction files.** Stack, naming conventions, module layout, with the canonical copy kept in the hub.
3. **Run one feature through the loop.** Feature file, cascade, one prompt per service with a dispatch header, implement in dependency order, reconcile.

A complete example Spec Hub, a fictional five-service storefront platform with one cross-service feature traced end to end, is on GitHub: [github.com/GenerosoCantu/SpecHub-Example](https://github.com/GenerosoCantu/SpecHub-Example). 

**Part 2 — What Fifty Features Taught Me** covers everything that only showed up under sustained use: what to do when a spec outgrows the context window, the repo-level instruction file, packaging procedures as skills, keeping the hub honest (staled features, the single-changelog rule, the status board that almost became a history file), routing sessions to the right model tier and the honest token economics, and how SpecHub compares to Spec Kit and OpenSpec.

---

*Written from a real multi-service content platform (six services, two frontends, a shared MongoDB Atlas cluster), roughly fifty features into this workflow. The artifacts shown above are lightly trimmed but otherwise verbatim. Drafted with AI assistance, working from my own specs, session notes, and the artifacts above.*

---

### Publishing notes (delete this section before importing to Medium)

- Push to GitHub, then use Medium's **Import a story** with the file URL.
- All tables are inside fenced code blocks and import cleanly; verify fenced blocks rendered as Medium code blocks.
- Replace the title/subtitle with Medium's native title + subtitle fields.
- After publishing Part 2, add its link in the intro and closing section.
