---
name: gemini-deep-research
description: Run comprehensive multi-source research via the Gemini Deep Research API. Analyzes 50-100+ web sources and uploaded documents (PDFs, CSVs) to produce cited reports. IMPORTANT — The GOOGLE_AI_API_KEY is exclusively for Deep Research. Do not use it for regular Gemini chat or any other Gemini API calls.
---

# Gemini Deep Research

Run comprehensive, multi-source research using the Gemini Deep Research API.

## CRITICAL RULE

The `GOOGLE_AI_API_KEY` environment variable is **exclusively for Deep Research**.

- DO use it for: `deep-research-pro-preview` via the Interactions API
- DO NOT use it for: regular Gemini chat, embeddings, or any other Gemini model
- Violation of this rule wastes budget and is not authorized by the board

## What Deep Research Does

Gemini Deep Research autonomously plans and executes multi-step research tasks. It:
- Analyzes 50-100+ web sources
- Accepts uploaded documents (PDFs, CSVs, docs) as additional context
- Produces comprehensive, fully-cited reports
- Takes several minutes to complete (asynchronous)

## API Access

Deep Research uses the **Interactions API**, not `generate_content`. The shape below is verified end-to-end against `google-genai` ≥ 1.65 (May 2026 — earlier examples in this skill were wrong, see "Gotchas" below).

```python
import os, sys, time
from google import genai

# 1) Build the client. The SDK's default env vars are GOOGLE_API_KEY / GEMINI_API_KEY,
#    NOT GOOGLE_AI_API_KEY — pass the key explicitly to avoid the SDK falling back
#    to a different Google API key (often a placeholder) that doesn't have Deep
#    Research access.
api_key = os.environ["GOOGLE_AI_API_KEY"]
assert api_key.startswith("AIza"), "GOOGLE_AI_API_KEY missing or wrong format"
client = genai.Client(api_key=api_key)

# 2) Submit the research interaction.
#    - agent (NOT agent_id): the dated model name "deep-research-pro-preview-12-2025"
#    - input (NOT user_message): a string OR an iterable of ContentParam dicts
#    - background, store, etc. are top-level kwargs (NOT config={...})
interaction = client.interactions.create(
    agent="deep-research-pro-preview-12-2025",
    background=True,
    store=True,
    input="Research the latest advances in perovskite solar cells",
)
print(f"Interaction id: {interaction.id}", flush=True)

# 3) Poll for completion.
#    - Status strings are LOWERCASE: "in_progress", "completed", "failed", "cancelled"
#    - Compare case-insensitively (.lower() or .upper()) — uppercase comparison silently
#      misses the completed state and the script polls until its own timeout.
#    - Use flush=True on every print or run python with python3 -u when output is
#      redirected to a file — Python uses block buffering for non-tty stdout, so
#      print() can sit in a 4 KB buffer for many minutes before reaching disk.
deadline = time.time() + 30 * 60
while True:
    if time.time() > deadline:
        sys.exit(f"Timed out waiting for {interaction.id}")
    result = client.interactions.get(interaction.id)
    status = (result.status or "").lower()
    print(f"  status={status}", flush=True)
    if status == "completed":
        break
    if status in ("failed", "cancelled"):
        sys.exit(f"Interaction {interaction.id} ended with status {status}")
    time.sleep(30)

# 4) Extract the assistant's report.
#    - The result lives in `result.outputs` — an Iterable[ContentParam] (NOT
#      `result.messages` — that field doesn't exist on the Interaction object).
#    - Each output is typically a TextContent with a `.text` attribute. There may
#      be additional output types (e.g. citation/document content) — filter on
#      whether `.text` is set.
report = "\n\n".join(o.text for o in result.outputs if getattr(o, "text", None))
print(f"Report length: {len(report)} chars")
```

## With document upload (PDFs, CSVs)

```python
# 1) Upload via the Files API. `display_name` is optional but useful for tracing.
manuscript = client.files.upload(
    file="manuscript.pdf",
    config={"display_name": "manuscript.pdf"},
)
figures = client.files.upload(
    file="figures_combined.pdf",
    config={"display_name": "figures_combined.pdf"},
)

# 2) Reference the uploaded files inside the input list using the document
#    ContentParam shape: {"type": "document", "mime_type": ..., "uri": file.uri}
#    NOTE: `uri` (NOT `file_uri` or `file_data`) — those keys are from older SDK
#    examples and will be ignored.
interaction = client.interactions.create(
    agent="deep-research-pro-preview-12-2025",
    background=True,
    store=True,
    input=[
        {"type": "text", "text": "Critically assess the attached paper against state-of-the-art literature 2024–2026 …"},
        {"type": "document", "mime_type": "application/pdf", "uri": manuscript.uri},
        {"type": "document", "mime_type": "application/pdf", "uri": figures.uri},
    ],
)
```

## Gotchas (verified bugs from prior runs — don't repeat)

