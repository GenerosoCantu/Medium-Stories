# Starting a New Project with the Spec Hub Workflow

This guide walks you through setting up a brand new multi-service project using the Spec Hub workflow from day one. No existing code, no legacy context — just a blank slate and a plan.

---

## What You're Building

Before you write a single line of code, you'll set up **two things**:

1. A **Spec Hub repo** — a plain Markdown repository that holds your architecture decisions, service specs, and feature designs. No source code lives here.
2. **One repo per service** — your actual application code (API, frontend, etc.), each with a small instruction file for your AI assistant.

Think of the Spec Hub as the blueprint room. The service repos are the construction sites. Decisions happen in the blueprint room first, then get built on the sites.

---

## Before You Start: Answer These Questions

Write down the answers before touching any tool. You'll need them to fill in your first specs.

**About your system:**
- What services will this project have? (e.g., a REST API, a web frontend, a background worker)
- What does each service do in one sentence?
- How do the services talk to each other? (HTTP, events, shared DB?)

**About each service's tech stack:**
- What language and framework? (e.g., NestJS/TypeScript, Next.js/React)
- What database, if any?
- How is auth handled?
- How are IDs generated? (e.g., UUID v4, ObjectId, CUID)

**About conventions:**
- What does a typical module/feature look like in terms of folder and file structure?
- What naming patterns do you follow for classes, DTOs, routes, slices?

You don't need perfect answers. You need enough to write a first draft. You'll refine as you go.

---

## Step 1: Create the Spec Hub Repository

Create a new git repository called `{your-platform}-specs`. This is a standalone repo — do not put it inside a service repo.

```bash
mkdir my-platform-specs
cd my-platform-specs
git init
```

Create these files to start:

```
my-platform-specs/
├── 00-architecture-overview.md
├── CHANGELOG.md
├── Features/          ← empty folder for now
└── Prompts/           ← empty folder for now
```

### Write `00-architecture-overview.md`

This is your system map. Keep it short. It should answer: *what services exist, what each one does, and how they connect.*

```markdown
# Architecture Overview

## Services

| Service | Purpose | Tech |
|---|---|---|
| my-api | REST API, handles data and auth | NestJS / MongoDB |
| my-webapp | User-facing web app | Next.js / React |

## How They Connect
- my-webapp calls my-api over HTTP (REST)
- my-api issues JWT tokens; my-webapp stores and sends them

## Feature Status
| Feature | Status |
|---|---|
| (none yet) | — |
```

### Write a Service Spec for Each Service

Create one Markdown file per service: `01-my-api.md`, `02-my-webapp.md`, etc.

At this stage, the spec is short. You're capturing what you know before writing code:

```markdown
# my-api

## Stack
NestJS v10 / TypeScript / MongoDB (Mongoose) / JWT auth

## Conventions
- UUIDs generated server-side via crypto.randomUUID()
- All schemas: versionKey: false, timestamps: true
- Every schema has a tenant field (indexed, required)
- Guards: @UseGuards(JwtAuthGuard) on the controller class, not per-method
- Error handling: HttpException with descriptive messages

## Module Structure
src/{feature}/{feature}.module.ts
src/{feature}/{feature}.service.ts
src/{feature}/{feature}.controller.ts
src/{feature}/schemas/{feature}.schema.ts
src/{feature}/dto/create-{feature}.dto.ts
src/{feature}/dto/update-{feature}.dto.ts

## Endpoints
(none yet — will be added as features are built)

## Environment Variables
PORT=4000
MONGODB_URI=from env
JWT_SECRET=from env
```

Write a similar file for each service. At this point it's mostly stack and conventions — the endpoint tables and schema sections will fill in as you build features.

---

## Step 2: Create the Service Repositories

Create each service's git repo as you normally would (scaffold with a CLI, create via GitHub, etc.).

```bash
npx @nestjs/cli new my-api
npx create-next-app my-webapp
```

### Add a Repo-Level Instruction File to Each Service Repo

This file auto-loads into your AI assistant when you open that repo. It tells the AI the local rules so it doesn't have to guess.

For GitHub Copilot, create `.github/copilot-instructions.md`.  
For Claude Code, create `CLAUDE.md` in the project root.  
For OpenAI Codex, create `AGENTS.md` in the project root.

The content is the same regardless of which file you use:

```markdown
# my-api — Copilot Instructions

## Stack
NestJS v10 / TypeScript / MongoDB (Mongoose) / JWT auth

## Conventions
- UUIDs via Node crypto.randomUUID() — never use nanoid or uuid package
- All schemas: versionKey: false, timestamps: true
- tenant field on every schema, indexed
- Guards: @UseGuards(JwtAuthGuard) on controllers, not individual methods
- Error handling: standard NestJS HttpException with descriptive messages

## Module Structure
src/{feature}/{feature}.module.ts
src/{feature}/{feature}.service.ts
src/{feature}/{feature}.controller.ts
src/{feature}/schemas/{feature}.schema.ts
src/{feature}/dto/create-{feature}.dto.ts
src/{feature}/dto/update-{feature}.dto.ts

## Environment Variables
PORT=4000
MONGODB_URI=from env
JWT_SECRET=from env
```

