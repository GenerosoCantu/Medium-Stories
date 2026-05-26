# DSX Team: Token Conservation Guide

*How to get more done with AI coding agents without burning through your monthly allocation.*

---

## Why Tokens Run Out Fast

Token budgets disappear faster than expected for two reasons:

1. **Context bloat** — sessions load far more source code than the task actually requires
2. **Model mismatch** — using a premium model for tasks a lighter model handles equally well

Both are fixable with habits, not willpower. This guide covers both.

---

## Two Core Concepts

Everything in this guide follows from two ideas. Understanding them makes the rules easier to apply — and easier to extend to situations this guide does not cover.

### Context compression

A well-written spec or prompt compresses a codebase. Instead of asking an agent to scan a repo and infer its conventions, you write down the relevant patterns once and point the agent at two or three reference files. The agent reads those files, implements eight, and never touches the other two hundred.

This is not just about efficiency. It is about what the model is working from. An agent that infers conventions by scanning source has to make assumptions. An agent that reads an explicit spec works from facts. The compressed version is both cheaper and more reliable.

Context compression is the principle behind repo-level instruction files, self-contained prompts, and the Spec Hub pattern. All three are ways of encoding what the agent needs to know in a small, reusable artifact rather than letting the agent rebuild that knowledge from source on every session.

### Stateless prompts

A stateless prompt works in a fresh session with no prior history. It does not assume the agent remembers the feature design from two sessions ago, knows which pattern file was used last time, or recalls the naming convention you mentioned earlier in the week.

The failure mode of a stateful prompt is invisible: the agent appears to be working, but it is reconstructing missing context by reading more source, making reasonable-sounding assumptions, or quietly drifting from the established conventions. Each of those costs tokens. Some of them cost rework.

The test is simple: could someone else on the team — or future you — open a fresh session, apply this prompt to the correct repo, and get a consistent result? If not, the prompt is implicitly stateful and will be unreliable across sessions and team members.

---

## Part 1: Context Discipline

### Rule 1: Tell the agent exactly what to read

The single most expensive habit is asking an agent to understand a codebase before doing a task. Agents that are not given explicit file references will scan broadly — reading services, controllers, schemas, tests, and config files — burning hundreds or thousands of tokens before writing a single line.

**Instead of:**
> "Add a notifications endpoint to the API."

**Do this:**
> "Study `src/articles/articles.controller.ts` as the pattern reference. Create `src/notifications/` with the same module structure. Use these exact field names and guard patterns: [spec]."

The difference is not just accuracy. It is the difference between the agent reading 3 files or 40.

---

### Rule 2: Keep sessions short and single-purpose

Every message in a session carries the full conversation history. A session that starts with a feature design discussion, drifts into a bug fix, then pivots to a schema change is accumulating tokens on every single message — including the early ones that are no longer relevant.

**Session rules:**
| Situation | Action |
|---|---|
| Starting a new feature | New session |
| Switching from one service repo to another | New session |
| Ad-hoc question about a different part of the codebase | New session |
| Continuing work on the exact same task | Same session |
| You've been in a session for more than ~1 hour | Consider a fresh session |

When in doubt, start fresh. The cost of re-establishing context in a new session is almost always less than continuing to grow an old one.

---

### Rule 3: Never ask an agent to "explore" or "understand" a codebase

Phrases like these are token killers:
- "Take a look at the codebase and tell me how it's structured"
- "Familiarize yourself with this repo before we start"
- "Explore the existing patterns and then implement X"

You already know the codebase. Write down the relevant conventions and point the agent at the two or three files it needs. This is what repo-level instruction files and specs are for.

---

### Rule 4: Use repo-level instruction files

Each repo should have a `.github/copilot-instructions.md` (or `CLAUDE.md` for Claude Code, `AGENTS.md` for Codex) that captures the conventions an agent needs for ad-hoc work. This file auto-loads into sessions and prevents the agent from having to infer conventions by scanning source.

A minimal file covers:
- Stack and framework versions
- Naming conventions (class names, file names, route patterns)
- Module/folder structure
- Which libraries to use for common tasks (e.g., "use `crypto.randomUUID()`, not `nanoid`")
- Environment variable names

Keep it under 100 lines. Its job is to make local conventions unambiguous, not to document the entire system.

---

### Rule 5: Never span multiple service repos in one session

When a feature touches three services and you run it all in one session, the model is simultaneously holding source patterns from Service A, Service B, and Service C in context. This is expensive and produces lower quality output: the model starts blending conventions across codebases.

One session per repo. One repo per session. The only thing that moves between sessions is the prompt file.

---

### Rule 6: Write stateless prompts as files, not chat messages

A stateless prompt is self-contained: it works in a fresh session with no prior context. It answers five questions explicitly rather than relying on anything the agent may have seen before:
1. What files to study (the minimum set that establishes the pattern)
2. What files to create or modify (exact paths)
3. What the data model looks like (field names, types, constraints)
4. What the API looks like (endpoints, methods, guards, shapes)
5. What to name things (class names, DTO names, route paths, slice names)

