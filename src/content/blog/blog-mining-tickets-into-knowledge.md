---
title: "I Mined 146,000 IT Tickets to Build a Company Knowledge Base (With AI)"
description: "Every step of turning 146,000 raw support tickets into a clean, reviewed knowledge base — explained like you're new to tech, including everything that went wrong."
pubDate: 2026-07-22
author: "Paul"
authorRole: "Founder, Pauls Workshop"
tags: ["knowledge-base", "data-engineering", "ai-agents"]
---

*A build-in-public field note. No prior tech knowledge required — every term is explained the first time it appears. Grab a coffee; this is the full story, including everything that went wrong.*

> **Why we're doing this — the end goal in one paragraph:** we're building an **IT Customer Support Agent** — an AI assistant inside Microsoft 365 that employees can chat with to solve IT problems ("my Wi-Fi isn't working", "how do I get project access?") without waiting on a human engineer, and that raises a well-formed ticket when self-help isn't enough. An agent like that is only as good as the knowledge behind it — and we didn't have a knowledge base. This post is the story of building one **out of our own ticket history**: every step of turning 146,000 raw support tickets into clean, reviewed knowledge articles an agent can trust. The agent build itself is the sequel post; this is the foundation it stands on.

> **![Terminal output showing the final quality-gate report](/images/myTerminal.png)**

---

## The problem

Every company with an IT helpdesk has the same buried treasure: years of support tickets. A user says "my Wi-Fi isn't working," an engineer asks questions, fixes it, closes the ticket. Multiply by six years and you get — in our case — **146,206 tickets and 351,929 comments** describing real problems and real fixes.

The treasure is buried because nobody can read 146,000 tickets. New employees ask the same questions; engineers solve the same problems from memory. And the support agent we set out to build hit the classic wall on day one: *the agent is only as good as the knowledge you give it — and we had no knowledge base. Just tickets.*

So we built a pipeline that turns raw tickets into a clean, reviewed knowledge base. This post walks through every step, tagged with **which data role does this kind of work** — because if you're wondering what "data engineer" or "data scientist" actually means day-to-day, this project is a tour of all of them.

**The cast of roles you'll meet:**

| Tag | Role | One-line job description |
|---|---|---|
| DE | Data Engineer | Moves data between systems without losing or breaking it |
| DA/DS | Data Analyst / Scientist | Asks questions of data and lets evidence change the plan |
| ML | Machine Learning | Finds patterns in data no one labeled by hand |
| AI | AI Engineer | Puts language models to work — safely and verifiably |

---

## The 30-second overview

```
 SharePoint (where tickets live)
        │
   [1] Look at the data's shape        DE
        │
   [2] Export everything, raw          DE
        │
   [3] Load into a local database      DE
        │
   [4] Debug the hidden relationship   DA
        │
   [5] Clean text + remove names       DE + AI
        │
   [6] Group tickets by problem type   ML
        │
   [7] Read samples, fix the groups    DS
        │
   [8] Pick the best example tickets   DS
        │
   [9] AI writes draft KB articles     AI
        │
  [10] Catch leaked names (safety net) AI
        │
  Human engineers review → Knowledge Base → AI agent
```

Ten scripts (plus an eleventh that packages the reviewed articles for publishing). Two languages (PowerShell and Python). One laptop. Let's go.

---

## Step 0 — Decide the rules before touching anything

**Role: a bit of everyone (this is "engineering judgment")**

Before writing code, we fixed three rules that shaped everything:

