# Multi-Repo AI Development Needs More Than Better Prompts

*A workflow for shipping features across multiple repos with AI, without the context mess.*

---

I'm a senior manager with a long background in coding and architecture who doesn't get to code that often anymore. There is a growing sentiment that this is the worst time in history to be a manager; you are going to miss the AI revolution from the sidelines. I'm not sure that's entirely true, but it pushed me to get involved more directly. Ironically, the manager skill set turns out to be oddly relevant: the work is shifting toward writing specs and coordinating context, which is closer to what managers already do than to writing code line by line. Architectural experience matters even more: knowing how services connect, where boundaries belong, and which decisions need to be explicit is exactly what makes a spec useful to an AI agent.

I tried AI-assisted coding last year. It was promising but the models weren't quite there; you still spent more time fixing the output than you saved generating it. The latest models changed that equation; they are good enough that the work shifted from writing code to writing specs and managing context.

That shift made me want to get my hands dirty again. Over the past few weeks I picked up a stale personal project, a multi-service content platform, and started building features end to end with AI agents. What started as catching up turned into an obsession with one question: how do you make this work when a single feature spans three repos? This article is what came out of that obsession.

---

## The Multi-Repo Gap

AI-assisted development starts to break down quietly when one feature spans multiple repos.

The agent produces code that compiles, but backend assumptions leak into frontend work, conventions drift across services, and every new session forces you to restate the same design decisions. Writing specs helps. Most public advice still assumes one repo at a time.

Once a feature touches three services, the problem is no longer just “give the agent enough context to implement this feature.” The problem is coordination: where the design lives, how prompts are generated, how context stays clean, and how implementation changes get reconciled back into a shared source of truth.

When I picked up my project again, I ran into all of these problems firsthand and spent a few weeks experimenting until a repeatable workflow emerged.

This article describes that workflow: a centralized Spec Hub above multiple repos, per-service implementation prompts treated as reusable artifacts, and session boundaries that align with service boundaries. None of the ingredients are new. The value is in the combination.

## Context Engineering, in Practical Terms

People often treat AI-assisted development as a prompting problem. In practice, it is usually a context problem.

The issue is not whether you can ask for the right thing. It is whether the model is working from the right inputs in the first place. Too much source, and it wanders. Too little structure, and it starts inventing. Mixed signals across repos, and it quietly blends conventions that were never meant to belong together.

That is what context engineering means here: deciding what enters the model’s working set, how explicit it is, and what stays out.

This workflow is built around that idea. Specs live outside the repos. Prompts are generated as self-contained artifacts. Each service gets its own session. After implementation, the final decisions get reconciled back into the specs.

The point is not to maximize context. It is to control it.

## The Architecture: A Centralized Spec Hub

The first structural decision is simple: keep the specification workspace separate from the service repos.

That sounds obvious until one feature touches three services. Then the same design decisions get copied into three places and start drifting. A shared spec layer avoids that.

A centralized Spec Hub is a standalone workspace that contains specifications, architecture notes, feature design files, and generated prompts. It does not contain application source code.

```text
platform-specs/
│
├── 00-architecture-overview.md
├── CHANGELOG.md
├── WORKFLOW.md
│
├── 02-webapp-frontend.md
├── 03-main-api.md
├── 04-tenant-service.md
├── 05-cdn-service.md
├── 06-banner-service.md
│
├── Features/
│   └── FEATURE-{name}.md
│
└── Prompts/
    ├── PROMPT-main-api-{feature}.md
    ├── PROMPT-webapp-frontend-{feature}.md
    └── PROMPT-{service}-{feature}.md

service-repos/
│
├── my-api/
│   ├── .github/
│   │   └── copilot-instructions.md
│   └── src/
│
├── my-webapp/
│   ├── .github/
│   │   └── copilot-instructions.md
│   └── src/
│
└── my-cdn-service/
    ├── .github/
    │   └── copilot-instructions.md
    └── src/
```

The service repositories contain source code and a lean repo-level instruction file with stack, conventions, and environment variables. Feature specs and design docs live in the Spec Hub.

The relationship is inverted from what most teams expect: the Spec Hub is the authority, and the service repos are implementation targets.

## Why Specs Matter: Context Compression

