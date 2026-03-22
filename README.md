# awesome-claude-code-telegram

**Turn Claude Code into an autonomous AI agent on Telegram.** Inline keyboards, scheduled tasks, document handling, formatted messages — everything the official plugin doesn't do yet.

```
You: "Track flight AI-302, departing 8 PM"
Claude: ✅ Scheduled — checking every 30 min, will alert on delays

You: [sends a PDF]
Claude: [reads it, summarizes, responds]

You: [taps "Approve" button on inline keyboard]
Claude: [executes the approved action]
```

> The official Claude Code Telegram plugin has 3 tools.
> This one has **14 tools**, **a built-in scheduler**, and handles documents, voice, and button interactions.

---

## The Problem

Claude Code's official Telegram plugin lets you chat — and that's about it. You can't:

- Schedule morning briefs or recurring tasks
- Send interactive buttons for approvals
- Share PDFs or documents
- Format messages with bold, code blocks, or HTML
- Pin important updates
- Track flights, monitor CI, or set reminders
- Register bot commands in Telegram's menu

This plugin fixes all of that. One file, zero new dependencies beyond what the official plugin already uses.

---

## What You Get

### 14 Tools

| Tool | Description |
|------|-------------|
| `reply` | Send messages with **MarkdownV2 or HTML** formatting + file attachments |
| `react` | Emoji reactions |
| `edit_message` | Update sent messages (with formatting) |
| `send_keyboard` | **Inline buttons** — approval flows, choices, URL buttons |
| `answer_callback` | Acknowledge button taps |
| `pin_message` | Pin important messages |
| `unpin_message` | Unpin messages |
| `set_commands` | Register `/commands` visible in Telegram's menu |
| `delete_message` | Clean up bot messages |
| `forward_message` | Forward between chats |
| `schedule_task` | **Cron-based recurring/one-shot tasks** |
| `schedule_delay` | Schedule with human delays — `"30m"`, `"2h"`, `"1d"` |
| `unschedule_task` | Remove scheduled tasks |
| `update_task` | Pause, resume, or modify tasks |
| `list_scheduled` | View all tasks with status and run history |

### 3 New Message Handlers

| Type | What happens |
|------|-------------|
| **Documents** | PDFs, spreadsheets, any file — downloaded, path sent to Claude for reading |
| **Voice** | Voice messages saved as .ogg — Claude gets the file path |
| **Button taps** | Callback queries routed to Claude as channel notifications |

---

## Built-in Scheduler

A cron-based task scheduler that runs **inside the plugin process**. No separate bot, no polling conflicts, no extra infrastructure.

```json
{
  "tasks": [
    {
      "id": "morning_brief",
      "cron": "57 8 * * *",
      "prompt": "Generate and deliver the morning brief.",
      "chat_id": "YOUR_CHAT_ID",
      "recurring": true,
      "description": "Daily brief at ~9 AM"
    }
  ],
  "timezone": "Asia/Kolkata",
  "quiet_hours": { "start": 1, "end": 7 }
}
```

**Features:**
- Standard 5-field cron with timezone support
- **Quiet hours** — non-urgent tasks pause overnight
- **Expiry** — `expires_at` auto-disables tasks after a timestamp
- **Max runs** — fire N times then stop (perfect for monitoring)
- **Tags** — categorize and filter tasks
- **Urgent flag** — bypasses quiet hours for critical alerts
- **Human delays** — `schedule_delay "2h"` instead of computing cron

**How it works:** When a task is due, the scheduler sends an MCP notification to Claude. Claude processes it like any message and replies via Telegram. One process. Zero conflicts.

### Example Use Cases

| Scenario | Configuration |
|----------|--------------|
| Daily morning brief | `cron: "57 8 * * *"`, recurring |
| Flight delay alerts | `cron: "*/30 15-20 * * *"`, expires_at: departure+1h, urgent |
| "Remind me in 2 hours" | `schedule_delay: "2h"`, one-shot |
| Watch CI pipeline | `cron: "*/5 * * * *"`, max_runs: 30 |
| Weekly news digest | `cron: "3 9 * * 6"`, recurring |
| Website uptime check | `cron: "17 * * * *"`, recurring, tags: ["ops"] |
| Post-launch HN monitoring | `cron: "0 */2 * * *"`, expires_at: 3 days, tags: ["launch"] |

---

## Quick Start

**Prerequisites:** [Bun](https://bun.sh/), a Telegram bot token from [@BotFather](https://t.me/BotFather), Claude Code.

```bash
# 1. Clone
git clone https://github.com/gitintel-ai/awesome-claude-code-telegram.git
cd awesome-claude-code-telegram
bun install

# 2. Set your bot token
mkdir -p ~/.claude/channels/telegram
echo "TELEGRAM_BOT_TOKEN=your-token-here" > ~/.claude/channels/telegram/.env
chmod 600 ~/.claude/channels/telegram/.env

# 3. Run
bun server.ts
```

Or register as an MCP server in your Claude Code config for automatic startup.

---

## Architecture

```
Claude Code Session
    ↕ stdio (MCP protocol)
awesome-claude-code-telegram
    ├── Messaging (reply, react, edit, keyboard, pin, delete, forward)
    ├── Scheduler (cron evaluation, timezone, quiet hours)
    └── Handlers (text, photo, document, voice, callback)
    ↕ Grammy (Telegram Bot API)
Telegram
```

**Single process.** The plugin is the messaging bridge AND the scheduler. No separate bot process = no polling conflicts.

---

## Comparison

| Feature | Official Plugin | This Plugin |
|---------|:-:|:-:|
| Text messages | :white_check_mark: | :white_check_mark: |
| Reactions | :white_check_mark: | :white_check_mark: |
| Edit messages | :white_check_mark: | :white_check_mark: |
| Photo handling | :white_check_mark: | :white_check_mark: |
| **MarkdownV2/HTML formatting** | :x: | :white_check_mark: |
| **Inline keyboards** | :x: | :white_check_mark: |
| **Document/PDF handling** | :x: | :white_check_mark: |
| **Voice messages** | :x: | :white_check_mark: |
| **Scheduled tasks** | :x: | :white_check_mark: |
| **Bot commands menu** | :x: | :white_check_mark: |
| **Pin/unpin** | :x: | :white_check_mark: |
| **Delete messages** | :x: | :white_check_mark: |
| **Forward messages** | :x: | :white_check_mark: |
| **Quiet hours** | :x: | :white_check_mark: |
| **Task expiry & max runs** | :x: | :white_check_mark: |
| Access control (pairing, allowlist, groups) | :white_check_mark: | :white_check_mark: |
| Security (path validation, outbound gating) | :white_check_mark: | :white_check_mark: |

---

## Security

Inherits the official plugin's full security model, plus additional hardening:

- **Access control** — pairing mode, allowlist, group support with @mention
- **Outbound gating** — all tools validate chat_id against allowlist
- **Path traversal protection** — document filenames sanitized, state files blocked
- **Callback validation** — only received callback queries can be answered
- **File permissions** — schedule.json and inbox/ locked to owner (0o600/0o700)
- **Prompt injection defense** — scheduler prompts treated with same skepticism as user messages

---

## Credits

Forked from Anthropic's official Claude Code Telegram plugin.
Copyright 2024-2026 Anthropic, PBC. Original licensed under Apache License 2.0.

See [NOTICE](NOTICE) for full attribution.

## License

Apache License 2.0 — see [LICENSE](LICENSE)
