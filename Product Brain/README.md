# Product BrAIn — Design Document

**Date:** 2026-03-20
**Status:** Approved

## Background

Personnel changes have caused institutional knowledge about platform features to become fragmented or lost. Sales and account managers need a reliable, always-available reference when pitching features to clients. This tool captures and exposes that knowledge through a conversational interface.

---

## Approach

**Option B selected:** A lightweight custom web app backed by the Claude API. Feature knowledge lives in structured markdown files maintained by the product team. Claude reads all docs on each query and answers with full context — no vector search needed at this scale (<25 features).

---

## 1. Knowledge Base Structure

One markdown file per feature in a `/features` folder. One additional `platform-overview.md` for general platform questions.

### Feature doc template

```markdown
# Feature Name

## What it does
One paragraph summary — what the feature is, what problem it solves.

## Supported options & configuration
- Option A: what it does, any limits
- Option B: ...

## Known limitations
- Limitation 1
- Limitation 2

## Works well with
- Feature X — why/how they complement each other
- Feature Y — ...

## Conflicts or caution
- Does not work with Feature Z because...
- Client type W often misunderstands this — clarify that...

## Real-world examples
- **Use case:** Company type / scenario — what they used it for, outcome
```

### Key decisions

- Plain markdown files — no database, no CMS, editable by product team directly
- Consistent template ensures Claude always finds what it needs
- Up to ~25 feature files — all fit in a single Claude context window
- Knowledge gaps are surfaced by Claude rather than filled with guesses (see AI layer)

---

## 2. Chat Interface

A single-page web app at one shared internal URL. No login required.

### Layout

- **Left sidebar:** list of all features as quick links — clicking pre-fills a prompt for that feature
- **Main area:** standard chat conversation (back-and-forth)
- **Clear button:** start a fresh session

### Supported query types

- "I'm pitching [feature] to [client type], what should I know?"
- "Can I use [feature X] with [feature Y]?"
- "What are the limitations of [feature]?"
- "Give me a real-world example I can use in a demo"

### Out of scope (v1)

- No user accounts or conversation history
- No file uploads
- No in-UI editing of the knowledge base

### Tech

Lightweight web app — HTML/JS frontend, small Node or Python backend. Deployed internally. Estimated build: a few days with developer involvement.

---

## 3. AI Layer

On each message, the app loads all feature markdown files and sends them to Claude alongside the user's question. At <25 features this fits in one request — no ranking or retrieval step needed.

### System prompt behavior

Claude is instructed to:

- Act as an internal assistant for sales and account managers
- Always surface relevant limitations, not just feature strengths
- Suggest related or complementary features when relevant
- **Never hallucinate.** If the docs don't cover something, respond explicitly: *"The documentation doesn't cover this — worth checking with the product team and adding it to the knowledge base."*

### The "I don't know" behavior

This is a core design requirement. Confident-sounding gaps are dangerous in a sales context. When Claude hits missing information it flags it clearly, which also naturally drives the living database: each gap is a prompt for the product team to fill in.

---

## Next Steps

1. Create the `/features` folder and write the first batch of feature docs (product team)
2. Build the web app shell with the feature sidebar and chat UI
3. Wire up the Claude API with the system prompt and doc loading
4. Internal review with one or two sales/account managers
5. **v2 consideration:** Automated code scanning to keep supported options and limits in sync with the codebase

---

## Living Database Model

The knowledge base is intentionally incomplete at launch. The product team adds and updates docs as knowledge resurfaces — through sales calls, client questions, or developer input. Claude's explicit "I don't know" responses make gaps visible and actionable.