For AI-assisted work, a good service spec is not just a planning document. It is a compressed representation of a codebase.

Ask an agent to “add a photo galleries module” in a real repo and it first has to reverse-engineer local conventions. It reads services, schemas, controllers, DTOs, tests, and routing patterns before it does anything useful.

A tighter workflow looks different. The prompt tells the agent exactly what to study, what to create, and what contracts to follow.

> Study `src/galleries/galleries.service.ts` as the pattern reference. Create `src/print-editions/print-editions.module.ts`, `print-editions.service.ts`, `print-editions.controller.ts`, and `print-editions.schema.ts`. Use these exact fields, DTO names, route paths, and guards.

The agent reads three files. It implements eight. It never scans the other 200.

That is the point. The spec compresses conventions and contracts into something small enough to fit cleanly into the model’s working set.


## The Stateless Prompt

The next structural choice is the prompt itself. In this workflow, prompts are not chat messages. They are versioned files.

A stateless prompt has to answer five questions without relying on prior session context:

1. **What to study**: the minimum set of existing files that establish the patterns to follow
2. **What to create**: the exact file paths to create or modify
3. **What the data looks like**: field names, types, defaults, indexes, and constraints
4. **What the API looks like**: endpoints, HTTP methods, guards, request shapes, and response shapes
5. **What to call things**: DTO names, slice names, action types, route paths, and class names

```markdown
## Example: Anatomy of a Stateless Prompt

### Prerequisites
- Backend prompt must be applied first: PROMPT-main-api-galleries.md

### Files to study (read these first)
- src/articles/articles.service.ts
- src/articles/articles.controller.ts
- src/articles/schemas/article.schema.ts

### Files to create
- src/galleries/galleries.module.ts
- src/galleries/galleries.service.ts
- src/galleries/galleries.controller.ts
- src/galleries/schemas/gallery.schema.ts
- src/galleries/dto/create-gallery.dto.ts
- src/galleries/dto/update-gallery.dto.ts
- src/app.module.ts

### Schema contract
Gallery:
  _id: String (UUID v4, server-generated via randomUUID())
  tenant: String (index, required)
  title: String (required)
  slug: String (auto-generated from title)
  status: String (enum: ['draft', 'published'], default: 'draft')

### Endpoint definitions
GET    /galleries         JWT guard   paginated list
GET    /galleries/:id     JWT guard   single gallery
POST   /galleries         JWT guard   create
PATCH  /galleries/:id     JWT guard   update
DELETE /galleries/:id     JWT guard   delete

### Naming rules
- Service class: GalleriesService
- Controller class: GalleriesController
- Module class: GalleriesModule
- Redux slice: galleries
- Create DTO: CreateGalleryDto
```

The agent should be able to produce convention-compliant code from this prompt without asking clarifying questions and without reading files beyond the ones listed. If it cannot, the prompt is incomplete. **Principle 1: The prompt must be self-contained.**

That is what makes the prompt portable. Any team member can open the right repo, start a fresh session, apply the prompt, and get a consistent implementation target.

## The Multi-Service Coordination Problem

Cross-service features are where most AI-assisted workflows start to fail. One session spans multiple repos, the context window fills up, and the model loses track of which conventions belong to which codebase.

The rule here is simple: **Principle 2: One prompt per service, one session per prompt.**

For a feature that spans three services, you generate three separate prompt files:

```text
Prompts/
├── PROMPT-main-api-notifications.md
├── PROMPT-webapp-frontend-notifications.md
└── PROMPT-cdn-service-notifications.md
```

Each prompt declares its dependencies explicitly:

```markdown
### Dependencies
This prompt requires PROMPT-main-api-notifications.md to be applied and
verified first. The frontend expects these endpoints to exist:
- GET /notifications (returns { items: Notification[], unread: number })
- PATCH /notifications/:id/read
```

Below is a short summary of the cross-service flow:

1. Design the feature in the Spec Hub
2. Cascade the design into each affected service spec
3. Generate one prompt per service
4. Apply each prompt in its own repo, in its own fresh session
5. Return to the Spec Hub and reconcile

Each repo gets its own session, and each session gets only its own prompt. The only thing that crosses from one service to the next is the prompt file itself.

## Session Boundaries Matter

