# Telegram Message Handler Agent

An n8n workflow that automatically picks up incoming Telegram messages, logs them to HubSpot as contacts and notes, drafts AI-powered replies using Claude, sends them back to the user on Telegram, and records the outbound reply in HubSpot.

---

## Workflow Diagram

```
[Telegram Trigger]
       |
       v
[Extract Message Data]
  chat_id, username, first_name, message_text, timestamp
       |
       v
[Search HubSpot Contact]
  Search by synthetic email: telegram_{user_id}@noreply.telegram
       |
       v
[Contact Found?] ──── TRUE ──► [Get Existing Contact ID]
       |                                  |
     FALSE                                |
       |                                  |
       v                                  |
[Create HubSpot Contact]                  |
       |                                  |
       v                                  |
[Set New Contact ID]                      |
       |                                  |
       └──────────────────┬───────────────┘
                          v
                   [Merge Contact]
                          |
                          v
               [Log Inbound Message]
                 HubSpot note: [INBOUND via Telegram] ...
                          |
                          v
               [Draft Reply with Claude]
                 claude-sonnet-4-6 → friendly 2-3 sentence reply
                          |
                          v
                [Extract AI Reply]
                          |
                          v
               [Send Telegram Reply]
                 Posts draft back to the original chat
                          |
                          v
               [Log Outbound Reply]
                 HubSpot note: [OUTBOUND - AI Draft] ...
```

---

## Prerequisites

- n8n instance (self-hosted or cloud, v1.30+)
- Telegram Bot Token (from @BotFather)
- HubSpot Private App Token
- Anthropic API Key

---

## Setup

### 1. Create a Telegram Bot

1. Open Telegram and search for **@BotFather**
2. Send `/newbot` and follow the prompts
3. Copy the **Bot Token** (format: `123456789:ABCdef...`)

### 2. Create a HubSpot Private App

1. In HubSpot: **Settings → Integrations → Private Apps → Create Private App**
2. Give it a name (e.g., "Telegram Agent")
3. Under **Scopes**, enable:
   - `crm.objects.contacts.read`
   - `crm.objects.contacts.write`
   - `crm.objects.notes.read`
   - `crm.objects.notes.write`
4. Click **Create App** and copy the **Access Token**

### 3. Get an Anthropic API Key

1. Go to [console.anthropic.com](https://console.anthropic.com)
2. Navigate to **API Keys** and create a new key
3. Copy the key

### 4. Import the Workflow into n8n

1. Open your n8n instance
2. Go to **Workflows → Add Workflow → Import from File**
3. Select `workflow.json` from this project
4. The workflow will appear with all nodes pre-wired

### 5. Configure Credentials in n8n

You need to create **3 credentials** in n8n before activating the workflow.

#### Credential 1 — Telegram Bot API
- Type: **Telegram API**
- Name: `Telegram Bot API`
- Access Token: your Telegram bot token

#### Credential 2 — HubSpot API
- Type: **HTTP Header Auth**
- Name: `HubSpot API`
- Name field: `Authorization`
- Value field: `Bearer YOUR_HUBSPOT_PRIVATE_APP_TOKEN`

#### Credential 3 — Anthropic API
- Type: **HTTP Header Auth**
- Name: `Anthropic API`
- Name field: `x-api-key`
- Value field: `YOUR_ANTHROPIC_API_KEY`

After creating each credential, open the corresponding node in the workflow and select the credential from the dropdown.

### 6. Activate the Workflow

1. Click **Save** in the workflow editor
2. Toggle the workflow to **Active**
3. n8n will register a webhook with Telegram automatically

---

## How It Works

| Step | Node | What Happens |
|------|------|--------------|
| 1 | Telegram Trigger | Receives incoming messages via Telegram Bot API webhook |
| 2 | Extract Message Data | Parses `chat_id`, `username`, `message_text`, `timestamp` |
| 3 | Search HubSpot Contact | Looks up contact by synthetic email `telegram_{id}@noreply.telegram` |
| 4 | Contact Found? | Branches: existing contact (true) vs new contact (false) |
| 5a | Get Existing Contact ID | Extracts the HubSpot contact ID from search results |
| 5b | Create HubSpot Contact | Creates a new contact with name + synthetic email |
| 6 | Merge Contact | Reunites both branches with a normalized `contact_id` |
| 7 | Log Inbound Message | Creates a HubSpot note tagged `[INBOUND via Telegram]` |
| 8 | Draft Reply with Claude | Sends the message to Claude (`claude-sonnet-4-6`) for a draft response |
| 9 | Extract AI Reply | Pulls the text from Claude's response |
| 10 | Send Telegram Reply | Posts the AI-drafted reply back to the Telegram chat |
| 11 | Log Outbound Reply | Creates a HubSpot note tagged `[OUTBOUND - AI Draft]` |

---

## Customising the AI Prompt

Open the **Draft Reply with Claude** node and edit the `system` field in the JSON body:

```json
"system": "You are a concise, friendly customer support assistant. Draft a helpful reply to the user message in 2-3 sentences. Be warm and professional."
```

Adjust the tone, length, or persona to match your use case.

---

## Contact Identity Strategy

Since Telegram users don't expose email addresses, this workflow uses a synthetic email address to uniquely identify each Telegram user in HubSpot:

```
telegram_{telegram_user_id}@noreply.telegram
```

This ensures each Telegram user maps to exactly one HubSpot contact, and repeat messages from the same user are appended to their existing record rather than creating duplicates.

---

## Environment Variables Reference

See `.env.example` for all credential values needed:

| Variable | Description |
|----------|-------------|
| `TELEGRAM_BOT_TOKEN` | From @BotFather |
| `HUBSPOT_API_KEY` | HubSpot Private App access token |
| `ANTHROPIC_API_KEY` | Anthropic console API key |
"# telegram-message-handler-agent" 