| Wrong (older docs / earlier SDK) | Right (≥ 1.65) |
|---|---|
| `client = genai.Client()` | `client = genai.Client(api_key=os.environ["GOOGLE_AI_API_KEY"])` — explicit, otherwise SDK picks up `GOOGLE_API_KEY` which may be a placeholder |
| `client.aio.interactions.create(...)` | `client.interactions.create(...)` (the sync .create works fine; .aio is for async/await contexts) |
| `agent_id="deep-research-pro-preview"` | `agent="deep-research-pro-preview-12-2025"` — note key name AND the date suffix |
| `config={"background": True, "store": True}` | `background=True, store=True` as top-level kwargs |
| `user_message="..."` | `input="..."` (or `input=[content_blocks]`) |
| `attachments=[file]` | embed in `input` as `{"type": "document", "uri": file.uri}` |
| `if status == "COMPLETED":` | `if (status or "").lower() == "completed":` — API returns lowercase |
| `result.messages[-1].content` | `result.outputs` — list of ContentParam, get `.text` on each |
| `print(...)` redirected to file | `print(..., flush=True)` or run with `python3 -u` — block-buffered stdout otherwise hides progress for many minutes |

## Costs

- Standard research: ~$2-3 per task
- Complex research: ~$3-5 per task
- Budget is monitored — use wisely

## When to Use

- Literature review for a research topic
- Finding related work and state-of-the-art
- Market or technology landscape analysis
- Verifying claims against multiple sources
- Deep-dive into a specific methodology

## When NOT to Use

- Simple factual questions (use Chrome instead)
- Summarizing a single document (use NotebookLM instead)
- Anything that doesn't require multi-source synthesis

## After the report: ALWAYS hand off cited PDFs to the user → NotebookLM

The Deep Research report is only the first half of the workflow. The second half is making the cited papers actually accessible to the team for follow-up questions. **You MUST do this step at the end of every Deep Research run** — otherwise the user has prose recommendations they cannot verify or build on.

The report contains two kinds of cited material:

1. **Vertex-Search redirect URLs** in the final `**Sources:**` section, e.g.
   `1. [acs.org](https://vertexaisearch.cloud.google.com/grounding-api-redirect/AUZIYQ…)`.
   These URLs redirect to the actual paper landing page when opened in a browser.
2. **Inline `[cite: N]` markers** in the body, which point at the same numbered Sources list.

### What you do at the end of every run

1. **Parse the report's Sources section** to get the (number, redirect-URL) pairs.
2. **Identify the named papers** in the body where they're discussed — typically you'll see them as italicized titles followed by year and `[cite: N]`. Match each named paper to its `[cite: N]` URL.
3. **Save the report itself** to `cleanroom-drive:CBZpaper/output/drafts/deep_research_<topic>__<YYYY-MM-DD>.md` (this is the agent's deliverable).
4. **In your reply to the issue, post a structured handoff table.** Required columns: `cite N`, `title (truncated)`, `journal/year`, `Vertex URL`. Mark each row with its role (e.g. "kritisch", "nice-to-have") so the user knows priority.
5. **End with this exact ask** so the user knows what's next:

   > **Bitte folgende PDFs runterladen und in NotebookLM ergänzen:** download from the URLs above, save as `<FirstAuthor>_<Year>.pdf`, push to `cleanroom-drive:CBZpaper/literature/`, then add to the `CBZ Paper` notebook in NotebookLM (notebook id is in `projects/{project}/status.json` → `notebooklm_notebook_id`).

6. **Do NOT attempt to download the PDFs yourself.** Most journal sites are paywalled, the Vertex redirects expire, and your agent runtime has no library credentials. The user has institutional access and can download them on their Mac.
7. After the user confirms the PDFs are in NotebookLM, follow-up questions about specific cited claims should be sent to NotebookLM via the `notebooklm` skill — that's where the PDFs live, not in Deep Research's transient context.

### Filename convention for downloaded PDFs

The team uses `<FirstAuthor>_<Year>.pdf` (Cleanroom Limited convention from
`cleanroom-drive:CBZpaper/literature/`). Examples: `Hai_2018.pdf`, `Kaiser_2021.pdf`.
For author/year collisions add `b`/`c` suffix: `Yang_2024b.pdf` if `Yang_2024.pdf`
already exists in the library. Always tell the user the exact filename you expect
so they can save with that name.

### Why this matters

Without the handoff:
- The Deep Research report cites papers the user cannot read, only paraphrase.
- NotebookLM can't be queried about the cited papers because they're not uploaded.
- Agents in subsequent issues have no way to verify Deep Research's claims.

With the handoff, the cited literature becomes a permanent team resource that
every agent (via `notebooklm`) and every reviewer can re-use — turning a $2–5
one-shot research run into reusable institutional knowledge.

This skill can be improved by agents as they develop research workflows.