Long AI sessions accumulate invisible state: prior assumptions, stale interpretations, and attention spread across multiple tasks. That makes later outputs less reliable even when the current prompt looks clear.

**Principle 3: Session boundaries should align with context boundaries.**

| Situation | Session rule |
|---|---|
| Designing and generating prompts | New session in the Spec Hub |
| Implementing in service repo A | New session in repo A |
| Implementing in service repo B | New session in repo B |
| Bug fix or follow-up on current work | Same session |
| Reconciling specs after implementation | New session in the Spec Hub |

## The Full Workflow in Five Steps

1. **Design**  
   Create `Features/FEATURE-{name}.md`. Define the feature, affected services, data model, API shape, and open design decisions.

2. **Cascade**  
   Write the pending design into each affected service spec and update the architecture overview to show the feature as pending.

3. **Prompt**  
   Generate `PROMPT-{service}-{feature}.md` for each affected service. Each prompt should be self-contained and explicit about dependencies.

4. **Implement**  
   Open each service repo in a fresh session, apply the prompt, verify the implementation, and move to the next service only after the current one is working.

5. **Close**  
   Return to the Spec Hub in a fresh session. Reconcile the final implementation into the service specs, remove pending markers, update the architecture overview, prepend the changelog, and delete the feature file. **Principle 4: Reconcile after every implementation, not later.**

### Why design before cascade?

The `Features/` file is disposable. You can still change design decisions there. The service specs should describe implemented reality, not half-settled ideas. **Principle 5: Specs describe what is implemented; feature files describe what is not.**

### Why cascade before prompting?

The prompt is generated from the spec. If the spec is vague, the prompt will be vague, and the agent will start inventing names, patterns, or access rules.

### Why delete the feature file after closing?

Once the design has been absorbed into the service specs, keeping the feature file around creates duplicate truth.

## A Concrete Example: Galleries Across Three Services

One feature on a multi-tenant content platform involved photo galleries with ordered images, publish states, and a public JSON snapshot served through a CDN.

It required changes in three places:

- the Main API: a new module, schema, and CRUD endpoints with a nested images array
- the web app frontend: a new state slice, list view, form view, detail view, and drag-and-drop image reorder
- the CDN service: a new published JSON snapshot written on create, update, and delete

Step 1 produced `Features/FEATURE-galleries.md` with the data model, publish workflow, and one important design decision: images would be uploaded to the CDN first and referenced by URL, not stored as binary in the database.

Step 2 cascaded the design into three service specs, each marked as pending. The Main API spec got the CRUD endpoints, images array schema, and CDN write contract. The frontend spec got the state slice name, view components, and reorder interaction. The CDN spec got the JSON file shape and temporary image cleanup rule.

Step 3 generated three prompts: `PROMPT-main-api-galleries.md`, `PROMPT-webapp-frontend-galleries.md`, and `PROMPT-cdn-service-galleries.md`. The frontend prompt declared its dependency on the backend prompt. The CDN prompt declared its dependency on the API’s publish call.

Step 4 applied the API prompt first in a fresh session, verified the endpoints, then applied the frontend prompt in a separate session in a different repo. The CDN integration was tested last.

Step 5 reconciled one implementation deviation (the image `order` field changed from a manual integer to an array-index-based position), removed the pending markers, updated the architecture overview, prepended the changelog, and deleted the feature file.

Three services. Three prompts. Three isolated sessions.

## The Repo-Level Instruction File as a Safety Net

Even in a Spec Hub workflow, people still ask ad-hoc questions inside repos. The repo-level instruction file is the safety net for that.

It auto-loads into sessions in that repo and supplies the local conventions the agent needs to avoid obvious mistakes:

```markdown
# my-api — Copilot Instructions

## Stack
NestJS v9 / TypeScript / MongoDB (Mongoose) / JWT auth

## Conventions
- UUIDs via Node crypto.randomUUID() — never use nanoid or uuid package
- All schemas: versionKey: false, timestamps: true
- tenant field on every schema, indexed
- Guards: @UseGuards(JwtAuthGuard) on controllers, not individual methods
- Error handling: standard NestJS HttpException with descriptive messages

## Module structure
src/{feature}/{feature}.module.ts
src/{feature}/{feature}.service.ts
src/{feature}/{feature}.controller.ts
src/{feature}/schemas/{feature}.schema.ts
src/{feature}/dto/create-{feature}.dto.ts
src/{feature}/dto/update-{feature}.dto.ts

## Environment
PORT=4000
MONGODB_URI from env
JWT_SECRET from env
CDN_URL from env
```

