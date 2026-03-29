# 🤖 The Agentic Edge — AI Newsletter Workflow

> **N8n automation** that listens on both **Gmail** and **Slack**, detects newsletter requests, generates a premium AI-written newsletter via **OpenAI GPT-4o-mini**, and delivers it as a beautifully formatted HTML email.

---

## 📋 Overview

| Property | Value |
|---|---|
| Workflow ID | `cau1LdRvBnNzkYEz` |
| Trigger modes | Gmail (polling) + Slack (webhook) |
| AI Model | GPT-4o-mini via OpenAI |
| Output | HTML email via Gmail SMTP |
| Execution order | v1 |
| Status | Inactive (enable to deploy) |

---

## 🏗️ Architecture

```
Gmail Trigger ──┐
                ├──► IF (keyword check) ──► Switch (route by source)
Slack Trigger ──┘         │                    │           │
                        [false]            [Gmail]      [Slack]
                           │                  │             │
                    No Operation         Edit Fields   Edit Fields
                                        (normalize)   (normalize)
                                               │             │
                                               └──────┬──────┘
                                                      │
                                               AI Agent (GPT-4o-mini)
                                                      │
                                              Code in JavaScript
                                              (HTML builder)
                                                      │
                                              Send a message
                                              (Gmail SMTP)
```

---

## 🔧 Node Reference

### 1. Gmail Trigger
- **Type:** `n8n-nodes-base.gmailTrigger`
- **Poll interval:** Every minute
- **Credentials:** Gmail OAuth2
- **Output fields used:** `snippet`, `Subject`, `From`, `To`

### 2. Slack Trigger
- **Type:** `n8n-nodes-base.slackTrigger`
- **Events:** `any_event`, `app_mention`, `message`
- **Channel:** `all-socialeagle` (`C0AP2V7N4KH`)
- **Credentials:** Slack API
- **Output fields used:** `text`, `channel_type`

### 3. IF — Keyword Gate
- **Purpose:** Only continues if the message contains the word `newsletter`
- **Gmail condition:** `$json.Subject.toLowerCase().includes("newsletter")`
- **Slack condition:** `$json.text.toLowerCase().includes("newsletter")`
- **Combinator:** OR (either match passes)
- **False branch:** Routes to `No Operation` — workflow ends silently

### 4. Switch — Source Router
Detects which trigger fired by checking **data shape** (not node reference — avoids "node hasn't been executed" errors):

| Output | Condition | Routes to |
|---|---|---|
| `Gmail` | `$json.snippet` is not empty | Gmail trigger (Set node) |
| `Slack` | `$json.channel_type` is not empty | Slack Trigger1 (Set node) |

### 5. Gmail trigger (Set node)
Normalises Gmail data into a unified schema:
```
from_email        ← $('Gmail Trigger').item.json.From
to_email          ← $json.To
subject           ← $json.Subject
newsletter_content ← $json.snippet
```

### 6. Slack Trigger1 (Set node)
Normalises Slack data into the same unified schema:
```
from_email         ← hardcoded: vickypaiyaa@gmail.com
to_email           ← hardcoded: vickypaiyaa@gmail.com
subject            ← $('Slack Trigger').item.json.text
newsletter_content ← $('Slack Trigger').item.json.text
```
> **Note:** Slack doesn't expose sender email by default. The email is hardcoded here. To make it dynamic, enable the `users:read.email` scope in your Slack app and fetch it via the Users API.

### 7. AI Agent
- **Type:** `@n8n/n8n-nodes-langchain.agent`
- **Model:** GPT-4o-mini (temperature: 0.7, max tokens: 3000)
- **Memory:** Simple Buffer Window (keyed on `newsletter_content`)
- **Response format:** `json_object` (forced structured JSON)
- **Prompt input:** `$json.newsletter_content`

**System prompt instructs the model to return this exact JSON shape:**
```json
{
  "title": "...",
  "intro": "...",
  "hero_stats": [{ "value": "...", "label": "..." }, ...],
  "feature": { "category": "...", "title": "...", "body": "..." },
  "quote": { "text": "...", "source": "..." },
  "developments": [{ "icon": "⚡", "title": "...", "body": "..." }, ...],
  "list_section": { "title": "...", "items": [...] },
  "tags": ["...", "..."],
  "cta": { "title": "...", "body": "...", "button_text": "...", "url": "..." },
  "footer": "..."
}
```

### 8. Code in JavaScript
- **Type:** `n8n-nodes-base.code`
- **Purpose:** Parses the AI JSON response and injects it into a full premium HTML email template
- **Key logic:**
  - `safeGet(nodeName)` — wraps `$node[name].json` in try/catch (prevents "node not executed" errors)
  - Detects `isGmail` / `isSlack` from which safeGet returned data
  - Extracts `topic` and `senderEmail` from the correct source
  - Builds numbered list HTML, tag chips, and the complete `htmlBody` string
  - Outputs: `htmlBody`, `emailSubject`, `toEmail`, `triggerSource`, `topic`