Copy the relevant parts of each service spec into the corresponding instruction file. They should say the same thing. The spec is the source of truth; the instruction file is a convenience copy for the AI to load automatically.

> **Keep these in sync.** When a convention changes, update both the spec and the instruction file.

---

## Step 3: Build Your First Feature

Now the project is wired up. Here's how you build the first real feature.

### 3a. Design (in the Spec Hub)

Open the Spec Hub in a **new AI session**. Create `Features/FEATURE-{name}.md`.

Write down:
- What the feature does
- Which services it touches
- The data model (fields, types, required/optional, defaults)
- The API shape (endpoints, HTTP verbs, who can call them)
- Any contracts between services (e.g., "the frontend expects this exact response shape")
- Anything you haven't decided yet — list open questions explicitly

```markdown
# FEATURE: user-profiles

## Summary
Users can set a display name, bio, and avatar URL on their profile.
The frontend shows a profile page and an edit form.

## Affected Services
- my-api: new /profiles endpoints, new Profile schema
- my-webapp: new profile page, edit form, Redux slice

## Data Model
Profile:
  _id: String (UUID v4)
  userId: String (required, unique)
  displayName: String (required)
  bio: String (optional, max 300 chars)
  avatarUrl: String (optional)
  updatedAt: Date (auto-managed)

## API Shape
GET    /profiles/:userId    public       fetch profile
PUT    /profiles/:userId    JWT guard    create or update own profile

## Cross-Service Contract
Frontend sends PUT with { displayName, bio, avatarUrl }
API returns the full Profile object

## Open Questions
- [ ] Should avatars be uploaded to the API or a separate CDN?
```

At this stage, don't touch the service specs. The feature file is where ideas are still in flux.

### 3b. Cascade (still in the Spec Hub)

Once you're happy with the design, copy the relevant pieces into each affected service spec and mark them `[PENDING]`.

In `01-my-api.md`, add:

```markdown
## Endpoints
| Method | Path | Guard | Description |
|---|---|---|---|
| GET | /profiles/:userId | None | Fetch user profile |
| PUT | /profiles/:userId | JWT | Create or update own profile |
<!-- [PENDING: FEATURE-user-profiles] -->

## Schemas
### Profile [PENDING: FEATURE-user-profiles]
- _id: String (UUID v4, server-generated)
- userId: String (required, unique)
- displayName: String (required)
- bio: String (optional)
- avatarUrl: String (optional)
```

In `02-my-webapp.md`, add:

```markdown
## State Slices
### userProfile [PENDING: FEATURE-user-profiles]
- Stores: { profile: Profile | null, status: 'idle' | 'loading' | 'error' }
- Actions: fetchProfile, updateProfile

## Views [PENDING: FEATURE-user-profiles]
- /profile/:userId — read-only profile page
- /profile/edit — edit form (own profile only)
```

Also update `00-architecture-overview.md`:

```markdown
## Feature Status
| Feature | Status |
|---|---|
| user-profiles | PENDING |
```

### 3c. Generate Prompts (still in the Spec Hub)

Create one prompt file per service. Each prompt must be completely self-contained — the AI reading it should be able to implement the feature without asking you anything.

**`Prompts/PROMPT-my-api-user-profiles.md`**

```markdown
# PROMPT: my-api — user-profiles

## Dependencies
None. This is the first prompt to apply.

## Files to Study (read these first)
- src/app.module.ts
- src/articles/articles.module.ts       ← use as pattern reference
- src/articles/articles.service.ts
- src/articles/articles.controller.ts

## Files to Create
- src/profiles/profiles.module.ts
- src/profiles/profiles.service.ts
- src/profiles/profiles.controller.ts
- src/profiles/schemas/profile.schema.ts
- src/profiles/dto/upsert-profile.dto.ts
- src/app.module.ts                     ← register ProfilesModule here

## Schema Contract
Profile:
  _id: String (UUID v4, server-generated via crypto.randomUUID())
  userId: String (required, unique index)
  displayName: String (required)
  bio: String (optional)
  avatarUrl: String (optional)
  timestamps: true
  versionKey: false

## Endpoint Definitions
GET  /profiles/:userId    No guard    returns Profile or 404
PUT  /profiles/:userId    JwtAuthGuard  upsert (findOneAndUpdate with upsert: true)

## Naming Rules
- Schema class: ProfileSchema / Profile
- Service class: ProfilesService
- Controller class: ProfilesController
- Module class: ProfilesModule
- DTO class: UpsertProfileDto
```

