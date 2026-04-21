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

Deep Research uses the **Interactions API**, not `generate_content`.

```python
import google.genai as genai

client = genai.Client()  # Uses GOOGLE_AI_API_KEY env var

# Start research with a query
interaction = client.aio.interactions.create(
    agent_id="deep-research-pro-preview",
    config={"background": True, "store": True},
    user_message="Research the latest advances in perovskite solar cells"
)

# Poll for completion
import time
while True:
    result = client.aio.interactions.get(interaction.id)
    if result.status == "COMPLETED":
        break
    time.sleep(30)

# Extract the report
report = result.messages[-1].content
```

## With Document Upload

PDFs and other files can be attached for context-aware research:

```python
# Upload a file first
file = client.files.upload("paper_draft.pdf")

# Then reference it in the research query
interaction = client.aio.interactions.create(
    agent_id="deep-research-pro-preview",
    config={"background": True, "store": True},
    user_message="Based on the attached paper, find related work and competing approaches",
    attachments=[file]
)
```

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

This skill can be improved by agents as they develop research workflows.
