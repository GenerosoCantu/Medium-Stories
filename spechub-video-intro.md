# SpecHub — Video Intro

*Companion video to the SpecHub article. Demo feature: adding estimated reading time ("3 min read") to an article.*

---

## Intro Script

> Hi. In this short video I will show you how the SpecHub workflow works. Rather than explain it in the abstract, I'll walk you through one real feature, start to finish.
>
> First of all, what is problem that SpecHub solves? AI coding assistants are great in a demo: one repo, one prompt, working feature. But on a real platform — where a single feature might touch a frontend, a backend, and a content delivery service — things start to drift. The agent invents conventions, mixes assumptions between services, and the context you built up either resets on a new session or slowly drifts in one that's been open too long.
>
> SpecHub fixes that, and it's not a tool you install — it's a way of organizing your specs. The core idea is simple: your **specifications live in their own workspace, above the code repos**. That hub becomes the single source of truth. Every feature flows through the same short loop — you **design** it, **cascade** the decisions into the relevant spec, **generate** a precise prompt from that spec, and **implement** it. The agent never reads the whole codebase; it works from the spec files for each service in the system, which hold the compressed knowledge of the application — which keeps it accurate and your token usage low.
>
> To keep this simple, I'll cascade a small, easy-to-follow feature on a **personal project I keep around for trying out new tools and techniques**. The feature is: **adding the estimated reading time to an article** — the little "3 min read" you see on most blogs or news websites. It's deliberately simple, so the focus stays on the *workflow*, not the code.
>
> And just to be clear — even though I'm showing a small example here, I've used this same workflow on much more complex features, and it works without any problem.
>
> Let's get into it.

---

## Step 1 — What to say to the LLM to get started

> I'm in the Spec Hub workspace, and I will start by defining the new feature. I will start to dictate the feature on a new Claude Code session.

"I want to add a new feature on all articles, and it's the estimated reading time for the article. I want the text at the top of the article that says '3 min read'. I also want a preview on edit story page of the front end web app. Suggest a formula to calculate the read time. Also include some time for each video and photo that is included inside the article. Make sure that only the photos and media files that are included in the article are considered. In other words, if I have three pictures loaded but I'm only using one in the text, the other two won't be considered in the calculations." 

---

## Step 2 — A quick tour of the Spec Hub

> While the agent works on that, let me show you what actually lives in this workspace — because this is where the whole approach comes from. There's no source code here. Just specs.

> At the top sits the **architecture overview**. Think of it as the map of the whole platform: what the services are, how they talk to each other, the shared conventions every service has to follow — API naming, how we store files, env vars — and a status table at the bottom listing every feature that's in flight and which services still have work pending. This is the file the agent reads first whenever it needs a system-wide answer.

> Then there's **one spec file per service** — the frontend, the web app, the main API, the CDN service, and so on. Each one is the compressed knowledge of that service: its modules, its endpoints, its data contracts. Not the code itself — the *shape* of the code. That's the trick: a spec is small enough to fit in the agent's context, but it tells the agent everything it needs to know to work on that service correctly, without ever reading the repo.

> Now, what actually teaches the agent how all of this fits together? Right here in the hub there's a **`.github/copilot-instructions.md`** — or a **`CLAUDE.md`** if you're using Claude Code. This is the file that holds the knowledge of how the SpecHub flow works: where the specs live, the order to read them in, and how a feature moves through the loop. And for the full detail, it points the agent at **WORKFLOW.md**.

> That **WORKFLOW.md** spells out the process itself — design, cascade, generate the prompt, implement — and sits next to a changelog that records what's already shipped.

> And here's the important part: these instructions are specific to the hub — they only live here, and they only describe the SpecHub flow. Each code repo has its *own* `copilot-instructions.md`, and it's a completely different file with a different job. It says nothing about the workflow. Instead it holds that service's stable, repo-wide conventions — its naming, its patterns, its details — and points the agent at the right spec to read before it touches anything. And because that file gets pulled into every new session automatically, a fresh agent with no memory routes itself straight to the right context. No re-explaining the architecture every time. The knowledge is already there, waiting.


