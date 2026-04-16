---
title: "How We Built a Custom AI Workshop That Actually Worked"
date: 2026-04-15
categories: [AI, Workshops]
tags: [copilot-studio, azure-ai-foundry, mcp, workshops, federal]
description: "Lessons from building and delivering a hands-on Copilot Studio + Azure AI Foundry workshop for a large federal customer — from repo to room in 10 days."
image:
  path: /images/federal-copilot-mcp-hero-web.png
---

Most AI workshops I've attended follow the same formula: two hours of slides about "the art of the possible," a canned demo that only works on the presenter's laptop, and a follow-up email nobody reads. Everyone leaves inspired but unchanged.

We wanted to build something different. Something where every person in the room walks out with a working agent — not a screenshot of someone else's.

Here's how we did it, what worked, what didn't, and what I'd do differently next time.

## The Brief

A federal customer wanted their technical and leadership teams to understand what's actually possible with Microsoft's AI stack — specifically Copilot Studio, Azure AI Foundry, and how to connect the two securely. Not theory. Not slides. Working code they could extend after we left.

We researched the customer's environment, collaborated across a team of Microsoft engineers, and iterated on the content using GitHub Copilot to accelerate development. The whole repo went from first commit to workshop-ready in about 10 days.

The constraints:
- **GCC tenant** — government cloud, which means half the features you see in demos don't work the same way
- **Mixed audience** — executives who needed to understand the "why" and engineers who needed to build the "how"
- **One day** — 6 hours total, including lunch
- **Zero local installs** — participants couldn't install software on their machines
- **Customer environment uncertainty** — we didn't have full access to their tenant until day-of

## The Approach: Least to Most Technical

We structured the day in three blocks, ordered from most accessible to most technical:

**Session 1 — Copilot Studio (Everyone)**
Build an AI agent from scratch. No code required. Executives and engineers side by side, same exercise. By 11 AM, every person in the room had a working agent that could answer questions about their domain.

**Lunch — Executive Working Session**
While the technical team ate, our account lead ran a separate strategy discussion with leadership: what they saw in the morning, where it fits in their roadmap, what the pilot looks like.

**Session 2 — Azure AI Foundry (Technical)**
The builders stayed. We went deeper — prompts, completions, responses API, agents, multi-agent patterns. Hands-on Python labs where participants ran real code against real models.

**Session 3 — Use Case Development (Everyone Rejoins)**
Brainstorm + team breakout. "Pick a use case from your organization, design an agent for it, present back." This is where it stopped being our workshop and started being their roadmap.

## The Repo Is the Workshop

Every workshop I've ever attended dies the moment the presenter closes their laptop. We built ours differently: **the GitHub repo is the single source of truth.**

```
Workshop-Repo/
├── GETTING-STARTED.md          # Step 0 → Step done
├── workshop/
│   ├── agenda.md               # Master schedule
│   ├── lab1-copilot-studio.md  # Full lab guide with screenshots
│   ├── lab2-azure-foundry.md   # Foundry labs
│   └── setup-environment.md    # 3 environment options
├── boilerplate/
│   └── mcp-backend/            # Production MCP server + deploy scripts
├── slides/                     # Reveal.js presentation
├── .devcontainer/              # GitHub Codespaces config
└── just.config.js              # Task runner for all workshop commands
```

Participants clone the repo, follow the guides, and run `npx just` commands. If they get stuck, the answer is in the markdown. If they want to revisit it in two weeks, the repo is still there.

We offered three environment options so nobody was blocked:

| Option | For Who | Setup Time |
|--------|---------|-----------|
| **GitHub Codespaces** | Everyone (recommended) | ~2 min |
| **vscode.dev/azure** | Those with Azure accounts | ~3 min |
| **Local install** | Experienced devs | ~15 min |

Zero installs for 90% of the room. That decision alone probably saved us an hour of troubleshooting.

## GCC: The Silent Workshop Killer

If you're running AI workshops in government environments, here's what nobody tells you until the demo breaks:

