# The Official Claude Code Telegram Plugin Has 3 Tools. Here's What's Missing — and How We Fixed It.

*March 2026*

---

Claude Code's Telegram integration is one of the most underrated features in the AI coding space. You can message Claude from your phone and get code written, bugs fixed, repos explored — all from Telegram. No laptop required.

But the official plugin ships with exactly 3 tools: `reply`, `react`, `edit_message`. That's it.

No scheduled tasks. No inline keyboards. No document handling. No formatted messages. No bot commands. No way to turn Claude into an autonomous agent that does things without you asking.

We spent a week building everything that's missing. Here's what we found, what we built, and why it matters.

## What the Official Plugin Can't Do

We mapped every Telegram Bot API capability against what the official plugin exposes. The gaps are significant:

| Capability | Telegram Bot API | Official Plugin | Gap |
|-----------|:---:|:---:|:---:|
| Send text messages | Yes | Yes | — |
| React with emoji | Yes | Yes | — |
| Edit own messages | Yes | Yes | — |
| Handle photos | Yes | Yes | — |
| **MarkdownV2/HTML formatting** | Yes | **No** | Missing |
| **Inline keyboards (buttons)** | Yes | **No** | Missing |
| **Handle documents/PDFs** | Yes | **No** | Missing |
| **Handle voice messages** | Yes | **No** | Missing |
| **Bot commands menu** | Yes | **No** | Missing |
| **Pin/unpin messages** | Yes | **No** | Missing |
| **Delete messages** | Yes | **No** | Missing |
| **Forward messages** | Yes | **No** | Missing |
| **Scheduled/timed messages** | No (API limitation) | **No** | Needs custom scheduler |
| **Callback queries (button taps)** | Yes | **No** | Missing |

13 capabilities missing. The plugin covers basic chat and nothing else.

## Why This Matters

The 3-tool limitation isn't just a feature gap — it blocks entire categories of use:

### You can't build approval workflows

Without inline keyboards, every approval is a text conversation:

```
Claude: "Should I deploy v0.1.1?"
You: "yes"
Claude: [deploys]
```

With inline keyboards:

```
Claude: [message with ✅ Deploy / ❌ Cancel buttons]
You: [taps ✅]
Claude: [deploys immediately]
```

The button version is faster, unambiguous, and works when you're on your phone with one hand.

### You can't schedule anything

Claude Code is reactive — it responds when you message it. But a real AI assistant should be proactive:

- Morning brief at 9 AM without you asking
- Flight status checks every 30 minutes before departure
- CI pipeline monitoring after a deploy
- Weekly digest of industry news

The official plugin has no scheduler. Every interaction requires you to initiate.

### You can't share files

Send Claude a PDF on Telegram? It sees "(document)" as text. The file is ignored. Send a voice message? Same thing — the audio is dropped.

For a tool that's supposed to be your AI co-founder, not being able to read documents you send is a fundamental gap.

### You can't format responses

Claude's replies arrive as plain text. No bold headers, no code blocks, no structured formatting. A morning brief that could look like this:

```html
<b>Morning Brief — March 22</b>

▸ GitIntel: v0.1.0 live, 8 downloads
▸ CI: all green
▸ <code>npm audit</code>: 0 vulnerabilities
```

Instead looks like this:

```
Morning Brief — March 22

▸ GitIntel: v0.1.0 live, 8 downloads
▸ CI: all green
▸ npm audit: 0 vulnerabilities
```

Small difference visually. Big difference in scanability when you're reading on a phone.

## What We Built

We forked the official plugin and added 11 new tools and 3 new message handlers. The entire fork is a single file — 1,275 lines of TypeScript, same two dependencies (Grammy + MCP SDK).

### The Scheduler

The most impactful addition. A cron-based task scheduler that runs inside the plugin process itself.

**Why inside the plugin?** Because running a separate bot process alongside the official plugin causes a `409 Conflict` error — Telegram only allows one process to poll a bot token at a time. By embedding the scheduler in the plugin, we get zero conflicts and zero extra infrastructure.

The scheduler reads a `schedule.json` file:

```json
{
  "tasks": [
    {
      "id": "morning_brief",
      "cron": "57 8 * * *",
      "prompt": "Generate the morning brief.",
      "chat_id": "123456789",
      "recurring": true
    }
  ],
  "timezone": "Asia/Kolkata",
  "quiet_hours": { "start": 1, "end": 7 }
}
```

