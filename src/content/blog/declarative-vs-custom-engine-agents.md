---
title: "Declarative vs custom engine agents: choosing your build path"
description: "The first architecture decision in every Copilot agent project — and the questions that actually decide it."
pubDate: 2026-07-06
author: "Paul"
tags: ["copilot", "agents", "architecture"]
---

Every Copilot agent project starts with the same fork in the road: do you build a **declarative agent** that runs on Microsoft 365 Copilot's orchestrator, or a **custom engine agent** where you own the orchestration, the model choice, and the runtime?

This article is the decision framework I use before writing a single line of code.

## The short version

| Question | Declarative | Custom engine |
|---|---|---|
| Who runs the orchestration? | Microsoft 365 Copilot | You |
| Model choice | Copilot's models | Any model you can host or call |
| Hosting | None needed | Your infrastructure (e.g. Azure) |
| Typical effort | Days | Weeks |
| Fine-grained control | Limited | Full |

If your agent is fundamentally "Copilot, but scoped and specialised" — grounded on specific knowledge, with defined instructions and a few actions — declarative is almost always the right first move.

## What a declarative agent actually is

A declarative agent is defined by a manifest: instructions, knowledge sources, and actions. No servers, no orchestration code. A minimal example:

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/copilot/declarative-agent/v1.4/schema.json",
  "version": "v1.4",
  "name": "Gate Pass Assistant",
  "description": "Answers visitor management policy questions and drafts gate pass requests.",
  "instructions": "You help facility staff manage visitor gate passes. Always confirm visitor identity requirements before drafting a pass.",
  "capabilities": [
    { "name": "OneDriveAndSharePoint", "items_by_url": [{ "url": "https://contoso.sharepoint.com/sites/Facilities" }] }
  ]
}
```

That manifest, plus the app package, is a working agent. The orchestration, safety layer, and model are Copilot's problem — which is exactly why the entry cost is so low.

## When custom engine is worth the effort

Choose a custom engine agent when at least one of these is true:

1. **You need a specific model** — a fine-tuned model, an open-weights model, or a specific frontier model version.
2. **You need multi-step orchestration you control** — long-running workflows, complex tool chains, or agent-to-agent handoffs.
3. **Your users aren't (all) in Microsoft 365 Copilot** — you want the same agent in Teams, a web app, and elsewhere.

The trade is real: you now own hosting, scaling, safety, and cost management. Budget for that before choosing this path.

## My rule of thumb

> Start declarative. Move to custom engine only when you hit a wall the manifest can't express — and write down which wall it was, because that's your business case.

In the next article, we'll look at what each path costs to run — because the licensing model is where many agent projects quietly succeed or fail.

*This is a sample starter article — edit or replace it with your own voice and latest platform details before publishing.*