The failure mode of a stateful chat prompt is that the agent silently reconstructs missing context — by scanning more source, making assumptions, or blending conventions from earlier in the conversation. That reconstruction burns tokens and produces inconsistent output.

When prompts are files, you can start a fresh session on any future day, hand the file to a colleague, or reapply it after a bug fix — all with zero ramp-up cost. Chat messages embedded in a long session are not reusable and are not stateless.

---

### Rule 7: Attach files selectively — not whole folders

When you attach files or drag them into context, be deliberate:
- Attach the specific file the agent needs, not the parent folder
- If only one function in a file is relevant, paste the snippet rather than the whole file
- Remove file attachments from context when you move on to a different task

Every attached file adds to the context window for every subsequent message in the session.

---

### Rule 8: Avoid redundant verification requests

Asking the agent to explain what it just did, summarize the changes it made, or confirm it understood your instructions costs tokens without adding value. If the output looks correct, move on.

Verification requests are useful when something looks wrong. Not as a default habit.

---

## Part 2: Model Selection

Not all tasks need the most powerful (and most expensive) model. Most token budgets are structured by model tier — requests to a premium model cost significantly more than requests to a standard one.

### Use a lighter model for:
- Simple code completions and autocomplete
- Renaming symbols, reformatting, or small refactors
- Writing boilerplate from a clear template
- Answering "how does X work" questions about your own codebase
- Generating test data or seed files
- Writing or editing documentation and comments

### Use a premium model for:
- Designing data models and API contracts
- Generating multi-file implementations from a spec
- Diagnosing complex bugs that span multiple layers
- Code that has security or correctness implications
- Tasks where a wrong first draft will cost significant time to fix

The rule of thumb: if you can verify the output in under a minute, a lighter model is probably fine. If a mistake would take an hour to unwind, use the best model you have access to.

---

## Part 3: Workflow Patterns That Scale

### The Spec Hub pattern (for multi-repo features)

If your team is working on a platform with multiple services, the highest-leverage thing you can do for token efficiency is to maintain a centralized spec workspace.

The pattern:
1. Design the feature once in a shared spec file (`Features/FEATURE-{name}.md`)
2. Cascade the design into each affected service spec
3. Generate one explicit prompt file per service (`Prompts/PROMPT-{service}-{feature}.md`)
4. Apply each prompt in a fresh session in the correct repo
5. Reconcile the final implementation back into the service specs

This prevents three expensive failure modes: agents scanning the wrong codebase for context, cross-service conventions bleeding together, and the same design decisions being re-derived in every session.

---

### The "three files in, eight files out" target

A well-written prompt should result in the agent reading three to five files as pattern references and creating or modifying eight to twelve files as output. If you notice an agent reading significantly more than that before it starts writing, the prompt is underspecified.

Good diagnostic question: *Could someone on the team apply this prompt in a fresh session and get a consistent result?* If the answer is no, the prompt needs more explicit contracts.

---

### Reconcile specs after every implementation

The most common cause of context bloat in the long run is stale specs. When specs drift from implemented reality, agents start finding contradictions and resolve them by reading more source. Keeping specs current keeps prompts reliable and sessions small.

Make reconciliation a closing step after every feature, not a quarterly cleanup.

---

## Part 4: Quick Reference

### Before starting a session
- [ ] Am I in the right repo for this task?
- [ ] Is this a new task or a continuation? (New task = new session)
- [ ] Do I have a prompt file, or am I relying on chat context?
- [ ] Which files does the agent actually need to read?
- [ ] Is the model tier appropriate for this task?

### Signs a session is getting expensive
- You have been in the same session for a long time and the task has shifted
- The agent is asking clarifying questions it should already know the answers to
- The agent is referencing conventions from a different service or earlier part of the session
- You have attached multiple files that are no longer relevant to the current task

### Signs a prompt needs improvement
- The agent reads many files before writing anything
- The agent asks for clarification about naming, structure, or field types
- Two people applying the same prompt get noticeably different results
- You had to correct the same mistake more than once

---

## Summary

| Practice | Token impact |
|---|---|
| Specify exact files to read in the prompt | High |
| Start a new session per task boundary | High |
| One session per repo for cross-service work | High |
| Use lighter models for simple tasks | High |
| Keep repo instruction files current and lean | Medium |
| Avoid "explore the codebase" requests | Medium |
| Attach files selectively, not whole folders | Medium |
| Skip redundant verification requests | Low |
| Reconcile specs after implementation | Low (prevents future cost) |

The underlying principle behind all of these is the same:

> Control what enters the model's context window. Put in the minimum necessary. Keep everything else out.

A session that starts clean, works from an explicit prompt, and ends when the task is done uses a fraction of the tokens of one that grows organically over hours. The habits that produce the first kind of session are learnable, and once they are routine, the savings compound across the whole team.
