# SpecHub, Part 2: What Fifty Features Taught Me About Running a Spec-First AI Workflow

*Part 1 introduced SpecHub: a spec workspace above your service repos, and a five-step loop (Design → Cascade → Prompt → Implement → Close) that keeps an AI agent precise across a multi-repo platform. This part covers everything that only showed up under sustained use: scaling the specs, packaging the procedures, keeping the hub honest, the token economics, and how it compares to the tools.*

---

## Where Part 1 left off

The short version, if you're landing here first: my platform's specifications live in their own workspace, the Spec Hub, separate from every service repo. Specs are treated as context compression: a 200-line contract document the agent reads instead of 2,000 lines of source. Every feature moves through five steps: design it in a feature file, cascade the decisions into the service specs, generate one self-contained prompt per service, implement each prompt in a fresh session inside its repo, then reconcile the specs against the code that actually shipped. ([Part 1 is here](https://medium.com/@gcantuw/spechub-part-1-how-i-ship-features-across-three-repos-with-an-ai-agent-and-stay-sane-edb6a1db8ffa).)

That's the core. What follows is the part no clean-room description gives you: what the system looks like after roughly fifty features of real use, and which extra pieces earned their place by fixing real friction.

---

## When a spec outgrows the window: split specs

Spec-as-compression buys you a long runway, but not an infinite one. Around forty features in, my two largest specs (the backend API and the admin web app) had each grown past the point where "read the spec" meant dragging a few thousand lines into context to answer a question about one module. The compression was still real; there was just too much of it in one file. The failure mode creeps back in a new costume: the agent reads twenty modules to work on one.

The fix is the same idea applied one level down: the spec itself becomes an index over per-module files.

```text
03-main-api.md            ← index: ~50 lines, a table of modules
03-main-api/
├── 00-core.md            ← stack, auth, cross-cutting behavior (~440 lines)
├── 01-environment.md     ← env/config, read only when config is involved
├── authors.md            ← one file per module (21 of them)
├── stories.md
├── sections.md
└── …
```

The routing rule that comes with it does the real work: for any module task, the agent reads the index, the directory's core file, and only the module file(s) the task touches, never the whole directory. A new module gets its own file plus one row in the index table; module content never migrates back into the index.

The effect is that context load per task stays roughly constant as the system grows. Working on the stories module costs the same window whether the platform has ten modules or forty, because the agent's required reads are index + core + `stories.md` either way. Both big specs are split this way now (21 and 22 module files respectively); the five smaller services still fit comfortably in single files, and stay that way until they don't.

So does spec-as-compression scale? Yes, but only if you're willing to apply the compression recursively. A monolithic 3,000-line spec is just a smaller version of the codebase-scanning problem it was meant to solve.

The same pattern reached the hub's top level. `CONVENTIONS.md` and `STATUS.md` both started life as sections of the architecture overview and were extracted the day I noticed the agent loading a full architecture document just to check a naming rule or a feature's status. Every distinct kind of question gets the smallest file that can answer it.

---

## The repo-level instruction file: a safety net

The Spec Hub is the authority, but people still ask ad-hoc questions inside a repo: "why is this guard failing?", "add a field to this DTO." The lean instruction file in each service repo (`CLAUDE.md` for Claude Code, `.github/copilot-instructions.md` for Copilot, `AGENTS.md` for Codex) is the safety net for those moments. It auto-loads into every session in that repo and supplies the stable, repo-wide conventions the agent needs to avoid obvious mistakes, and nothing feature-specific, so there's almost nothing in it that can drift from the hub.

Here's a representative one for a NestJS API:

```markdown
# my-api — Agent Instructions

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
```

Its job is not to carry feature design; that's what prompts are for. It exists to make local conventions unambiguous, so even an off-the-cuff request lands in the right shape. One rule keeps it trustworthy: the canonical copy of every instruction file lives in the hub's `repo-instructions/` folder. You edit there and copy into the repo, never the other way around, so the one artifact that could silently drift has a master copy under the hub's control.

---

## Skills: the workflow's procedures, packaged

Specs and skills are complementary because they operate on different things. A spec encodes a contract: what the system is. A skill encodes a repeatable procedure: how to perform an activity. When I first wrote about this workflow, packaging its routines as skills was a suggestion; it's now how the hub actually runs, and the two that exist map exactly onto the two steps with the most sub-steps to forget:

- **`generate-prompts`** (Step 3) encodes the required reads (conventions, feature file, spec index + core + only the touched module files), the preconditions (no open design decisions, cascade complete, implementation order declared), the dispatch-header format, and the self-containment quality gate.
- **`close-loop`** (Step 5) encodes the whole close-out checklist: the code-level drift check, spec reconciliation, the one-line status flip, the changelog entry with commit refs, archival, and the repo-instructions sync.

The payoff is twofold. The procedure stays out of the context window until it's actually invoked; an agent answering a quick spec question never carries the close-out checklist as baggage. And every run is identical instead of re-improvised, which matters most for close-out, where a forgotten sub-step is precisely how drift starts. Specs say what; skills say how to operate on it. A mature setup uses both.

---

## Keeping the hub honest

A Spec Hub has exactly one existential failure mode, drift: the specs claiming contracts the code silently changed. Three disciplines emerged to hold the line, each after watching something fail.

**Close-out verifies against the code, not against memory.** The first move of every Step 5 is opening the actual service repo (the prompt's dispatch header says where) and reading the files the prompt named (schema, routes, components) to confirm the spec's claims match what was really built. Where they don't, reality wins, and the deviation gets noted with the implementing commit SHAs. Early on, close-out was a trust exercise done from the hub side; a spec is only a source of truth if something regularly forces it to be true.

**One document accumulates history; everything else describes current state.** Close-out summaries kept landing in the `STATUS.md` Notes column until the "status board" was forty rows of paragraph-length history nobody could scan. A board you can't scan isn't a board. So the rule hardened: status rows get a one-line note, ever; the detail goes into `CHANGELOG.md` as one dated entry with commit refs, and the changelog is the only file in the hub allowed to accumulate dated entries. That sounds like overkill until you've deleted the same creeping history block from three different specs.

**Parked designs get their own state.** Some features get deprioritized indefinitely or blocked on a decision nobody is ready to make. Leaving them in `Features/` makes them read as active; archiving them makes them read as shipped. So they park in `Features/Staled/`, and any `PENDING` markers they left in the specs are removed. Reviving one requires re-validating it against the current specs first: the system moved while it slept, and prompts must never be generated from a stale design.

---

## The sixth principle: right-size the model to the prompt

Part 1 ended with five principles and a promise of a sixth: a fully-specified task runs on the cheapest model tier that fits, decided at design time rather than session time.

Not every implementation session deserves the same brain. A schema field plus a CRUD endpoint is a cheap-model task; an intricate drag-and-drop editor UI is not. So every feature file carries a per-service model recommendation, and every generated prompt inherits it in its dispatch header:

```markdown
> **Recommended model:** Haiku 4.5 — backend-only, straightforward CRUD
```

This turned out to be one of the simplest cost levers in the whole system. Most implementation sessions on my platform run on the cheapest tier, because a prompt that names every file, field, and convention has already done the hard part. Routing a fully-specified CRUD task to a premium model is paying flagship prices for typing. The expensive model is reserved for the sessions that actually need it, and the feature file records that decision once instead of leaving it to whoever opens the session.

Context control compounds this, because it's a billing concern as much as an accuracy one. Every serious coding agent now meters usage in tokens, and I've watched people burn through a monthly allocation faster than expected, largely because every request dragged too much codebase into context. A workflow that keeps each session small protects your token budget as much as your precision.

But be honest about where the savings come from, because the hub isn't free. Designing the feature file, cascading it into the specs, and reconciling at the end all cost tokens too, so a single one-off change in one repo can easily cost more under this workflow than just prompting the agent directly. The savings are real but they're amortized: a spec earns its keep when the same contract is read by several services, reused across many fresh sessions, or handed to other people. The break-even is reuse. For a solo change in a single repo, skip the ceremony and lean on the cheap half: an explicit, file-referenced prompt and a current repo instruction file capture most of the per-session savings on their own.

---

## "Isn't this just Spec Kit?" (and the sharper question: "isn't this OpenSpec?")

It's the natural objection, and the answer sharpens what SpecHub is for. Tools like GitHub Spec Kit and AWS Kiro popularized spec-driven development, and they're good at what they do, but they're single-repo by design. They answer "how do I give my agent enough context to build a feature in this project?" They don't answer "how do I coordinate one feature across five services and three repos without duplicating context or tangling sessions?" Spec Kit can handle per-feature implementation phases *inside* a repo while a Spec Hub handles the cross-repo coordination above it; the two coexist naturally.

The sharper 2026 objection is OpenSpec, whose Stores concept puts planning in a dedicated repo shared across projects: a spec layer above the repos, which is genuinely the same structural idea. I take that as convergent evidence the shape is right, not as a reason to switch. Two real differences remain. First, tooling versus convention: OpenSpec gives you a CLI that scaffolds and validates spec structure; SpecHub is plain Markdown plus two skills, which means nothing to install and nothing to outgrow, but also no mechanical validation. The close-out drift check is my substitute, and a CLI-style validator is a reasonable thing to add if your hub grows contributors. Second, the parts of this workflow I lean on hardest (the dispatch header with its status lifecycle, per-service model routing, the split-spec index pattern, the staled state, canonical repo-instruction copies) are exactly the parts that came from this platform's fifty features and exist in no tool.

If you're starting from zero and want guardrails out of the box, evaluate OpenSpec seriously. If your coordination problems are cross-repo and your conventions are your own, a hub you fully control is hard to beat, and it's trivially portable to whatever agent wins next year.

---

## Where this sits among the alternatives

It helps to see the workflow next to the things people reach for instead. The comparison is qualitative (workflow tradeoffs, not benchmarks):

```text
Approach                           Context   Convention    Cross-repo    Main failure mode
                                   load      reliability   coordination
──────────────────────────────────────────────────────────────────────────────────────────────────────────
Centralized Spec Hub               Low       High*         Strong        Specs drift if not reconciled
Per-repo spec workflow             Low–med   High in-repo  Weak          Cross-service deps stay implicit
Per-service spec copies per repo   Medium    Medium        Weak          Design drift across copies
Direct prompting inside a repo     High      Med–low       Weak          Agent infers from too much source
Cross-service work in one session  Very high Low           Weak          Attention split across codebases
──────────────────────────────────────────────────────────────────────────────────────────────────────────
* if specs stay current
```

The tradeoff to respect is the top row's own failure mode: a Spec Hub is only as good as your discipline in reconciling it. That's the entire reason Step 5 is non-negotiable, and why its first move is a check against the code rather than a trust exercise.

---

## Let your hub earn its complexity

Nothing in this article was designed in advance. Split specs showed up when two files outgrew the window. The skills came after the third forgotten close-out sub-step. The staled state was invented when a parked design kept reading as active work. The one-line status rule followed the day the board stopped being scannable. Model routing arrived with the first surprising bill.

That's the meta-lesson of fifty features: start with the skeleton from Part 1 (a hub, one spec, one feature through the loop) and add each of these pieces only when you feel the friction it fixes. The prompts get sharper because the specs get more explicit, and the specs get more explicit because the prompts keep exposing whatever was underspecified.

A complete example Spec Hub, a fictional five-service storefront platform ("Plaza") with one cross-service feature traced end to end, is on GitHub: [github.com/GenerosoCantu/SpecHub-Example](https://github.com/GenerosoCantu/SpecHub-Example).

---

*Written from a real multi-service content platform with six services, two frontend applications, and a shared MongoDB Atlas cluster, roughly fifty features into running this workflow. The instruction file and dispatch header shown above are lightly trimmed but otherwise verbatim from the workspace they describe. Drafted with AI assistance, working from my own specs, session notes, and the artifacts above.*

---

### Publishing notes (delete this section before importing to Medium)

- Push to GitHub, then use Medium's **Import a story** with the file URL.
- The intro links to the published Part 1 URL; add Part 2's URL to Part 1's intro/closing once this part is published.
- The approach-comparison block is wide; on mobile it will scroll horizontally. If you want it polished, export that single block as an image and add alt text.
- After import, verify the fenced blocks rendered as Medium code blocks; apply Code Block formatting to any that flattened.
- Replace the title/subtitle with Medium's native title + subtitle fields.