### 9. Send a message (Gmail)
- **Type:** `n8n-nodes-base.gmail` (send mode)
- **To:** `$json.toEmail`
- **Subject:** `$json.emailSubject`
- **Body:** `$json.htmlBody` (full HTML)
- **Credentials:** Gmail OAuth2

---

## ⚙️ Setup Guide

### Prerequisites
- N8n instance (cloud or self-hosted)
- Gmail account with OAuth2 credentials configured in N8n
- Slack workspace with a Bot app created
- OpenAI API key

### Step 1 — Import the workflow
1. In N8n, go to **Workflows → Import**
2. Paste the JSON or upload the file
3. Save

### Step 2 — Configure credentials
| Node | Credential type | Where to create |
|---|---|---|
| Gmail Trigger | Gmail OAuth2 | N8n → Credentials → New → Gmail OAuth2 |
| Send a message | Gmail OAuth2 | Same credential, reuse |
| Slack Trigger | Slack API | N8n → Credentials → New → Slack API |
| OpenAI Chat Model | OpenAI API | N8n → Credentials → New → OpenAI |

### Step 3 — Configure Slack app
1. Go to [api.slack.com/apps](https://api.slack.com/apps) → Create App
2. Add **Bot Token Scopes:** `channels:history`, `app_mentions:read`, `chat:write`
3. Enable **Event Subscriptions** → point to your N8n webhook URL
4. Subscribe to: `message.channels`, `app_mention`
5. Install app to your workspace

### Step 4 — Update hardcoded email (Slack path)
In the **Slack Trigger1** Set node, update `from_email` and `to_email` from `vickypaiyaa@gmail.com` to your own address, or make it dynamic:
```
to_email ← your-email@domain.com
```

### Step 5 — Activate
Toggle the workflow to **Active** in the top-right of the editor.

---

## 🚀 How to Trigger

### Via Gmail
Send an email to your connected Gmail account with the word **"newsletter"** anywhere in the subject line:
```
Subject: Create newsletter about AI Agents in production
Body: (anything — the snippet is used as context)
```

### Via Slack
Post a message in the `all-socialeagle` channel containing the word **"newsletter"**:
```
newsletter about multi-agent orchestration frameworks
```
Or @mention the bot:
```
@AgenticEdge newsletter on RAG vs fine-tuning tradeoffs
```

---

## 📧 Newsletter Output

The generated email includes:

| Section | Description |
|---|---|
| Header | Logo, issue badge, date, topic, read time |
| Hero | AI-generated title, intro paragraph, 3 stat widgets |
| Feature card | Deep-dive article with category tag and gradient accent bar |
| Quote block | Pull quote with left-border treatment |
| 3-column grid | Key developments mini-cards with icons |
| Numbered list | 5 top insights with large display numbers |
| CTA block | Call to action with button |
| Footer | Unsubscribe, privacy links, source attribution |

**Design:** Dark editorial aesthetic — `#0F0F12` base, Playfair Display serif headlines, DM Sans body, `#7B6FFF` accent.

---

## 🛠️ Customisation

### Change the AI model
In the **OpenAI Chat Model** node, switch `gpt-4o-mini` to `gpt-4o` for higher quality output (slower, more expensive).

### Change the newsletter topic keyword
In the **IF** node, replace `"newsletter"` with any trigger word you prefer.

### Add a third trigger (e.g. webhook)
1. Add a Webhook node
2. Connect it to the **IF** node
3. Add a new **Switch** route checking for a field unique to webhook payloads
4. Add a new **Set** node to normalise webhook data into `newsletter_content` + `from_email`

### Make Slack email dynamic
Add an **HTTP Request** node after the Slack Trigger:
```
GET https://slack.com/api/users.info?user={{ $json.event.user }}
Authorization: Bearer xoxb-your-bot-token
```
Then reference `$json.user.profile.email` in the Set node.

---

## ⚠️ Known Limitations

| Limitation | Detail |
|---|---|
| Gmail snippet length | Gmail `snippet` is capped at ~100 chars. Long email bodies are truncated as the AI's topic context |
| Slack sender email | Hardcoded — requires `users:read.email` scope + API call to make dynamic |
| Memory scope | Simple Buffer Window memory is keyed on `newsletter_content` — same topic text reuses the same session |
| Poll latency | Gmail trigger polls every minute, so there's up to a 60-second delay from email receipt to newsletter generation |

---

## 📁 File Structure

```
├── README.md                    ← This file
├── workflow.json                ← N8n workflow export
├── n8n_code_node_fixed.js       ← Standalone JS for the Code node
└── ai_agents_newsletter.html    ← Static preview of newsletter template
```

---

## 🔑 Credentials Summary

```
Vignesh Gmail account   → gmailOAuth2      → ID: RS55S7TBhtnXIVSr
Vignesh OpenAi account  → openAiApi        → ID: 6lBzo07NcIfhlhQS
Slack account           → slackApi         → ID: MrFj2BnVE57acgEh
```
> ⚠️ Never commit real credential IDs to a public repository. Rotate keys if this file is shared publicly.

---

## 👨‍💻 Author

*Vignesh Nagarajan - AI Engineer*
