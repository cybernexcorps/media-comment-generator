# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

**Media Comment Generator** — AI-agent n8n workflow that generates expert media comments in Russian for DDVB agency speakers. Called by a Telegram Router or directly via HTTP webhook.

**Current versions:**
- `media-comment-generator-agent-v1.0.json` — 3-agent pipeline (deprecated/reference)
- `media-comment-generator-agent-v2.0.json` — 4-agent pipeline with Strategist + Web webhook (active)

## Input Format

Sent via Telegram bot (routed from Telegram Router) or POST webhook:

```
PERSON: maria
LINK: https://www.sostav.ru/publication/example
QUESTION: Как вы оцениваете текущие тренды в брендинге?
```

Field aliases: `LINK` / `ССЫЛКА` / `CONTEXT` / `КОНТЕКСТ` and `QUESTION` / `ВОПРОС`. Default PERSON is `maria` → `maria_arkhangelskaya`.

## v2.0 Architecture (4-Agent Pipeline)

```
[Telegram Router] ──► Execute Workflow Trigger ──►┐
[Web POST /media-comment/generate] ──► Normalize ──►┘
     ↓
Parse Request → Validate Fields → Fetch Profile (GitHub)
     ↓
Strategist Agent (Claude + Tavily Extract)
  → builds structured research brief
     ↓
Research Agent (Claude + Perplexity sonar-pro)
  → structured report with article + market context + citations
     ↓
Writer Agent (Claude)
  → КОНТЕКСТ / КОММЕНТАРИЙ / ПРОВЕРКА sections
     ↓
Extract Comment → Humanizer Agent (GPT-4o)
  → 24-pattern AI removal, adds personality
     ↓
Compliance Review → Route Output
  ├─ Web: Build Callback → POST to callbackUrl
  └─ Telegram: Format → Send Final Comment
```

## Key Implementation Details

**Speaker profiles** are fetched from GitHub raw:
```
https://raw.githubusercontent.com/cybernexcorps/ceo-comment-writer/main/profiles/{profileId}.json
```
Fallback profile (maria_arkhangelskaya) is hardcoded in `Parse Profile` node if GitHub fetch fails.

**Prompt node IDs (for API patching):**
- `b2-0002` Parse Request, `b2-0008` Build Strategist Prompt, `b2-0011` Build Research Prompt
- `b2-0015` Build Writer Prompt, `b2-0019` Build Humanizer Prompt

**Humanizer (b2-0019) content-lock rule:** STYLE editor only — must not rewrite content. Rewriting from scratch causes loss of specific facts (ОКВЭД codes, named details) from Writer output. Current: v2.2, deployed 2026-02-27, ~2829 chars system message.

**Perplexity search (v2.0):** Research Agent uses the native `n8n-nodes-base.perplexityTool` node with model `sonar-pro` and `searchRecency: month`. Russian context is injected into the query text (prefix with `Россия, российский рынок`). The search query is built by `Build Research Prompt` from the Strategist's `## АНАЛИЗ ТЕМЫ` section, capped at 2000 chars.

**Web webhook trigger (v2.0):** `POST /media-comment/generate` with body `{ jobId, person, articleUrl, question, callbackUrl }`. Responds 202 immediately, then POSTs result JSON to `callbackUrl` with `X-DDVB-API-Key` header.

**Compliance thresholds:** 1200–2200 chars (with ±15% tolerance), Russian Cyrillic >100 chars, no emojis, max 2 `**bold**` markers, question relevance keyword check.

**Humanizer framework:** 24 AI-pattern categories (content, language, style, communication, filler). Output must be Russian only — GPT-4o sometimes reverts to English if system prompt context is lost.

**Format Telegram Output node (b2-0023):** Telegram output format:
```
Спикер: Мария Архангельская
Генеральный директор и управляющий партнёр, DDVB

[comment text]
```
- Uses `profile.name_ru` for full name, `profile.title_ru` + `profile.company` for second line.
- No character count line (removed).
- ⚠️ `inputData.profile` is `undefined` at this node — Compliance Review does not forward the profile object. Must fall back to `$('Finalize Profile').first().json.profile`.

## Credentials Required

| Credential | n8n Name | Nodes |
|------------|----------|-------|
| Anthropic API | `DDVB Anthropic` | Strategist Model, Research Model, Writer Model |
| OpenAI API | `DDVB OpenAI` | Humanizer Model (GPT-4o) |
| Tavily API | `Tavily account` | Strategist Tavily Extract Tool |
| Perplexity API | `Perplexity account` | Perplexity Research Tool |
| Telegram Bot | `DDVB Test Bot` | Send nodes |
| GitHub | `GitHub account` | Fetch Profile from GitHub, Get Default Profile |

## Linking to Telegram Router

After import, update the Telegram Router workflow (ID: `ArQkIrLPcz5dOMvB`), node `exec-comment-writer` → change the target workflow ID to the newly imported workflow's ID.

## Differences: v1.0 vs v2.0

| Aspect | v1.0 | v2.0 |
|--------|------|-------|
| Agents | Research + Comment Generator + Humanizer | **Strategist** + Research + Writer + Humanizer |
| Search | Yandex MCP SSE endpoint | **Perplexity sonar-pro** (native n8n node) |
| Triggers | Execute Workflow only | Execute Workflow + **Web Webhook** |
| Output routing | Single Telegram path | Switch: Web callback or Telegram |
| Nodes | 20 | ~43 |

## File Notes

- `AGENTS.md` — Contains Codex (OpenAI) skills configuration; not relevant to Claude Code.
- `yandex_prompt_example.md` — Legacy reference for Yandex research brief format. Archived; Perplexity now handles search.
