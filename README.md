# Extremis MCP server

Connect Claude, ChatGPT, Claude Code, Codex, Cursor or any MCP client to your [Extremis](https://extremis.online) inbox.

Extremis is a shared team inbox for WhatsApp, Instagram, Messenger and email, built on the official WhatsApp Cloud API. Its MCP server lets your AI tool read your conversations, contacts and reports and, if your workspace allows it, reply to customers and organise chats for you.

```
https://extremis.online/api/mcp
```

- **Hosted, remote MCP server** (Streamable HTTP). Nothing to install or run.
- **OAuth 2.1 sign-in** in the browser. No API keys to copy.
- **Acts as you.** A connection sees exactly what your role lets you see in Extremis.
- **Read-only by default.** Replying and organising chats need a workspace switch plus your approval.
- **Official WhatsApp Cloud API.** No unofficial WhatsApp Web bridge, which breaks WhatsApp's terms and puts numbers at risk of a ban.

> This repository holds documentation and the MCP Registry entry (`server.json`). The server itself runs inside Extremis and is not open source.

## Before you start

You need an **Extremis workspace**. AI connections are included in every plan at no extra cost.

An **Owner or Admin** turns them on once:

1. Open **Settings → AI connections** (under **Integrations**).
2. Turn on **Allow AI connections in this workspace**.

Nothing can connect until this switch is on. Turning it off stops every connected tool at once.

## Connect your AI tool

| Tool | How to add it |
|---|---|
| **Claude Code** | `claude mcp add --transport http extremis https://extremis.online/api/mcp`, then run `/mcp` and choose **extremis** to sign in. |
| **Codex** | Add the block below to `~/.codex/config.toml`, then run `codex mcp login extremis`. |
| **Claude (claude.ai and the desktop app)** | Add a **custom connector** and paste the server URL. |
| **ChatGPT** | Add a custom **MCP connector** (developer mode) and paste the server URL. |
| **Cursor, VS Code and other MCP clients** | Add a remote (HTTP) MCP server with the same URL. See the snippet below. |

Codex:

```toml
[mcp_servers.extremis]
url = "https://extremis.online/api/mcp"
```

Cursor (`.cursor/mcp.json`) and other clients that use an `mcpServers` file:

```json
{
  "mcpServers": {
    "extremis": {
      "url": "https://extremis.online/api/mcp"
    }
  }
}
```

VS Code (`.vscode/mcp.json`):

```json
{
  "servers": {
    "extremis": {
      "type": "http",
      "url": "https://extremis.online/api/mcp"
    }
  }
}
```

Your tool then opens an Extremis page in the browser. Sign in, check the tool's name and where you'll be sent afterwards, choose the workspace, leave ticked what the tool may do, and press **Allow**.

## Tools

A tool only appears when your role and the connection's permissions allow it.

**Reading** (always available)

| Tool | What it does |
|---|---|
| `whoami` | Your workspace, role and what the connection may do |
| `list_conversations` | Chats you can see, newest first, or by longest waiting or WhatsApp window closing soonest |
| `get_conversation` | One chat with its messages and notes |
| `search_messages` | Search the text of messages you can see |
| `list_contacts` / `get_contact` | Contacts with their channels, tags, custom fields and notes |
| `get_report` | Volume, first response and resolution times and the agent table for a period (Owners and Admins) |
| `get_live_overview` | Open, pending, unassigned and waiting right now (Owners and Admins) |
| `list_teammates`, `list_labels`, `list_quick_replies`, `list_templates`, `list_whatsapp_numbers`, `list_contact_fields` | The lists a tool needs to act correctly |

**Acting** (only when an Owner or Admin turns on **Allow AI tools to act**, and you tick the permission when connecting)

| Permission | Tools |
|---|---|
| Reply to customers as you | `send_reply`, `send_template`, `add_note`, `react_to_message` |
| Organise conversations | `assign_conversation`, `set_conversation_status`, `snooze_conversation`, `unsnooze_conversation`, `set_conversation_labels` |
| Edit contacts | `update_contact`, `set_contact_tags`, `add_contact_note`, `set_contact_fields` |
| Start new WhatsApp conversations | `start_conversation` (approved templates only) |

There are no tools to delete messages, contacts or conversations, export contacts in bulk, change settings or members, or send files.

## Things to ask

- "Which customers are waiting longest for a reply?"
- "Summarise yesterday's WhatsApp chats."
- "Make me this week's first response time report per agent."
- "Which WhatsApp customers have waited more than an hour? Draft a reply for each and show me before sending."
- "Close every pending chat with no customer message in the last 7 days."

## Safety

- **Same limits as the inbox.** A tool sees and changes only what your role and chat access allow. On WhatsApp, free text goes out only within 24 hours of the customer's last message; after that the tool must use an approved template.
- **Rate limits.** A connection can make up to 20 changes a minute.
- **Full log.** Settings → AI connections lists every action AI tools took over the last 30 days. Any connection can be revoked there at once.
- **Customer messages are data.** Extremis tells the tool to treat message text as information, never as instructions. Still read what your tool is about to send.
- **Keep a person in the loop.** Connected tools are for work you supervise. For automatic replies around the clock, use the built-in [AI Agent](https://extremis.online/help/ai-agent/what-is-the-ai-agent).

Message text, notes and contact details you ask about are sent to the AI tool you connect, and its provider handles them under its own terms. See our [Privacy policy](https://extremis.online/privacy) and [Data Processing Agreement](https://extremis.online/dpa).

## Discovery

| | |
|---|---|
| MCP endpoint | `https://extremis.online/api/mcp` |
| Server card | [`/.well-known/mcp/server-card.json`](https://extremis.online/.well-known/mcp/server-card.json) |
| Protected resource metadata | [`/.well-known/oauth-protected-resource/api/mcp`](https://extremis.online/.well-known/oauth-protected-resource/api/mcp) |
| MCP Registry name | `online.extremis/extremis` |

## Help and support

- [Connect AI tools with MCP](https://extremis.online/help/ai-tools/connect-ai-tools-with-mcp)
- [Let AI tools reply and make changes](https://extremis.online/help/ai-tools/allow-ai-tools-to-act)
- [All MCP tools](https://extremis.online/help/ai-tools/extremis-mcp-tools)
- [Manage and troubleshoot AI connections](https://extremis.online/help/ai-tools/manage-ai-connections)
- Email: support@extremis.online

Security issues: see [SECURITY.md](SECURITY.md).