Its job is not to carry feature design. Its job is to make local conventions unambiguous.

## How This Scales With a Team

This scales because the Spec Hub is shared, reviewable, and version-controlled. The main cost is maintenance discipline: specs have to be updated after implementation, not months later.

There is still a normal coordination problem. If two developers cascade different features into the same service spec at the same time, they can create a merge conflict. That is not a special failure of this workflow. It is the same problem teams already have when two people edit the same source file. The answer is the same: keep changes small and merge often.

A new team member can get the system model from a small set of Markdown files instead of digging through multiple repos.

## Scope and Limits

- **Not a replacement for code review.** Prompts produce a first draft, not a finished system.
- **Not fully automated.** Human judgment is still required for naming, data modeling, API shape, and tradeoffs.
- **Instruction files are agent-specific; prompts are portable.** Repo-level instruction files may need parallel copies for different agents. The prompt files themselves are plain Markdown.
- **This sits above per-repo spec workflows rather than replacing them.** Per-repo workflows can structure implementation well inside a single codebase. The gap addressed here is multi-repo coordination.

## Comparison With Alternative Approaches

The comparison below is qualitative. It reflects workflow tradeoffs, not benchmark measurements.

| Approach | Context load | Convention reliability | Cross-repo coordination | Maintenance burden | Main failure mode |
|---|---|---|---|---|---|
| **Centralized Spec Hub** | Low | High, if specs stay current | Strong | Medium | Specs drift if they are not reconciled after implementation |
| **Per-repo spec workflow** | Low to medium | High within one repo | Weak | Low | Cross-service dependencies stay implicit |
| **Per-service spec copies in each repo** | Medium | Medium | Weak | High | Design drift across duplicated specs |
| **Direct prompting inside a repo** | High | Medium to low | Weak | None | The agent infers conventions from too much source |
| **Cross-service work in one shared session** | Very high | Low | Weak | None | Attention gets split across codebases |

Context load is not just a performance concern. Tools like GitHub Copilot now enforce token-based usage limits. I have seen team members burn through their monthly allocation faster than expected, likely because their sessions were loading too much codebase context with every request.

> A workflow that keeps each session small is not just about accuracy; it is about making your token budget last.

The important difference is not just whether specs exist. It is whether the workflow preserves isolation when a single feature crosses repository boundaries.

## Getting Started: A Minimal Setup

You do not need a perfect system to start getting value from this.

1. **Create the hub**  
   Create a standalone `{platform}-specs` repository. Add `00-architecture-overview.md`, a status table, and one service spec for your primary API.

2. **Add repo-level context files**  
   Add `.github/copilot-instructions.md` to each service repo (or `CLAUDE.md` for Claude Code users, `AGENTS.md` for Codex) with stack, naming conventions, and module structure.

3. **Run one feature through the workflow**  
   Design the next feature in `Features/FEATURE-{name}.md`, cascade it into the relevant service specs, generate the prompts, and implement them in dependency order.

After a few features, the pattern becomes natural. The prompts improve because the specs get more explicit. The specs improve because the prompts expose what was previously underspecified.

## Closing

Every structural choice in this workflow follows from one overarching idea:

> Control what enters the model's context window. Put in the minimum necessary. Keep everything else out.

The five principles are applications of that idea: make prompts self-contained, isolate sessions per service, align session boundaries with context boundaries, reconcile immediately, and keep specs honest. The Spec Hub is not documentation infrastructure. It is context infrastructure.

The value of this workflow is not that it produces better prompts in isolation. It gives multi-repo AI work a structure: one source of truth, one prompt per service, one session per boundary, and one reconciliation step back into the specs. That is what keeps the system usable as complexity grows.

As these models get more capable, the work is shifting. Less time writing code, more time designing the workflows that produce it. That is not a loss of control. It is a different kind of control, and the sooner we get comfortable with it, the better.

