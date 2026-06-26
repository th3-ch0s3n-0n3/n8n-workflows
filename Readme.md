# Daily Security News — n8n Workflow

An automated n8n workflow that aggregates daily security news from curated RSS feeds, filters it using AI-powered CTI analysis, and delivers relevant findings to a Discord channel.

---

## Overview

This workflow runs on demand (via chat trigger) and performs the following:

1. Posts a "Here are your security news for today" header message to Discord
2. Fetches the latest posts from 4 security-focused RSS feeds (capped at 5 items each)
3. Merges and aggregates the feed data into a single batch
4. Sends the batch to **Gemini 2.5 Flash Lite** with a detailed Senior CTI Analyst prompt
5. Posts the AI-filtered, formatted findings back to Discord

---

## Workflow Diagram

```
[Chat Trigger]
      │
      ▼
[Send Header Message → Discord]
      │
      ├──► [Bleeping Computer RSS] → [Limit: 5]──┐
      ├──► [Aikido RSS]            → [Limit: 5]──┤
      ├──► [Endor Labs RSS]        → [Limit: 5]──┤→ [Merge] → [Aggregate] → [Gemini 2.5 Flash Lite] → [Send Results → Discord]
      └──► [Stepsecurity RSS]      → [Limit: 5]──┘
```

---

## RSS Feed Sources

| Node | Feed URL |
|---|---|
| Bleeping Computer | `https://www.bleepingcomputer.com/feed/` |
| Aikido | `https://www.aikido.dev/blog/rss.xml` |
| Endor Labs | `https://www.endorlabs.com/blog/rss.xml` |
| Stepsecurity | `https://www.stepsecurity.io/blog/rss.xml` |

Each feed is limited to the 5 most recent items before merging.

---

## AI Filtering Logic

The aggregated batch is sent to **Gemini 2.5 Flash Lite** with a role-prompted CTI analyst system prompt. The model is instructed to use **only** the provided news batch (no external knowledge) and filter strictly by the following criteria.

### Focus Areas (report only these)

- Supply chain attacks: malicious or typosquatted packages on npm, PyPI, RubyGems, crates.io, Maven, NuGet
- Malicious browser extensions targeting developers
- IDE or coding agent attacks (VS Code, JetBrains, Cursor, Copilot, Claude, etc.)
- Critical CVEs (CVSS ≥ 9.0) in widely-used developer tools or libraries
- Credential theft or backdoors via CI/CD tools, GitHub Actions, or build pipelines

### Strict Ignore List

- Opinion pieces, editorials, award announcements
- Government breaches or geopolitical events
- Product launches, feature updates, or pricing changes
- Generic phishing/social engineering without a technical package/tool vector
- Company data breaches (unless caused by a supply chain or qualifying CVE)
- CVEs in server-side templating or serialization libraries without confirmed active exploitation
- CVEs where the affected library is not a direct developer dependency

### Output Format (per matching item)

```
🚨 TITLE: [Exact title from feed]
🔗 Link: [Exact link from feed]
📅 Date: [Exact pubDate from feed]
```

If no items match the criteria, the model outputs `SKIP` and nothing is posted.

---

## Discord Output

Posts go to the **#general** channel of the configured Discord server. Two messages are sent per run:

1. A static header: *"Here are your security news for today"*
2. The AI-generated findings (or nothing if the model returns `SKIP`)

---

## Prerequisites

| Requirement | Details |
|---|---|
| n8n instance | Self-hosted or n8n Cloud |
| Discord Bot | Bot must have `Send Messages` permission in the target channel |
| Google Gemini API key | Used for the `googlePalmApi` credential (`models/gemini-2.5-flash-lite`) |

---

## Setup

1. Import `Daily_Sec_News.json` into your n8n instance via **Workflows → Import from file**.
2. Configure credentials:
   - **Discord Bot account** — add your bot token under n8n credentials
   - **Google Gemini (PaLM) API account** — add your Gemini API key
3. Update the Discord `guildId` and `channelId` values in both Discord nodes to match your server and channel.
4. Activate the workflow or trigger it manually via the chat trigger.

---

## Customization

- **Add more feeds** — duplicate any RSS node, add a matching Limit node, and wire both into the Merge node (increase `numberInputs` accordingly).
- **Change the item limit** — adjust the `maxItems` value in any Limit node (currently `5` per feed).
- **Swap the model** — replace the Gemini node with any other LangChain-compatible model node in n8n.
- **Change the output channel** — update `guildId` and `channelId` in the Discord nodes.
- **Adjust focus areas** — edit the system prompt in the "Message a model" node to narrow or expand CTI scope.

---

## Notes

- The workflow is set to **inactive** by default; enable it or use a Schedule Trigger to run it automatically each morning.
- Output is capped at **1900 words** by the prompt to stay within Discord's message character limit.
- Quality over quantity is enforced — the model is instructed to prefer 2 strong hits over 5 weak ones.
