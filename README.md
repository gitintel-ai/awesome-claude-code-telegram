<p align="center">
  <h1 align="center">awesome-claude-code-telegram</h1>
  <p align="center">
    <strong>The missing Telegram superpowers for Claude Code</strong>
  </p>
  <p align="center">
    <a href="LICENSE"><img src="https://img.shields.io/badge/license-Apache%202.0-blue.svg" alt="License"></a>
    <img src="https://img.shields.io/badge/runtime-Bun-f472b6" alt="Bun">
    <img src="https://img.shields.io/badge/tools-15-green" alt="14 Tools">
    <img src="https://img.shields.io/badge/scheduler-built--in-orange" alt="Built-in Scheduler">
    <img src="https://img.shields.io/badge/MCP-compatible-purple" alt="MCP Compatible">
  </p>
</p>

---

The official Claude Code Telegram plugin gives you 3 tools: `reply`, `react`, `edit_message`.

This plugin gives you **15 tools**, a **built-in task scheduler**, and handles **documents, voice, and interactive buttons** — all in a single process with zero polling conflicts.

## Table of Contents

- [See It In Action](#see-it-in-action)
- [What You Get](#what-you-get)
- [Built-in Scheduler](#built-in-scheduler)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Comparison](#comparison)
- [Architecture](#architecture)
- [Security](#security)
- [Contributing](#contributing)
- [Credits](#credits)

---

## See It In Action

**Inline keyboard for approvals:**

```
Claude → Telegram:
┌─────────────────────────────────────────┐
│ Deploy gitintel v0.1.1 to production?   │
│                                         │
│ Changes: 3 files, bug fix for scan cmd  │
│ Tests: 76/76 passing                    │
│                                         │
│  [ ✅ Deploy ]  [ ❌ Cancel ]           │
└─────────────────────────────────────────┘

CEO taps "Deploy" →

Claude receives: callback_query with data "deploy_approved"
Claude: executes deployment, replies with result
```

**Scheduled morning brief (fires automatically at 9 AM):**

```
Claude → Telegram (9:00 AM IST):
┌─────────────────────────────────────────┐
│ 🌅 Morning Brief — March 22            │
│                                         │
│ ▸ GitIntel: v0.1.0 live, 8 downloads   │
│ ▸ CI: all green                         │
│ ▸ Pending: HN launch post ready        │
│ ▸ Today: deploy blog, prep launch day  │
└─────────────────────────────────────────┘
```

**Document handling:**

```
CEO sends: quarterly_report.pdf

Claude receives: file downloaded to inbox/
Claude: reads PDF, extracts key metrics, responds with summary
```

---

## What You Get

### 15 Tools (vs 4 in official)

**Messaging**

| Tool | What it does | Example |
|------|-------------|---------|
| `reply` | Send with **MarkdownV2/HTML** formatting | `reply(chat_id, "<b>Done</b>", parse_mode: "HTML")` |
| `react` | Emoji reaction | `react(chat_id, msg_id, "🔥")` |
| `edit_message` | Update sent messages | `edit_message(chat_id, msg_id, "Updated text")` |
| `download_attachment` | **Lazy file download** from Telegram | `download_attachment(file_id)` → local path |
| `send_keyboard` | **Inline buttons** | `send_keyboard(chat_id, "Approve?", [[{label: "Yes", callback_data: "yes"}]])` |
| `answer_callback` | Acknowledge button tap | `answer_callback(query_id, "Approved!")` |
| `pin_message` | Pin important updates | `pin_message(chat_id, msg_id)` |
| `unpin_message` | Unpin | `unpin_message(chat_id, msg_id)` |
| `set_commands` | Register `/commands` menu | `set_commands([{command: "status", description: "System status"}])` |
| `delete_message` | Clean up messages | `delete_message(chat_id, msg_id)` |
| `forward_message` | Forward between chats | `forward_message(from, to, msg_id)` |

**Scheduling**

| Tool | What it does | Example |
|------|-------------|---------|
| `schedule_task` | Cron-based recurring/one-shot | `schedule_task(id, "0 9 * * *", "Morning brief", chat_id)` |
| `schedule_delay` | Human-readable delay | `schedule_delay(id, "30m", "Check CI status", chat_id)` |
| `unschedule_task` | Remove task | `unschedule_task("flight_ai302")` |
| `update_task` | Pause/resume/modify | `update_task(id, {enabled: false})` |
| `list_scheduled` | View all tasks | `list_scheduled()` or `list_scheduled(tag: "ops")` |

### 3 New Message Handlers

```
                    Official Plugin          This Plugin
                    ───────────────          ───────────
Text messages       ✓                        ✓
Photos              ✓                        ✓
Documents/PDFs      ✗                        ✓ → downloaded, path sent to Claude
Voice messages      ✗                        ✓ → saved as .ogg, path sent
Button callbacks    ✗                        ✓ → routed as channel notification
```

---

## Built-in Scheduler

A cron engine that runs **inside the plugin** — same process as the Telegram bot. No separate daemon, no polling conflicts, no infrastructure.

### How It Works

```
schedule.json defines tasks with cron expressions
        ↓
Plugin checks every 30 seconds
        ↓
When a task is due → sends MCP notification to Claude
        ↓
Claude processes it like any message
        ↓
Claude calls reply/send_keyboard/etc to deliver results
```

### Schedule Config

```json
{
  "tasks": [
    {
      "id": "morning_brief",
      "cron": "57 8 * * *",
      "prompt": "Generate the morning brief. Include: venture status, pending items, today's plan.",
      "chat_id": "YOUR_CHAT_ID",
      "enabled": true,
      "recurring": true,
      "description": "Daily brief at ~9 AM"
    },
    {
      "id": "flight_check",
      "cron": "*/30 15-20 * * *",
      "prompt": "Check flight AI-302 status. Alert on delays or gate changes.",
      "chat_id": "YOUR_CHAT_ID",
      "enabled": true,
      "recurring": true,
      "urgent": true,
      "expires_at": "2026-03-22T20:00:00+05:30",
      "tags": ["travel"],
      "description": "Track flight AI-302 until 8 PM IST"
    }
  ],
  "timezone": "Asia/Kolkata",
  "quiet_hours": { "start": 1, "end": 7 }
}
```

### Scheduler Features

| Feature | Description |
|---------|-------------|
| **Cron expressions** | Standard 5-field: `M H DoM Mon DoW` |
| **Timezone** | IANA timezone string (e.g., `Asia/Kolkata`, `America/New_York`) |
| **Quiet hours** | Non-urgent tasks pause during specified hours |
| **Expiry** | `expires_at` — task auto-disables after a timestamp |
| **Max runs** | `max_runs: 6` — fire 6 times then stop |
| **Tags** | Categorize and filter: `list_scheduled(tag: "ops")` |
| **Urgent** | `urgent: true` — bypasses quiet hours |
| **Human delays** | `schedule_delay("2h")` instead of computing cron |
| **One-shot** | `recurring: false` — fire once, auto-delete |
| **Debouncing** | Won't fire twice in the same minute |

### Real-World Use Cases

| Scenario | Cron | Options |
|----------|------|---------|
| Daily morning brief | `57 8 * * *` | recurring |
| Flight delay alerts | `*/30 15-20 * * *` | expires_at, urgent |
| "Remind me in 2 hours" | — | `schedule_delay("2h")` |
| Watch CI pipeline | `*/5 * * * *` | max_runs: 30 |
| Weekly news digest | `3 9 * * 6` | recurring, tags: ["research"] |
| Website uptime | `17 * * * *` | recurring, tags: ["ops"] |
| Post-launch monitoring | `0 */2 * * *` | expires_at: 3 days |
| Monthly billing reminder | `0 10 1 * *` | recurring, tags: ["finance"] |

---

## Quick Start

**Prerequisites:** [Bun](https://bun.sh/) runtime, a [Telegram bot token](https://t.me/BotFather), [Claude Code](https://claude.ai/claude-code).

### 1. Clone and install

```bash
git clone https://github.com/gitintel-ai/awesome-claude-code-telegram.git
cd awesome-claude-code-telegram
bun install
```

### 2. Set your bot token

```bash
mkdir -p ~/.claude/channels/telegram
echo "TELEGRAM_BOT_TOKEN=123456789:AAH..." > ~/.claude/channels/telegram/.env
chmod 600 ~/.claude/channels/telegram/.env
```

### 3. Create a schedule (optional)

```bash
cat > ~/.claude/channels/telegram/schedule.json << 'EOF'
{
  "tasks": [],
  "timezone": "Asia/Kolkata",
  "quiet_hours": { "start": 1, "end": 7 }
}
EOF
```

### 4. Register as MCP server

Add to your Claude Code MCP config (`~/.claude/mcp.json` or project `.mcp.json`):

```json
{
  "mcpServers": {
    "telegram": {
      "command": "bun",
      "args": ["/path/to/awesome-claude-code-telegram/server.ts"]
    }
  }
}
```

Or run directly: `bun server.ts`

---

## Configuration

### Access Control

Managed via `~/.claude/channels/telegram/access.json`:

```json
{
  "dmPolicy": "pairing",
  "allowFrom": ["123456789"],
  "groups": {},
  "ackReaction": "⚡",
  "replyToMode": "first",
  "chunkMode": "newline"
}
```

| Field | Options | Default |
|-------|---------|---------|
| `dmPolicy` | `"pairing"`, `"allowlist"`, `"disabled"` | `"pairing"` |
| `ackReaction` | Any Telegram-whitelisted emoji | none |
| `replyToMode` | `"first"`, `"all"`, `"off"` | `"first"` |
| `chunkMode` | `"length"`, `"newline"` | `"length"` |
| `textChunkLimit` | 1–4096 | 4096 |

---

## Comparison

```
                                    Official     This Plugin
                                    ────────     ───────────
Tools                               3            14
Message handlers                    2            5

Messaging
  Text + photos                     ✓            ✓
  MarkdownV2 / HTML formatting      ✗            ✓
  Inline keyboards                  ✗            ✓
  Document / PDF handling           ✗            ✓
  Voice messages                    ✗            ✓
  Pin / unpin                       ✗            ✓
  Delete messages                   ✗            ✓
  Forward messages                  ✗            ✓
  Bot commands menu                 ✗            ✓

Scheduling
  Recurring tasks                   ✗            ✓
  One-shot reminders                ✗            ✓
  Quiet hours                       ✗            ✓
  Task expiry                       ✗            ✓
  Human delay syntax                ✗            ✓

Security
  Pairing + allowlist               ✓            ✓
  Outbound chat gating              ✓            ✓
  Path traversal protection         ✓            ✓ (enhanced)
  Callback query validation         n/a          ✓
  File permission hardening         partial      ✓
```

---

## Architecture

```
┌──────────────────────┐
│   Claude Code        │
│   (your session)     │
└──────────┬───────────┘
           │ stdio (MCP protocol)
┌──────────▼───────────┐
│  awesome-claude-     │
│  code-telegram       │
│                      │
│  ├─ Messaging        │  reply, keyboard, pin, delete, forward
│  ├─ Scheduler        │  cron eval, timezone, quiet hours
│  └─ Handlers         │  text, photo, doc, voice, callback
│                      │
└──────────┬───────────┘
           │ Grammy (Telegram Bot API)
┌──────────▼───────────┐
│   Telegram           │
└──────────────────────┘
```

**Single process.** The plugin is both the messaging bridge and the scheduler. No separate bot = no `409 Conflict` polling errors.

---

## Security

Inherits the official plugin's security model with additional hardening:

| Layer | Protection |
|-------|-----------|
| **Access control** | Pairing mode, allowlist, group @mention gating |
| **Outbound gating** | All tools validate `chat_id` against allowlist before sending |
| **Path traversal** | Document filenames sanitized (strips `/`, `\`, `..`), state files blocked |
| **Callback validation** | Only callback queries received in this session can be answered |
| **File permissions** | `access.json` 0o600, `schedule.json` 0o600, `inbox/` 0o700 |
| **Token isolation** | `.env` file with 0o600 permissions, never in git |
| **Prompt injection** | Scheduler content treated as untrusted (same as user messages) |
| **Approved ID validation** | Pairing approval files validated as numeric Telegram IDs |

---

## Contributing

Contributions welcome. This is a fork of Anthropic's official plugin — the goal is to upstream the best features.

1. Fork the repo
2. Create a feature branch (`git checkout -b feat/my-feature`)
3. Make changes to `server.ts`
4. Test with `bun server.ts` connected to a test bot
5. Verify build: `bun build --target=bun server.ts --outfile=/dev/null`
6. Submit a PR

**Areas that need help:**
- Demo GIF showing inline keyboards in action
- Windows-specific token security (NTFS ACLs)
- Inbox file cleanup (TTL-based eviction)
- More message type handlers (stickers, location, contacts)

---

## Credits

Forked from Anthropic's official Claude Code Telegram plugin.
Copyright 2024-2026 Anthropic, PBC. Original licensed under Apache License 2.0.

See [NOTICE](NOTICE) for full attribution.

## License

Apache License 2.0 — see [LICENSE](LICENSE)
