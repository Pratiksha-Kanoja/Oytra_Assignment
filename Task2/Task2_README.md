# Task 2 — n8n API Integration Workflow: Daily GitHub Trending Digest

## APIs Used

### 1. GitHub REST API — Search Repositories
- **Endpoint:** `GET https://api.github.com/search/repositories`
- **Why:** GitHub's search API is free (no API key required for unauthenticated requests at 10 req/min), well-documented, and returns rich structured data. It lets us query repositories created in the last 24 hours, sorted by star count, giving us a "trending" feed.

### 2. GitHub REST API — Get Repository README
- **Endpoint:** `GET https://api.github.com/repos/{owner}/{repo}/readme`
- **Why:** This is the second API call used to *enrich* each top repository with a snippet of its README content. This satisfies the "second HTTP request" requirement by fetching deeper detail from a second endpoint of the same API.

---

## Workflow Overview

```
Schedule Trigger (every 1h)
    │
    ▼
Fetch Trending Repos ──(continueOnFail)──▶ Transform Top 5
                                              │
                                              ▼
                                     Enrich – Fetch README ──(continueOnFail)
                                              │
                                              ▼
                                     Merge README Data
                                              │
                                              ▼
                                      IF stars > 1000
                                       /           \
                                      /             \
                            TRUE ▼              FALSE ▼
                    Format High-Star       Format Regular
                       Digest                 Digest
                         │                      │
                         ▼                      ▼
                  Discord – Hot Repos   Discord – Regular Repos


Error Trigger ──▶ Log Error ──▶ Discord – Error Alert
```

---

## What the Transformation Does

The **Transform Top 5** Code node receives the raw GitHub search response (which can contain hundreds of fields per repository) and:

1. **Validates** that the API response is not an error.
2. **Slices** the results to keep only the **top 5** repositories by star count.
3. **Extracts** only the relevant fields: `rank`, `name`, `description`, `stars`, `forks`, `language`, `url`, `owner`, `repo_name`, and `created_at`.
4. **Returns** them as separate n8n items so downstream nodes process each repo individually.

The **Merge README Data** node then decodes the base64-encoded README content and attaches a 200-character snippet to each repo item.

---

## Conditional Branch

The **IF stars > 1000** node splits the flow:

| Condition | Path | Behavior |
|---|---|---|
| `stars > 1000` | **TRUE** → Format High-Star Digest | Sends a 🔥 "Hot Repos" message to Discord with full details including README snippets |
| `stars ≤ 1000` | **FALSE** → Format Regular Digest | Sends a 📊 "Trending Repos" summary to Discord without README snippets |

This ensures that exceptionally viral repos get highlighted differently in the team's notification channel.

---

## Error Handling

The workflow uses a **defense-in-depth** error strategy:

### Node-Level: `continueOnFail`
Both HTTP Request nodes (`Fetch Trending Repos` and `Enrich – Fetch README`) have `continueOnFail: true` enabled. This means:
- If GitHub's API returns a 403 (rate limit) or 5xx error, the workflow does **not** crash.
- The error response is passed downstream as regular data.
- The Transform node checks for error indicators and produces a safe fallback message.

### Workflow-Level: Error Trigger
A separate **Error Trigger** node catches any *unhandled* exceptions that propagate despite node-level handling:
1. **Error Trigger** activates when any node fails fatally.
2. **Log Error** (Code node) extracts the error message, the failing node name, and the execution ID into a structured log object.
3. **Discord – Error Alert** sends a `⚠️ WORKFLOW ERROR` notification to the same Discord channel so the team is immediately aware.

This two-layer approach ensures the workflow **never crashes silently** — either the error is gracefully handled inline, or it's caught, logged, and reported.

---

## Credentials

The Discord webhook URL is stored in n8n's **Credentials store** as a custom HTTP Auth credential (`Discord Webhook`). It is referenced by the Discord output nodes via `$credentials.discordWebhookUrl`. No secrets are hard-coded in the workflow JSON.

**To set up:**
1. In n8n, go to **Credentials → Add Credential → Header Auth** (or Custom Auth).
2. Name it `Discord Webhook`.
3. Add a field `discordWebhookUrl` with your Discord channel's webhook URL.
4. Save. The workflow nodes will automatically pick it up.

---

## How to Import and Run

1. Open your n8n instance.
2. Go to **Workflows → Import from File** and select `Task2_Workflow_PratikshaKanoja.json`.
3. Set up the Discord Webhook credential (see above).
4. Click **Execute Workflow** to test, or toggle it **Active** to run on the hourly schedule.
