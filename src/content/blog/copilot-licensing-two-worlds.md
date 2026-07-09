---
title: "Copilot agents: when are you covered by the license, and when do credits kick in?"
description: "A one-page decision guide for enterprise teams — which agent scenarios fall under the Microsoft 365 Copilot license, and which ones burn Copilot Credits."
pubDate: 2026-07-09
author: "Paul"
tags: ["copilot", "licensing", "copilot-credits", "copilot-studio"]
---

Every agent conversation in an enterprise eventually arrives at the same question: **"Wait — is this included in our Copilot licenses, or does it cost extra?"**

I kept re-deriving the answer from the licensing guide, so I condensed it into a single decision picture. Here's the full map, then the reasoning behind it.

![Enterprise ways to build and consume Microsoft 365 Copilot & Copilot Studio agents — decision guide](/images/copilot-agent-licensing-decision-guide.png)

*Based on the Microsoft Copilot Studio Licensing Guide (April 2026 edition). Licensing changes frequently — always verify against the current guide and Microsoft Product Terms before making procurement decisions.*

## The two worlds

Everything splits into two buckets:

**World 1 — Included under the Microsoft 365 Copilot User SL** (fair usage applies). If your scenario lives entirely inside the Microsoft 365 Copilot experience, used by licensed employees, there is no separate agent charge.

**World 2 — Charged via Copilot Credits** (consumption-based licensing). The moment your agent steps outside that boundary — different surface, different audience, different identity — you're paying per consumption.

## World 1: what the Copilot license already covers

Your scenario stays in the "included" world when **all** of these are true:

1. **Build:** you're building agents and plugins that *extend Microsoft 365 Copilot*
2. **Consume:** employees use those agents *inside the Microsoft 365 experience* — Teams, SharePoint, the Microsoft 365 Copilot app
3. **Conditions:**
   - It's a business-to-employee / internal employee scenario
   - The user has a Microsoft 365 Copilot license
   - The user is authenticated
   - The agent operates using *that user's identity*
   - Usage stays within fair usage limits

Typical outcome: **no separate Copilot Credit charge** for these scenarios.

The condition that trips people up most is the identity one — the agent must act *as the authenticated, licensed user*. Break that link and you've left World 1.

## World 2: when Copilot Credits apply

Credits come into play when you **build and publish your own agents and agent flows anywhere** — internal web, external web, Teams, WhatsApp, Facebook — and the interactions are performed *by* Copilot Studio agents and their actions.

You are **not** covered by included Microsoft 365 rights when:

- The employee does **not** have a Microsoft 365 Copilot license
- It's a **standalone or internal web chatbot** outside the M365 experience
- It's an **external / customer / public-facing** agent
- The agent is **not operating as the authenticated licensed employee** (service accounts, shared identities)

### Ways to buy credits

| Option | Model | Notes |
|---|---|---|
| Pay-as-you-go | ~$0.01 per Copilot Credit | Meter runs with usage |
| Credit Pack | 25,000 credits/pack, monthly | Unused credits expire at term end |
| Copilot Credit P3 | Annual prepaid; 1 CCU = ~100 credits | Expires at term end |
| Microsoft Agent P3 | Annual prepaid; 1 ACU = ~100 credits | Expires at term end |

The prepaid options trade flexibility for predictability — the right choice depends on how confident you are in your usage forecast (I've written before about [modelling that forecast honestly](/blog/agent-licensing-cost-model/)).

## The quick decision guide

| # | Scenario | Answer |
|---|---|---|
| 1 | Licensed employee uses an agent in Teams / SharePoint / M365 Copilot | **M365 Copilot User SL** |
| 2 | Builder creates an agent/plugin to extend Microsoft 365 Copilot | **M365 Copilot User SL** |
| 3 | Internal users *without* M365 Copilot use the agent | **Copilot Credits** |
| 4 | Standalone internal web chatbot | **Copilot Credits** |
| 5 | External website / WhatsApp / Facebook / customer agent | **Copilot Credits** |
| 6 | Shared or service-account style access instead of the licensed user identity | **Treat as Copilot Credits** — not clearly covered by included rights |

Rows 3 and 6 are where budgets get surprised: the *agent* is the same, but the *audience or identity* changed — and that alone moves you from "included" to "metered."

## Footnotes worth knowing

- **Dataverse capacity:** Copilot Studio's default per-tenant capacity is Database 15 GB, File 20 GB, Log 2 GB
- **Managed Environments** have their own licensing details worth reviewing if you're operating at scale
- For final procurement decisions, consult the **Microsoft Product Terms** and your Microsoft account team — this article is a builder's field map, not licensing advice

## My takeaway

Before designing any agent, ask two questions first: **who will use it, and under whose identity does it run?** Those two answers alone place you in World 1 or World 2 — and they should be settled *before* the architecture, because they change the economics of everything downstream.