When a task is due, the scheduler sends an MCP notification to Claude. Claude processes it like any inbound message and replies via Telegram. The CEO gets a morning brief at 9 AM without lifting a finger.

Features we added beyond basic cron:
- **Quiet hours** — non-urgent tasks don't fire between 1-7 AM
- **Task expiry** — "check flight status until 8 PM, then stop"
- **Max runs** — "check CI 30 times (every 5 min for 2.5 hours), then stop"
- **Tags** — filter tasks by category
- **Urgent flag** — bypass quiet hours for critical alerts
- **Human delays** — `schedule_delay("2h")` instead of computing cron

### Inline Keyboards

Two new tools: `send_keyboard` and `answer_callback`.

`send_keyboard` sends a message with buttons. Each button has a label and a callback data string. When the user taps a button, the callback data arrives as a channel notification with `callback_query: true` in the metadata.

Claude receives the callback, processes it, and can take action — deploy, approve, cancel, whatever the button represents.

Security note: we track received callback IDs in memory and only allow `answer_callback` for queries the plugin actually received. An attacker can't craft a fake callback_query_id.

### Document and Voice Handling

Documents (PDFs, spreadsheets, any file) and voice messages are now downloaded to an inbox directory when received. The file path is sent to Claude via the MCP notification metadata — same pattern the official plugin uses for photos.

Filenames from Telegram are sanitized (path separators and `..` stripped) to prevent directory traversal. The inbox directory is created with `0o700` permissions.

### Message Formatting

The `reply` and `edit_message` tools now accept an optional `parse_mode` parameter: `"MarkdownV2"` or `"HTML"`. This lets Claude send formatted messages with bold text, code blocks, links, and structured layouts.

## Security Audit

We ran a full security audit before releasing. Key findings and fixes:

| Issue | Severity | Status |
|-------|----------|--------|
| Document filename path traversal | HIGH | Fixed — filenames sanitized |
| `answer_callback` bypassed allowlist | HIGH | Fixed — callback ID tracking |
| Scheduler `chat_id` not validated | HIGH | Fixed — `assertAllowedChat` added |
| `schedule.json` written without `chmod` | LOW | Fixed — `0o600` mode |
| Inbox created without restrictive perms | LOW | Fixed — `0o700` mode |
| Apache 2.0 license compliance | CRITICAL (legal) | Fixed — proper NOTICE + LICENSE |

The full audit covered path traversal, command injection, SSRF, access control bypass, race conditions, and license compliance.

## The Numbers

| Metric | Official Plugin | This Plugin |
|--------|:-:|:-:|
| Tools | 3 | 14 |
| Message handlers | 2 | 5 |
| Lines of code | 602 | 1,275 |
| Dependencies | 2 | 2 (same) |
| Scheduling | None | Full cron engine |
| Button support | None | Inline keyboards |
| File handling | Photos only | Photos + docs + voice |

## Try It

```bash
git clone https://github.com/gitintel-ai/awesome-claude-code-telegram.git
cd awesome-claude-code-telegram
bun install

# Set your bot token
mkdir -p ~/.claude/channels/telegram
echo "TELEGRAM_BOT_TOKEN=your-token" > ~/.claude/channels/telegram/.env
chmod 600 ~/.claude/channels/telegram/.env

# Run
bun server.ts
```

Register as an MCP server in `~/.claude/mcp.json`:

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

The plugin is backward-compatible with the official one — same access control, same pairing, same security model. It just does more.

## What's Next

- Demo GIF showing inline keyboards in action
- More message handlers (stickers, location, contacts)
- Inbox cleanup with TTL-based eviction
- Windows token security via NTFS ACLs
- Upstream contribution to Anthropic's official plugin

The repo is at [github.com/gitintel-ai/awesome-claude-code-telegram](https://github.com/gitintel-ai/awesome-claude-code-telegram). Apache 2.0 licensed, forked from Anthropic's official plugin.

---

*Built by [gitintel-ai](https://github.com/gitintel-ai). We also make [GitIntel](https://github.com/gitintel-ai/GitIntelAI) — a CLI that tracks AI-generated code in git repos.*
