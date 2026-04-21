---
name: browser-use
description: AI-powered browser automation with self-healing. Navigate websites, fill forms, extract data, take screenshots. More robust than Playwright — uses AI to understand page context, survives UI changes. Use for web research, NotebookLM interaction, or any browser task.
---

# Browser Use

AI-powered browser automation that understands web pages like a human. Unlike Playwright (which breaks when selectors change), Browser Use combines HTML parsing with visual understanding for self-healing automation.

## Installation

```bash
pip install "browser-use[cli]"
browser-use doctor    # Verify installation
```

IMPORTANT: You must install with the `[cli]` extra. Plain `pip install browser-use` is not enough.

## Core Workflow

1. **Navigate**: `browser-use open <url>` — opens page in headless browser
2. **Inspect**: `browser-use state` — returns clickable elements with indices
3. **Interact**: `browser-use click 5`, `browser-use input 3 "text"`
4. **Verify**: `browser-use state` or `browser-use screenshot`
5. **Repeat**: browser stays open between commands

## Using Existing Chrome (preserves logins/cookies)

```bash
browser-use connect    # Connects to running Chrome
```

This is essential for services requiring login (Google, NotebookLM, etc.). The agent inherits the user's authenticated session.

If connect fails, the user needs to enable remote debugging:
- Relaunch Chrome with `--remote-debugging-port=9222`
- Then retry `browser-use connect`

## Common Tasks

### Web Research
```bash
browser-use open "https://scholar.google.com"
browser-use state                           # Find search box index
browser-use input 3 "graphene biosensor"    # Type search
browser-use click 5                         # Click search button
browser-use state                           # Read results
```

### Screenshot for Documentation
```bash
browser-use open "https://example.com/figure"
browser-use screenshot output.png
```

### Extract Text
```bash
browser-use open "https://arxiv.org/abs/2401.12345"
browser-use state    # Returns page text and elements
```

## Troubleshooting

- Command fails? Run `browser-use close` first, then retry
- Need fresh session? `browser-use close && browser-use open <url>`
- Need user's cookies? `browser-use connect` before any other command

## When to Use vs. Alternatives

- **Simple URL fetch**: use curl or WebFetch instead
- **Known API available**: use the API directly (faster, cheaper)
- **Complex multi-step web interaction**: use browser-use
- **Login-protected sites**: use browser-use with connect

This skill can be improved by agents as they develop web automation workflows.