**`Prompts/PROMPT-my-webapp-user-profiles.md`**

```markdown
# PROMPT: my-webapp — user-profiles

## Dependencies
Apply PROMPT-my-api-user-profiles.md and verify the API is running first.
Expected endpoints:
- GET /profiles/:userId → Profile object or 404
- PUT /profiles/:userId → upserted Profile object

## Files to Study (read these first)
- src/store/articlesSlice.ts            ← use as Redux slice pattern
- src/pages/articles/[id].tsx           ← use as page pattern
- src/components/ArticleForm.tsx        ← use as form pattern

## Files to Create
- src/store/userProfileSlice.ts
- src/pages/profile/[userId].tsx
- src/pages/profile/edit.tsx
- src/components/ProfileForm.tsx

## State Shape
userProfile slice:
  profile: { userId, displayName, bio, avatarUrl } | null
  status: 'idle' | 'loading' | 'error'
  error: string | null

## Naming Rules
- Redux slice: userProfile
- Thunks: fetchUserProfile, updateUserProfile
- Page component (view): UserProfilePage
- Page component (edit): EditProfilePage
- Form component: ProfileForm
```

**Key rule:** If you cannot write the prompt without vague language like "follow existing patterns" or "use a similar approach," the spec is not specific enough yet. Go back and tighten the spec before generating the prompt.

### 3d. Implement (in each service repo, one at a time)

Open your first service repo. Start a **fresh AI session**. Paste in the prompt or load the prompt file. Let the AI implement. Review and test.

Do not move to the next service until the current one is working. The dependency chain matters: the frontend prompt assumes the API is live.

**Rules for this step:**
- One session per service. Do not mix repos in one session.
- If something doesn't match the spec, decide whether to fix the code or update the spec — but do one or the other, not neither.
- Keep notes on any deviations (fields renamed, endpoints adjusted, etc.) — you'll need them in the next step.

### 3e. Close (back in the Spec Hub)

Open the Spec Hub in a **new AI session**. Do the following:

1. **Update the service specs** to reflect actual implementation. Remove all `[PENDING]` markers.
2. **Record any deviations.** If a field was renamed or an endpoint changed during implementation, update the spec to match what was built — not what was planned.
3. **Update `00-architecture-overview.md`** — change the feature status from PENDING to SHIPPED.
4. **Prepend to `CHANGELOG.md`:**
   ```markdown
   ## 2026-04-15 — user-profiles
   - Added GET /profiles/:userId and PUT /profiles/:userId to my-api
   - Added userProfile Redux slice and profile pages to my-webapp
   - Note: bio field max length not enforced server-side (deferred)
   ```
5. **Delete `Features/FEATURE-user-profiles.md`.** The design is now absorbed into the service specs. The feature file would only create confusion if left around.

---

## What Your Project Looks Like After a Few Features

After two or three features, your Spec Hub will contain:
- A filled-in architecture overview with a status table
- Dense service specs that serve as accurate documentation of what's actually built
- A growing `Prompts/` folder of reusable, version-controlled prompt files
- A `CHANGELOG.md` that doubles as a project history

Your service repos will each have a tight instruction file that makes ad-hoc AI work inside those repos reliable without needing the full Spec Hub context.

---

## Common Mistakes to Avoid

**Skipping cascade and going straight to prompting.**  
The prompt is generated from the spec. If the spec doesn't have the detail, the prompt won't either, and the AI will start inventing names and shapes.

**Designing in the service repo instead of the Spec Hub.**  
Cross-service decisions made inside one service's context will be inconsistent with the other services. Keep design decisions above the repos.

**Leaving `[PENDING]` markers in the spec forever.**  
The close step is not optional. Specs that lag behind implementation stop being useful. Set a rule: the close step happens the same session or the same day as implementation.

**Running one big session across multiple repos.**  
The context window fills up, conventions from different codebases get mixed, and the output quality drops. One repo, one session.

**Keeping the feature file after closing.**  
Once the design is in the service specs, the feature file becomes a source of conflicting truth. Delete it.

---

## Quick Reference: What Goes Where

| Content | Lives in |
|---|---|
| Architecture decisions | Spec Hub: `00-architecture-overview.md` |
| Service stack, conventions, module structure | Spec Hub: `{N}-{service}.md` AND service repo: `.github/copilot-instructions.md` |
| Active feature design (in progress) | Spec Hub: `Features/FEATURE-{name}.md` |
| Implementation prompts | Spec Hub: `Prompts/PROMPT-{service}-{feature}.md` |
| Source code | Service repos only |
| Project history | Spec Hub: `CHANGELOG.md` |