**Endpoints are different.** Every `login.microsoftonline.com` becomes `login.microsoftonline.us`. Every `graph.microsoft.com` becomes `graph.microsoft.us`. Copilot Studio lives at `gcc.powerva.microsoft.us`, not the commercial URL. Miss one and your auth silently fails.

**Features are gated.** Bing Search? Disabled by default. AI Builder Vision? Disabled. Exchange OData access for Copilot? Blocked by allowlist. Each of these showed up as a surprise during the live session. We adapted in real time, but I'd rather have known in advance.

**The pivot nobody saw coming:** On workshop morning, we discovered the customer's environment had restrictions we couldn't work around in time — admin-gated features, blocked user agents, disabled capabilities. Instead of spending the first hour troubleshooting, we made a call: spin everything up on our own tenant. Within minutes, we had a working environment and the workshop continued without a hitch. The participants never felt the scramble.

This is why the repo-first approach matters. Because the entire workshop was code and markdown — not slides and prayers — we could redeploy the whole thing to a different tenant and keep moving. If we'd built the workshop around a specific pre-configured environment, we'd have been dead in the water.

**My recommendation:** Always have a fallback tenant ready. Do a full dry run in the actual customer environment at least 48 hours before, but assume something will break. The feature matrix between commercial and GCC is a moving target.

## What Worked

**The repo-first approach.** Participants who fell behind could catch up asynchronously. Two people emailed the week after saying they'd gone through the labs again on their own.

**Least-to-most technical ordering.** Executives got value in the first two hours and left at lunch feeling informed, not overwhelmed. Engineers got the deep dive they wanted in the afternoon without executives asking "but what does this mean for our budget?" every 10 minutes.

**The use case brainstorm.** This was the highest-energy part of the day. Teams pitched agent ideas specific to their actual workflows. One team designed a document processing assistant. Another built a concept for automated cross-referencing of internal policies. These weren't our ideas — they were theirs. That's the point.

**Task runner abstraction.** `npx just provision` instead of "now run these 14 Azure CLI commands." The `just-task` setup meant we could hide complexity without removing it. Participants who wanted to see what was happening under the hood could read `just.config.js`.

## What I'd Do Differently

**Pre-flight the GCC tenant harder.** We hit three admin-gated features during the live session. All fixable, but each one cost us 5 minutes of momentum. A comprehensive GCC checklist would have caught these.

**Record the sessions.** We didn't, and multiple participants asked for recordings afterward. Even a simple screen capture would have been valuable.

**Add a "take-home" lab.** The workshop covered a lot of ground. A structured post-workshop exercise — "deploy your use case agent by Friday" — would have extended the learning beyond the room.

**More buffer time.** We had the schedule tight. Everything fit, but there was no slack for the inevitable "my login isn't working" moments that eat 10 minutes.

## The Pattern

If you're building a similar workshop, here's the template:

1. **Start with the repo.** If it's not in the repo, it doesn't exist. Slides are secondary.
2. **Three environment options.** Codespaces, cloud shell, local. Meet people where they are.
3. **Order by audience, not by architecture.** Put the accessible stuff first. Let executives leave feeling smart. Let engineers stay and go deep.
4. **Use a task runner.** `just`, `make`, whatever — abstract the complex stuff behind simple commands.
5. **Separate the executive track.** A working lunch or breakout where leadership discusses strategy without slowing down the technical labs.
6. **End with their ideas, not yours.** The brainstorm session is where ownership transfers. That's the real deliverable.

## The Stack

For reference, here's what we used:

- **Copilot Studio** (GCC) — Agent builder, no-code
- **Azure AI Foundry** — GPT-4o, RAG, agent service
- **Azure API Management** — JWT validation, secure gateway
- **MCP (Model Context Protocol)** — Structured tool calling via Streamable HTTP
- **GitHub Codespaces** — Zero-install dev environments
- **just-task** — Microsoft's JS task runner
- **Reveal.js** — Presentation slides (in the repo, versioned with the code)

Everything is open source or uses standard Azure/M365 licensing. No special SKUs required.

---

*The best workshop isn't the one with the best slides. It's the one where everyone leaves with something that works — and a reason to keep building.*
