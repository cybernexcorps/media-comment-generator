# Media Comment Generator (AI Agent) v1.0

AI-powered media comment generation workflow using autonomous AI agents with connected tools.

## Overview

| Field | Value |
|-------|-------|
| Version | 1.0 |
| Platform | n8n workflow automation |
| Architecture | AI Agent with Tools (Tavily + Yandex MCP) |
| AI Models | Claude Sonnet 4.5 (research + generation), GPT-4o (humanization) |
| Language | Russian (business communication) |
| Trigger | Execute Workflow (called by Telegram Router) |

## Architecture

```
[Telegram Router] --> Execute Workflow --> [THIS WORKFLOW]

Execute Workflow Trigger
  --> Parse Request (PERSON, LINK, QUESTION)
  --> Validate Fields
  --> Fetch Profile from GitHub
  --> Parse Profile & Build Research Input
  --> Research Agent (Claude + Tavily + Yandex MCP)
  --> Build Comment Generation Prompt
  --> Comment Generator Agent (Claude)
  --> Extract Comment
  --> Build Humanizer Prompt
  --> Humanizer Agent (GPT-4o)
  --> Compliance Review
  --> Format Telegram Output
  --> Send Final Comment
```

## Input Format

Sent to Telegram bot (routed by Telegram Router):

```
PERSON: maria
LINK: https://www.sostav.ru/publication/example-article
QUESTION: Как вы оцениваете текущие тренды в брендинге?
```

### Fields

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| PERSON / ПЕРСОНА | No | maria | Speaker alias (maria, ilya, etc.) |
| LINK / ССЫЛКА / CONTEXT / КОНТЕКСТ | Yes | - | Article URL |
| QUESTION / ВОПРОС | Yes | - | Journalist's question |

### Person Aliases

| Alias | Profile ID |
|-------|-----------|
| maria, мария, masha, маша | maria_arkhangelskaya |
| ilya, илья, morozov | ilya_morozov |
| dmitry, дмитрий | dmitry_balobanov |
| vlad, влад | vlad_sitaev |
| toure, туре | toure_benin |
| kirill, кирилл | kirill_fomin |
| katya, катя | ekaterina_antonova |
| julia, юля | julia_shlyahova |
| leonid, леонид | leonid_feigin |

## AI Agents

### 1. Research Agent (Claude Sonnet 4.5)

**Tools:**
- **Tavily Extract** -- extracts full article content from URL
- **Yandex Search MCP** -- researches topic with Russian-language search (search_region: ru)

**Output:** Structured report with topic summary, key facts, market context, statistics, expert opinions, sources.

### 2. Comment Generator Agent (Claude Sonnet 4.5)

Generates a structured media comment in the speaker's voice:
- KONTEKST (context analysis)
- KOMMENTARIY (the comment, 1500-2000 chars)
- PROVERKA (self-check)

Uses full profile injection (expertise, tone, speaking patterns, do-not-say list).

### 3. Humanizer Agent (GPT-4o)

Applies the v2.11 humanization framework:
- 24 AI pattern detection and removal (content, language, style, communication, filler)
- Add soul & personality (opinions, varied rhythm, strategic first person)
- Russian-specific cleanup (remove bureaucratic language, passive voice)
- Preserve speaker voice and factual accuracy

## Compliance Review

Automated checks before delivery:
- Character count within target range (1200-2200)
- Russian Cyrillic text present (>100 characters)
- No AI markers (emoji, excessive bold, English phrases)
- Question relevance check (keyword matching)

## Output

Telegram message format:
```
Спикер: Мария Архангельская
Генеральный директор и управляющий партнёр, DDVB

[comment text]

---
Замечания:        ← only if compliance warnings exist
- …
```
- No character count line.
- Warnings section appended only when compliance checks flag issues.

## Credentials Required

| Credential | Type | Used By |
|------------|------|---------|
| DDVB Marketing Workspace | Telegram Bot API | Error messages, Final comment |
| Anthropic account | Anthropic API | Research Agent, Comment Generator |
| OpenAi account | OpenAI API | Humanizer Agent |
| Tavily account | Tavily API | Article extraction tool |
| Yandex Search MCP | HTTP Headers Auth | Topic research tool |

## Files

| File | Description |
|------|-------------|
| `media-comment-generator-agent-v1.0.json` | n8n workflow (import this) |
| `README.md` | This file |
| `IMPLEMENTATION-GUIDE.md` | Setup and configuration guide |

## Differences from CEO Comment Writer (v2.10)

| Aspect | CEO Comment Writer v2.10 | This Workflow v1.0 |
|--------|--------------------------|-------------------|
| Architecture | Direct API calls | AI Agent with tools |
| Research | Perplexity API (single query) | Tavily Extract + Yandex MCP (agent-orchestrated) |
| Generation | GPT-4o direct | Claude Sonnet 4.5 agent |
| Humanization | GPT-4o direct | GPT-4o agent |
| Input fields | PERSON, МЕДИА, ВОПРОС, КОНТЕКСТ, ДЛИНА | PERSON, LINK, QUESTION |
| Approval | 4-button Telegram approval flow | Direct send after compliance review |
| Feedback loop | Unlimited revisions (max 10) | Not included (v1.0) |
| Memory | Conversational memory per session | Stateless |

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-02-09 | Initial release with AI Agent architecture |
| 2.0 | 2026-02-17 | 4-agent pipeline (Strategist added), Perplexity search, Web webhook trigger, quality gate + retry |
| 2.0.1 | 2026-02-19 | Telegram output: speaker label format, removed char count, fixed profile lookup |
