# Implementation Guide -- Media Comment Generator v1.0

## Prerequisites

- n8n instance running (self-hosted or cloud)
- Telegram Bot configured (`DDVB Marketing Workspace`)
- Anthropic API key (Claude Sonnet 4.5 access)
- OpenAI API key (GPT-4o access)
- Tavily API key
- Yandex Search MCP endpoint configured

## Step 1: Import Workflow

1. Open n8n dashboard
2. Go to **Workflows** -> **Import from File**
3. Select `media-comment-generator-agent-v1.0.json`
4. The workflow will appear as "Media Comment Generator (AI Agent) v1.0"

## Step 2: Configure Credentials

### Anthropic API (2 nodes)

1. Open **Research Agent Model** node
2. Select or create Anthropic API credential
3. Open **Comment Generator Model** node
4. Select the same Anthropic API credential

### OpenAI API (1 node)

1. Open **Humanizer Model (GPT-4o)** node
2. Select or create OpenAI API credential

### Tavily API (1 node)

1. Open **Tavily Extract Tool** node
2. Select or create Tavily API credential

### Yandex Search MCP (1 node)

1. Open **Yandex Search MCP Tool** node
2. Verify the endpoint URL matches your Yandex MCP deployment
3. Select or create the HTTP Headers Auth credential with your Yandex API key
4. Current default: `https://d5dj4o5pbnqgca1d546v.cmxivbes.apigw.yandexcloud.net:3000/sse`

### Telegram Bot (3 nodes)

The workflow references `DDVB Marketing Workspace` (ID: `BnEQsFHcvbF7jdMc`). If this credential already exists in your n8n instance, it will auto-connect. Nodes using it:
- Send Error: Missing Fields
- Send Final Comment
- Send Error: Workflow Failed

## Step 3: Update Telegram Router

The existing Telegram Router (workflow ID: `ArQkIrLPcz5dOMvB`) has a node called `exec-comment-writer` that currently points to the old Comment Writer workflow (`ZbQ1QdooJhLlz1Io`).

1. Open the Telegram Router workflow
2. Find the **Call Comment Writer** node (Execute Workflow)
3. Change the workflow ID to the **new workflow's ID** (shown in n8n after import)
4. Save the Router workflow

## Step 4: Activate

1. Toggle the workflow to **Active**
2. The workflow is now ready to receive requests from the Telegram Router

## Step 5: Test

Send a test message to the Telegram bot:

```
PERSON: maria
LINK: https://www.sostav.ru/publication/brandtech-trends-2026.html
QUESTION: Как вы оцениваете текущие тренды в брендинге?
```

### Expected Flow

1. Router detects comment format, sends confirmation message, calls this workflow
2. Parse Request extracts PERSON, LINK, QUESTION
3. Validate Fields checks required fields are present
4. Profile fetched from GitHub (maria_arkhangelskaya.json)
5. Research Agent uses Tavily to extract article, Yandex to research topic
6. Comment Generator writes structured comment in Maria's voice
7. Humanizer removes AI patterns, adds personality
8. Compliance Review checks character count and formatting
9. Formatted comment sent to Telegram chat

### Verification Checklist

- [ ] Confirmation message received from Router (not this workflow)
- [ ] Profile fetched successfully (check execution log)
- [ ] Research Agent called both Tavily and Yandex MCP tools
- [ ] Comment generated in Russian, in speaker's voice
- [ ] Humanized comment has no obvious AI patterns
- [ ] Character count within 1200-2200 range
- [ ] Final comment delivered to Telegram with proper formatting

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| No response after sending message | Router not pointing to new workflow | Update Router's exec-comment-writer node |
| "Missing fields" error | Message format incorrect | Check LINK: and QUESTION: fields present |
| Profile fetch 404 | Profile ID not on GitHub | Check profileId mapping, add profile to repo |
| Research Agent timeout | Yandex MCP endpoint down | Check MCP endpoint URL and credentials |
| Tavily extraction empty | Article behind paywall | Tool will continue with Yandex search only |
| Humanizer produces English text | System prompt issue | Check GPT-4o has access to the model |
| Compliance warnings | Comment too short/long | Adjust targetLength in profile or prompt |
| Telegram send error | Bot token expired | Re-authenticate Telegram credential |

## Node Reference

| # | Node | Type | Purpose |
|---|------|------|---------|
| 1 | When Called by Router | executeWorkflowTrigger | Entry point |
| 2 | Parse Request | code | Extract PERSON, LINK, QUESTION |
| 3 | Validate Fields | if | Check required fields |
| 4 | Send Error: Missing Fields | telegram | Error notification |
| 5 | Fetch Profile from GitHub | httpRequest | Load speaker profile JSON |
| 6 | Parse Profile & Build Research Input | code | Prepare data for research |
| 7 | Research Agent | agent | Autonomous research with tools |
| 7a | Research Agent Model | lmChatAnthropic | Claude Sonnet 4.5 |
| 7b | Tavily Extract Tool | tavilyTool | Article content extraction |
| 7c | Yandex Search MCP Tool | mcpClientTool | Russian web search |
| 8 | Build Comment Generation Prompt | code | Combine research + profile |
| 9 | Comment Generator Agent | agent | Write structured comment |
| 9a | Comment Generator Model | lmChatAnthropic | Claude Sonnet 4.5 |
| 10 | Extract Comment | code | Extract KOMMENTARIY section |
| 11 | Build Humanizer Prompt | code | Build v2.11 humanization prompt |
| 12 | Humanizer Agent | agent | Remove AI patterns, add soul |
| 12a | Humanizer Model (GPT-4o) | lmChatOpenAi | GPT-4o |
| 13 | Compliance Review | code | Automated quality checks |
| 14 | Format Telegram Output | code | HTML formatting |
| 15 | Send Final Comment | telegram | Deliver to chat |
| 16 | Send Error: Workflow Failed | telegram | Error handler |
| 17-20 | Sticky Notes (4) | stickyNote | Documentation on canvas |
