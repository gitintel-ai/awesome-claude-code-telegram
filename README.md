# awesome-claude-code-telegram

Extended Telegram channel plugin for [Claude Code](https://claude.ai/claude-code). Adds inline keyboards, a built-in task scheduler, bot commands, document/voice handling, message formatting, and more — on top of the official plugin's foundation.

## Why

The official Claude Code Telegram plugin ships 3 tools: `reply`, `react`, `edit_message`. That covers basic messaging but leaves out a lot of what makes Telegram powerful — and what makes an AI co-founder actually useful.

This fork adds **11 new tools** and **3 new message handlers** while keeping full backward compatibility with the official plugin's access control, pairing, and security model.

## What's New

### Tools (14 total)

| Tool | What it does |
|------|-------------|
| `reply` | Send message with optional `parse_mode` (MarkdownV2/HTML) and file attachments |
| `react` | Add emoji reaction |
| `edit_message` | Edit bot's own message with optional `parse_mode` |
| **`send_keyboard`** | Send message with inline buttons (callbacks or URLs) |
| **`answer_callback`** | Acknowledge button taps with toast or alert |
| **`pin_message`** | Pin a message (silent option) |
| **`unpin_message`** | Unpin a message |
| **`set_commands`** | Register /commands in Telegram's menu |
| **`delete_message`** | Delete a message |
| **`forward_message`** | Forward between chats |
| **`schedule_task`** | Create recurring or one-shot scheduled tasks |
| **`schedule_delay`** | Schedule a task with human delay ("30m", "2h", "1d") |
| **`unschedule_task`** | Remove a scheduled task |
| **`update_task`** | Pause/resume/modify scheduled tasks |
| **`list_scheduled`** | List all tasks with status, runs, expiry |

### Message Handlers (3 new)

| Handler | What it does |
|---------|-------------|
| **Documents** | PDFs, files downloaded to inbox/ — Claude can read them |
| **Voice** | Voice messages saved as .ogg — path sent to Claude |
| **Callback queries** | Button taps routed to Claude as channel notifications |

### Built-in Scheduler

Cron-based task scheduler that lives inside the plugin. No separate process, no conflicts.

Features:
- Standard 5-field cron expressions
- Timezone support (defaults to Asia/Kolkata)
- Quiet hours (non-urgent tasks pause overnight)
- Task expiry (`expires_at` — auto-disable after a time)
- Max runs (`max_runs` — fire N times then stop)
- Tags for filtering and organization
- Urgent flag (bypasses quiet hours)
- One-shot and recurring tasks
- Human-readable delays ("in 30m", "in 2h")

**How it works:** The scheduler runs inside the plugin's event loop (same process as the Telegram bot). When a task is due, it sends an MCP notification to Claude. Claude processes it and replies via the `reply` tool. One process, zero conflicts.

```json
{
  "tasks": [
    {
      "id": "morning_brief",
      "cron": "57 8 * * *",
      "prompt": "Generate and deliver the morning brief.",
      "chat_id": "YOUR_CHAT_ID",
      "enabled": true,
      "recurring": true,
      "description": "Daily morning brief at ~9 AM"
    }
  ],
  "timezone": "Asia/Kolkata",
  "quiet_hours": { "start": 1, "end": 7 }
}
```

## Installation

### Prerequisites

- [Bun](https://bun.sh/) runtime
- A Telegram bot token from [@BotFather](https://t.me/BotFather)
- Claude Code with channel support

### Setup

1. Clone this repo:
```bash
git clone https://github.com/gitintel-ai/awesome-claude-code-telegram.git
cd awesome-claude-code-telegram
bun install
```

2. Save your bot token:
```bash
mkdir -p ~/.claude/channels/telegram
echo "TELEGRAM_BOT_TOKEN=your-token-here" > ~/.claude/channels/telegram/.env
chmod 600 ~/.claude/channels/telegram/.env
```

3. Create a schedule (optional):
```bash
cat > ~/.claude/channels/telegram/schedule.json << 'EOF'
{
  "tasks": [],
  "timezone": "Asia/Kolkata"
}
EOF
```

4. Register as an MCP server in your Claude Code settings or run directly:
```bash
bun server.ts
```

## Use Cases

| Use Case | How |
|----------|-----|
| Morning brief | `schedule_task` with cron `"57 8 * * *"` |
| Flight tracking | `schedule_task` with `"*/30 * * * *"`, `expires_at`, `urgent: true` |
| Reminders | `schedule_delay` with `"2h"` or `"30m"` |
| CI monitoring | `schedule_task` with `"*/5 * * * *"`, `max_runs: 30` |
| Approval flows | `send_keyboard` with Yes/No buttons + `answer_callback` |
| Formatted reports | `reply` with `parse_mode: "HTML"` |
| Weekly digest | `schedule_task` with cron `"3 9 * * 6"` |

## Architecture

```
Claude Code Session
    ↕ MCP (stdio)
awesome-claude-code-telegram (this plugin)
    ↕ Grammy (Telegram Bot API)
Telegram
```

Single process. The plugin is both the messaging bridge AND the scheduler. No separate bot process, no polling conflicts.

## Access Control

Inherits the official plugin's full access control:
- **Pairing mode** — unknown senders get a 6-char code, approved via CLI
- **Allowlist mode** — only pre-approved user IDs
- **Group support** — opt-in with @mention triggering
- **Security** — outbound gated by allowlist, state files blocked from sending

## Credits

Forked from [Anthropic's official Telegram plugin](https://claude.ai/claude-code) (Apache 2.0).

## License

MIT