1. **Never modify the source.** The ticket system is live — the IT team uses it daily. We only ever *copy* data out. All cleanup happens on the copy.
2. **Extract dumb, filter smart.** Export *everything*, raw, with no filtering — then do all the thinking locally where mistakes cost seconds instead of hours. (This rule saved us three times. You'll see.)
3. **No personal data leaves the laptop.** Tickets contain names, emails, phone numbers. Before any text goes to a cloud AI service, it gets scrubbed. Twice, it turned out.

> **New-to-tech takeaway:** the best data people spend surprising amounts of time on rules like these *before* coding. Code is cheap to fix; a leaked name or a corrupted source system is not.

---

## Step 1 — Reconnaissance: what does the data even look like? (DE)

You can't move data you don't understand. Our first script didn't extract anything — it just asked the ticket system to *describe itself*: what fields (columns) exist, what type is each one, and show me one sample row.

```powershell
Get-PnPField -List "Tickets" |
    Where-Object { -not $_.Hidden } |
    Select-Object InternalName, Title, TypeAsString
```

**Reading this line by line:** `Get-PnPField` asks SharePoint (Microsoft's document/list platform where our tickets live) for the field definitions of the list called "Tickets". The `Where-Object` part filters out hidden system fields. `Select-Object` picks just three properties to display: the internal name (what code must use), the display name (what humans see), and the type.

This mattered because internal names are often ugly and unguessable — a column displayed as "Issue Description" might internally be `Issue_x0020_Description`. Guess wrong and every later step breaks.

> **Jargon unlocked — API:** an *Application Programming Interface* is just "a way for programs to talk to each other." Instead of clicking through SharePoint's website, our script talks to SharePoint's API directly.

---

## Step 2 — Export everything, raw (DE)

Now the big copy: every ticket and every comment, out of SharePoint, into two plain files on disk. Three real-world problems had to be handled:

**Problem 1: You can't ask for 146,000 rows at once.** SharePoint refuses any query that would touch more than 5,000 items (the infamous *list view threshold* — a speed-protection rule). The workaround is **paging**: ask for 5,000, then the next 5,000, like reading a book chapter by chapter.

**Problem 2: 400,000 rows don't fit comfortably in memory.** So the script *streams* — each row is written to disk the moment it arrives, never holding more than a page in memory.

**Problem 3: What file format?** We used **JSONL** — one JSON object per line. (JSON is a universal text format for structured data — imagine a labeled box: `{"Title": "Wifi broken", "Priority": "P3"}`.) JSONL is beloved by data engineers because it streams beautifully and nothing chokes on it.

```powershell
Get-PnPListItem -List "Tickets" -PageSize 5000 | ForEach-Object {
    $writer.WriteLine(($_.FieldValues | ConvertTo-Json -Compress))
}
```

**Line by line:** fetch list items in pages of 5,000; for each item, convert its field values to a compact JSON string and write it as one line to the output file. Roughly an hour later: two files, ~500,000 lines, complete history.

Notice what we did **not** do: filter by date, status, or anything else. Rule 2 — extract dumb. Every later change of mind ("actually, let's use 24 months, not 12") became a one-line local query instead of another hour against SharePoint.

---

## Step 3 — Load into a real database (DE)

Two giant text files are an archive, not a workspace. We loaded them into **SQLite** — a complete database that lives in a single file on your laptop, no server needed. It's probably the most deployed software on Earth (it's in your phone right now).

Why a database? Because the killer question is *relational*: "give me this ticket **and all its comments, in order**." In text files, answering that means scanning 400,000 lines every time. In an indexed database: **milliseconds**.

```python
conn.executescript("""
    CREATE INDEX ix_comments_ticket ON comments(ticket_guid);

    CREATE VIEW threads AS
    SELECT t.title, t.status, c.body, c.created_utc
    FROM tickets t
    LEFT JOIN comments c ON c.ticket_guid = t.app_guid
    ORDER BY t.sp_id, c.created_utc;
""")
```

**Translated:** an *index* is like the index at the back of a book — instead of reading every page to find "Wi-Fi," you jump straight there. A *JOIN* stitches two tables together on a shared key (each comment carries its parent ticket's ID). A *view* is a saved question you can ask again and again.

One design choice paid off enormously: alongside the tidy columns, we kept **every row's complete original JSON** in a `raw` column. Insurance. Which brings us to…

---

## Step 4 — The case of the missing relationship (DA)

Here's the honest part they don't put in tutorials. When we first joined comments to tickets, the result was:

```
tickets.guid       links 0/351,929 comments
tickets.unique_id  links 0/351,929 comments
```

**Zero percent.** Not one comment matched one ticket. Yet the helpdesk app clearly *knew* which comments belonged where — so a link had to exist. We just couldn't see it. The hunt, hypothesis by hypothesis:

1. *A visible link column?* No — checked every field. Ruled out.
2. *A hidden link column?* Checked hidden fields of the "lookup" type. Nothing. Ruled out.
3. *Comments stored in folders named after tickets?* Pulled five real rows. Folder paths were flat. Ruled out.
4. *Comments matched by title?* The titles did repeat per ticket — but titles aren't unique (dozens of tickets share "Request: Software Access"). A trap. Ruled out.
5. Then — a look at the list's **Indexed Columns** settings page revealed a hidden field called `TicketGuid`. The app indexes it because "fetch this ticket's comments" is its hottest query. **An application's indexes are a map of its query patterns** — write that one down. Confirmed.

And the punchline: the ticket side of that relationship wasn't SharePoint's built-in ID at all — the app stamps **its own GUID** (a *Globally Unique Identifier*, a long random-looking string like `3b585b2e-d870-...`) on every ticket. That value was sitting, all along, in the `raw` column we'd kept "just in case." One mapping change later:

```
tickets.app_guid   links 351,926/351,929 comments   (100.0%)
```

No re-export. Rule 2 vindicated. Three orphan comments out of a third of a million — their parent tickets had been deleted. We noted them and moved on.

> **This is what data analysis actually is:** not dashboards — detective work. Form a hypothesis, test it against real rows, kill it, form the next one. The tools were trivial (five sample rows, one settings page). The method was everything.

---

## Step 5 — Clean the text, tag the speakers, remove the people (DE + AI)

Raw ticket text is a swamp: HTML tags, email signatures, "Thank you for contacting IT!" templates, quoted reply chains, and — critically — people's names everywhere.

This script rebuilt every ticket as a clean, readable **thread**:

```
TICKET: Incident: Guest wi-fi does not work
ISSUE:
[USER] I kindly ask you please check the guest wi-fi. It's not working.
[ENGINEER-PRIVATE] Tested on my phone — intermittent. Some phones show
"no internet connection".
[ENGINEER] We restarted the Wi-Fi access points — please retest.
```

Three jobs happened here:

**1. Text cleanup** with *regular expressions* (regex) — a mini-language for describing text patterns. One example, the pattern that finds email addresses:

```python
EMAIL_RE = re.compile(r"[\w.+-]+@[\w-]+\.[\w.-]+")
text = EMAIL_RE.sub("<EMAIL>", text)
```

Read it as: "some word-ish characters, then an @, then more word-ish characters, then a dot, then more" — i.e., the *shape* of an email. `.sub()` replaces every match with the token `<EMAIL>`.

**2. Speaker tagging.** Every comment got labeled `[USER]` or `[ENGINEER]` by comparing who wrote it against the ticket's requester and the IT team roster — which we didn't type by hand: we *derived* it from the data (everyone who ever appeared in the "Assigned To" field = 36 identities).

**3. A quality gate.** Here's a dirty secret of every helpdesk: many tickets get solved over the phone and closed with a stock phrase — no written fix. Our IT lead estimated "about half." We built a scoring pass that graded every thread: **FULL** (a real documented fix), **PARTIAL** (fixed, but the story is thin), **NONE** (nothing usable). The verdict on 51,004 tickets:

```
FULL      21,786  (37.7%)
PARTIAL   27,689  (47.9%)
NONE       1,529  ( 2.6%)
```

PARTIAL + NONE ≈ 50.5%. The human estimate was 50%. **Practitioner intuition, confirmed by data to within half a percent** — my favorite result of the whole project.

*(A bug confession: our first version reported 78% FULL — suspiciously great. The cause: the closing template itself is ~300 characters long, and our "did the engineer write anything substantial?" check was counting the template as substance. We stripped boilerplate before measuring, and truth emerged. Metrics that flatter you deserve extra suspicion.)*

---

## Step 6 — Let the machine find the problem types (ML)

51,000 cleaned threads. Nobody labeled them "this is a printer problem" / "this is a VPN problem." How do you group them without reading them?

This is **unsupervised machine learning** — finding structure with no labels. Two techniques, both older and simpler than the AI hype suggests:

**TF-IDF** (*term frequency–inverse document frequency*) turns each ticket into a numeric profile of its *distinctive* words. The intuition: a word scores high if it's frequent in **this** ticket but rare across **all** tickets. So "revit" (a design software) scores high for the tickets that mention it; "please" appears everywhere and scores ~zero.

**KMeans clustering** then groups tickets whose profiles look alike. We asked for 40 groups:

```python
vec = TfidfVectorizer(max_features=20000, ngram_range=(1, 2), stop_words=stop)
X = vec.fit_transform(docs)          # 51,004 tickets -> number profiles
labels = KMeans(n_clusters=40).fit_predict(X)   # -> 40 groups
```

Out came clusters for printers, VPN, Teams, monitors, software licences… and two surprises worth their own paragraphs:

**Surprise 1 — the biggest "gap" wasn't a gap.** One cluster held **12,000 tickets, only 3.6% documented**. Alarming — until we read five of them. They were *auto-generated*: an access-request form that files a ticket automatically after approval, closed with an automatic confirmation. Not missing knowledge — **automation exhaust**. That cluster needs a workflow route in the agent, not a knowledge article. About a quarter of all our tickets turned out to be this.

**Surprise 2 — garbage features, garbage clusters.** Our first sub-clustering attempt produced a group whose most distinctive words were… **"hi," "dear," "team."** The algorithm had clustered tickets by *how politely they open*. The fix wasn't a smarter algorithm — it was adding greeting words to the ignore-list (*stopwords*). When a cluster's top terms could open any email, your features are broken, not your data.

> **Jargon unlocked — features:** the numeric representation you feed a machine-learning algorithm. Most ML failures are feature failures wearing an algorithm costume.

---

## Steps 7 & 8 — Humans read; then pick the best examples (DS)

An algorithm guarantees *similarity*, never *usefulness*. So we built a small inspection tool: point it at any cluster and it prints the quality mix, the most common titles and closing lines, and a few full threads to read. Ten minutes of reading resolved what no statistic could — which clusters deserve articles, which are workflows, which are noise.

Then a selector picked, for each of 19 chosen topics, the **8 best example threads**: documented fixes first, scored by richness and recency, with a diversity rule — *max two threads per person* — so one colleague's cursed laptop can't become company-wide doctrine.

And here the most instructive failure of the project happened. Our "Wi-Fi problems" topic used keyword matching, and the first selection confidently returned: a SIM-card procurement saga (the SIM plan brochure said "Wifi Data Unlimited"), a printer install (the engineer had pasted a vendor Wi-Fi guide), and an office-construction thread. **Every console metric was green. Only reading the file caught it.** Two iterations later, the rule that fixed it: match keywords only against *the title and the user's own words* — never against the whole conversation, because conversations mention everything.

> **The lesson I'd tattoo on every dashboard:** aggregate metrics can all pass while the content is wrong. Spot-read the actual artifact. It costs minutes and catches what numbers can't.

---

## Step 9 — The AI writes the first drafts (AI)

Only now — nine steps in — does the language model appear. This ordering *is* the point: an LLM (*Large Language Model* — the technology behind ChatGPT, Claude, etc.) can't extract a fix from threads that never recorded one, and it will happily synthesize garbage from contaminated input. Steps 1–8 existed so that step 9 could be trusted.

For each topic, we send the 8 curated threads plus a strict **prompt** (the instructions given to the model) and get back one draft knowledge-base article. The prompt's hard rules do the heavy lifting:

- **Grounding:** every claim must come from the threads. "Not established in source threads" is a *valid and welcome* answer. Inventing steps is forbidden.
- **Audience split** (a rule a colleague's sharp question added): steps an *end user* can safely do go in "Self-help"; things only IT does — restarting access points, calling the internet provider — go in a separately labeled internal section. Why? The future agent talks to end users, and end users contact IT, never the ISP. Without the split, the agent would cheerfully tell an intern to ring the telecom company.
- **Honesty section:** each draft ends with *Evidence notes* for the human reviewer — how many threads support the fix, where they disagreed, what's missing.

The result for the Wi-Fi topic looked like a real article: user-phrased symptoms ("wifi shows available and on maximum but not working"), self-help steps, questions to ask before raising a ticket, internal steps for IT — and an honest note that the engineers' internal diagnostics were probably tribal knowledge worth adding by hand.

Total cost of drafting all 19 topics with a cloud model: **pocket change**. The expensive ingredient was never the AI — it was the clean input.

---

## Step 10 — The safety net for names (AI)

One last honest failure. Our regex-based name removal (Step 5) went through *four rounds* of patching — greetings, honorifics like "Eng. Someone", form fields like "Name: …" — and then a thread turned up containing a **staffing table with ~30 bare names in plain table cells**. No pattern can catch a bare name; nothing marks it as a name except *meaning*.

That's what **NER** (*Named Entity Recognition*) is for — a model trained to recognize that a string of characters *is* a person's name, regardless of surroundings. We added Microsoft's open-source **Presidio** as a final gate: every file gets an NER pass before it leaves the laptop, with a protected list so product names (which NER loves to flag as people) survive.

The layered lesson: **regex for the patterns you can describe, NER for the ones you can't, and a human spot-check on top.** Privacy is a stack, not a step.

---

## The toolbox — every library we used, and why

A detail worth noticing before the recap: **most of this pipeline runs on things that come free with Python** — no installs. We only reached for external packages at three specific moments. Here's the full inventory:

**PowerShell side (the extraction):**

| Tool | Install? | What it did for us |
|---|---|---|
| PnP.PowerShell | `Install-Module PnP.PowerShell` | The community-standard toolkit for talking to SharePoint from scripts — paging through lists, reading field definitions |

**Python — built-in (no install, part of the language):**

| Module | What it did for us |
|---|---|
| `json` | Read/write JSON — parsing every exported ticket line |
| `sqlite3` | The entire database layer — tables, indexes, joins, views |
| `re` | Regular expressions — all the pattern cleanup and the first PII pass |
| `csv` | Reading our topics configuration file |
| `html` | Decoding HTML entities like `&nbsp;` hiding in ticket text |
| `pathlib` / `os` | Finding and writing files |
| `argparse` | Giving scripts command-line options like `--db` and `--months` |
| `datetime` | Date math for the scope window and recency scoring |
| `collections.Counter` | "What are the 10 most common titles in this cluster?" in one line |

**Python — installed (the three deliberate reaches):**

| Package | Install | Why we reached for it |
|---|---|---|
| `scikit-learn` | `pip install scikit-learn` | The machine-learning step: TF-IDF vectorization + KMeans clustering (Step 6) |
| `presidio-analyzer` + `presidio-anonymizer` (+ a spaCy language model) | `pip install presidio-analyzer presidio-anonymizer`<br>`python -m spacy download en_core_web_lg` | NER-based name detection — the privacy safety net regex couldn't provide (Step 10) |
| `openai` | `pip install openai` | The official SDK for calling the Azure-hosted language model (Step 9) |

> **The pattern to steal:** standard library by default, install only when a capability genuinely doesn't exist there. Every dependency you add is something that can break, conflict, or need updating — and for data work, `json + sqlite3 + re` covers a shocking share of real jobs. Our 500,000-row loader is pure standard library and runs in minutes.

## What all this actually taught (the recap table)

| # | Step | Role | The transferable lesson |
|---|---|---|---|
| 1 | Schema recon | DE | Understand data before moving it |
| 2 | Raw export | DE | Extract dumb, filter smart, never twice |
| 3 | Local database | DE | Move questions to where they're cheap |
| 4 | Join detective work | DA | Hypothesis → 5 real rows → verdict; indexes reveal app behavior |
| 5 | Cleanup + quality gate | DE + AI | Boilerplate lies to metrics; validate intuition with data |
| 6 | Clustering | ML | Garbage features → garbage clusters; big "gaps" may be automation |
| 7 | Human inspection | DS | Algorithms find similarity; humans assign meaning |
| 8 | Example selection | DS | Green metrics ≠ correct content — read the artifact |
| 9 | LLM synthesis | AI | The model is the last 20%; grounding rules beat model shopping |
| 10 | NER privacy gate | AI | Privacy is layered: regex + NER + human eyes |

And one meta-lesson that spans them all: at three separate points, a **human question redirected the machine** — "half our tickets have no written fix," "titles start with Incident: or Request: and that means something," and "our users must never be told to call the ISP." None of those could have come from the data alone. If you're worried AI work leaves no room for domain knowledge: it's the opposite. Domain knowledge is the steering wheel.

## How it ended: the results

The batch run produced **19 draft articles**. We reviewed every one against its source threads. Final score: **10 straight accepts, 5 accepted with renaming, 4 kept but downgraded to low confidence — and 0 rejected.** Total AI cost for all drafting: still pocket change. The review itself surfaced three findings worth more than the articles:

**1. The dominant failure was labels, not content.** Five articles were *good articles about the wrong topic* — e.g., the file named "Autodesk licensing" contained an excellent general software-procurement workflow built from Visum, ArcGIS, and Miro tickets, with zero Autodesk in sight. The cause traced back to shortcuts we took when mapping topics to clusters. The fix cost nothing (rename the file); the lesson didn't: **always check that content matches its label before checking whether content is good** — mislabeling hides in plain sight because the content looks fine.

**2. Our selector had a "saga bias."** We scored candidate threads by richness — length, comment count — and long dramatic threads won. But epics are, almost by definition, *unusual* cases: three of our four thinnest articles trace to a single spectacular thread beating out dozens of short boring ones describing the everyday version of the problem. Next iteration scores threads on *representativeness within their group*, not drama. **Optimize a proxy and you get the proxy** — the oldest lesson in data science, learned again.

**3. The model caught a policy contradiction humans had lived with for years.** In the ex-employee-mailbox topic, one thread said "we typically don't allow full mailbox access" while others showed it being granted. The AI didn't resolve it — correctly — it *flagged it for the reviewer*, and the question is now with IT leadership. Reading eight threads side by side surfaces inconsistencies that no one notices living them one ticket at a time. An unexpected bonus use of synthesis: **contradiction detection in your own operations.**

One more number for the skeptics: the single best article contains a complete, working Power Query solution to a gnarly SharePoint limitation — written by one engineer, in one ticket, two years ago, findable by no one. It's now a reviewed article anyone (and soon, an AI agent) can retrieve. That one recovery arguably justifies the pipeline by itself.

---

The knowledge base now goes to the engineers for review (each draft article is a pull request — a proposed change a human approves), and then it becomes the grounding for a Copilot support agent inside Microsoft 365 — built, deliberately, within the zero-extra-cost boundary of our existing licences. That's the next post.

*Questions, corrections, or war stories from your own ticket mines — I'd genuinely love to hear them.*

---

*Numbers lightly rounded and all examples anonymized. The failures are reported exactly as they happened, because those were the useful parts.*
