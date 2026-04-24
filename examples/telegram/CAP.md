---
name: telegram
description: Send messages via Telegram Bot API
version: "1.0"
tags: messaging, telegram, notifications
runtime: repl
requires: [store, web]
schema:
  input:
    chat_id:
      type: string
      required: true
      description: Telegram chat ID to send to
    text:
      type: string
      required: true
      description: Message text (Markdown supported)
  output:
    ok:
      type: boolean
    message_id:
      type: number
---

## Purpose

Send a message to a Telegram chat via the Bot API. Requires a bot token stored in cap_store config table.

## Script

```javascript
async function handler({ chat_id, text }) {
  const cfg = await store({ action: 'query', table: 'config',
    where: 'key = ?', args: ['telegram_bot_token'] })
  const token = cfg?.[0]?.value
  if (!token) throw new Error('telegram_bot_token not configured — run setup first')

  const url = 'https://api.telegram.org/bot' + token + '/sendMessage'
  const body = JSON.stringify({ chat_id, text, parse_mode: 'Markdown' })
  const resp = await web({ action: 'fetch', url, method: 'POST',
    headers: { 'Content-Type': 'application/json' }, body })
  return JSON.parse(resp)
}
```

## Database

```sql
CREATE TABLE cap_telegram__config (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  key TEXT UNIQUE,
  value TEXT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

## Actions

### Setup

To use this cap, you need a Telegram Bot Token:

1. Open [@BotFather](https://t.me/BotFather) in Telegram
2. Send `/newbot` and follow the prompts to create a bot
3. Copy the token BotFather gives you
4. Store it:

```javascript
store({ action: 'upsert', table: 'config',
        data: { key: 'telegram_bot_token', value: '<YOUR_TOKEN>' } })
```

5. Find your chat ID by sending a message to your bot, then:

```javascript
const r = await web({ action: 'fetch',
  url: 'https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates' })
// Look for "chat":{"id": <number>} in the response
```

6. Test it:

```javascript
telegram({ chat_id: '<YOUR_CHAT_ID>', text: 'Hello from cap!' })
```
