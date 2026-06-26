# SpecHub — Video Intro

*Companion video to the SpecHub article. Demo feature: adding estimated reading time ("3 min read") to an article.*

---

## Intro Script

> Hi. In this short video I want to show you a workflow I call **SpecHub** — and rather than explain it in the abstract, I'll walk you through one real feature, start to finish.
>
> Here's the problem it solves. AI coding assistants are great in a demo: one repo, one prompt, working feature. But on a real platform — where a single feature might touch a frontend, a backend, and a delivery service — things start to drift. The agent invents conventions, mixes assumptions between services, and every new session starts from scratch.
>
> SpecHub fixes that, and it's not a tool you install — it's a way of organizing your specs. The core idea is simple: your **specifications live in their own workspace, above the code repos**. That hub becomes the single source of truth. Every feature flows through the same short loop — you **design** it, **cascade** the decisions into the relevant spec, **generate** a precise prompt from that spec, and **implement** it. The agent never reads the whole codebase; it works from a tight, deliberate slice — which keeps it accurate and your token usage low.
>
> To keep this grounded, I'll cascade a small, easy-to-follow feature on a **personal project I keep around for trying out new tools and techniques** — exactly this kind of workflow experiment. The feature: **adding the estimated reading time to an article** — the little "3 min read" you see on most blogs. It's deliberately simple, so the focus stays on the *workflow*, not the code.
>
> Let's get into it.

---

## Step 1 — What to say to the LLM to get started

> I'm in the Spec Hub workspace, and I will start by defining the new feature. I will start to dictate the feature on a new Claude Code session.

"I want to add a new feature on all articles, and it's the estimated reading time for the article. I want the text at the top of the article that says '3 min read'. Suggest a formula to calculate the read time. Also include some time for each video and photo that is included inside the article. Make sure that only the photos and media files that are included in the article are considered. In other words, if I have three pictures loaded but I'm only using one in the text, the other two won't be considered in the calculations." 


