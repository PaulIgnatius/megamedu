---
title: "Consuming the Microsoft 365 Copilot Retrieval API"
description: "Grounding your agents on tenant knowledge without building your own RAG pipeline — with working request examples."
pubDate: 2026-06-28
author: "Paul"
tags: ["retrieval-api", "rag", "graph"]
---

If you're building agents that need to answer from your organisation's own content, you have two options: build and maintain your own RAG pipeline, or let the platform do the retrieval for you. The **Retrieval API** (part of Microsoft Graph) is the second option — it returns relevant, permission-trimmed text chunks from SharePoint and Copilot connectors, ready to feed into your model.

## Why this matters

A homegrown RAG pipeline means you own chunking, embedding, indexing, security trimming, and freshness. The Retrieval API hands you extracts that already respect the caller's permissions. For enterprise scenarios, that last part — **security trimming by default** — is the headline feature.

## A basic retrieval request

The call is a single POST. Here's the shape of it:

```http
POST https://graph.microsoft.com/v1.0/copilot/retrieval
Content-Type: application/json

{
  "queryString": "visitor gate pass approval workflow",
  "dataSource": "sharePoint",
  "resourceMetadata": ["title", "author"],
  "maximumNumberOfResults": 5
}
```

## Calling it from TypeScript

```typescript
import { Client } from "@microsoft/microsoft-graph-client";

async function retrieve(client: Client, query: string) {
  const body = {
    queryString: query,
    dataSource: "sharePoint",
    maximumNumberOfResults: 5,
  };

  const response = await client
    .api("/copilot/retrieval")
    .version("v1.0")
    .post(body);

  // Each hit contains extracts you can pass straight to your model
  return response.retrievalHits.map((hit: any) => ({
    url: hit.webUrl,
    extracts: hit.extracts.map((e: any) => e.text),
  }));
}
```

## And from Python

```python
import requests

def retrieve(token: str, query: str) -> list[dict]:
    url = "https://graph.microsoft.com/v1.0/copilot/retrieval"
    payload = {
        "queryString": query,
        "dataSource": "sharePoint",
        "maximumNumberOfResults": 5,
    }
    headers = {"Authorization": f"Bearer {token}"}

    resp = requests.post(url, json=payload, headers=headers, timeout=30)
    resp.raise_for_status()

    return [
        {"url": hit["webUrl"], "extracts": [e["text"] for e in hit["extracts"]]}
        for hit in resp.json()["retrievalHits"]
    ]
```

## Practical notes from using it

- **Query quality matters.** This is a retrieval endpoint, not a chat endpoint — send a focused query string, not a whole conversation.
- **Permissions are the caller's.** Results are trimmed to what the signed-in user can access. Test with real user accounts, not just your admin account.
- **Pair it with your own prompt discipline.** Retrieved extracts are raw material; your agent's instructions decide how they're used and cited.

Verify current endpoint details, licensing prerequisites, and limits against the official Microsoft Graph documentation before shipping — this area of the platform moves fast.

*This is a sample starter article — replace the examples with your own tested code before publishing.*
