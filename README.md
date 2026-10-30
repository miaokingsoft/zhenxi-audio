# Zhenxi Audio · Agent Skill

**English** | [简体中文](README.zh-CN.md)

> An Agent Skill that connects AI assistants to **Zhenxi Audio** — 1,600+ Chinese audiobooks
> (storytelling, audio novels, radio dramas) across 54,000+ episodes. Search by title, preview
> episode 1, then pull the whole book as a ZIP archive.

The skill ships **no audio at all**. It only orchestrates calls to a remote MCP server.

## Capabilities

| Capability | Description |
| --- | --- |
| Search | Query the library by title, author, or narrator |
| Details | Episode count, synopsis, requirements for the full book |
| Preview | Signed streaming URL for episode 1 |
| Full download | Issues a ZIP link; poll while packaging |
| Account | Account state, recent records, owned titles |
| Rules lookup | Query the server's current rules and parameters |

## How it works — MCP over Streamable HTTP

The server is a **remote, stateless** MCP endpoint. Tokens travel in the request header only.

```
Endpoint : https://zhenxi.miaoking.com/mcp.php
Transport: Streamable HTTP (stateless)
Auth     : Authorization: Bearer <access-token>
```

> Tokens are accepted **only** via the `Authorization` header. Passing a token as a URL parameter
> is not supported.

## Install

### Option 1 — install as a skill (recommended)

```bash
# WorkBuddy
git clone <this-repo> ~/.workbuddy/skills/zhenxi-audio

# Claude Code or any Agent Skills-compatible client
git clone <this-repo> ~/.claude/skills/zhenxi-audio
```

Restart your client. The skill activates on phrases like "audiobook", "storytelling", "search a
title", or "download the whole book".

### Option 2 — MCP configuration only

```json
{
  "mcpServers": {
    "zhenxi-audio": {
      "type": "http",
      "url": "https://zhenxi.miaoking.com/mcp.php",
      "headers": {
        "Authorization": "Bearer <your-access-token>"
      }
    }
  }
}
```

## Getting an access token

### 1. Pairing code (no signup, no email)

Open the [service site](https://zhenxi.miaoking.com), go to the "MCP Service" page, click "Copy a
sentence", and send the whole sentence (it contains a `PX-` prefixed pairing code) to your assistant.
Valid for 10 minutes, single use. The assistant calls `auth_pair` to exchange it for a token.

### 2. Email verification code

If you have no pairing code:

```
auth_send   { email: "you@example.com", intent: "register" }  # sends code, returns ticket
auth_verify { ticket, mail_code }                             # → member_id + token
```

Existing accounts on a new device: switch `intent` to `"login"` to recover the same token.

Store the token in your MCP config header. Keep it safe — it can be regenerated from the token page
on the website.

## Tools

| Tool | Purpose | Auth |
| --- | --- | --- |
| `auth_pair` | Exchange pairing code for token | No |
| `auth_send` | Send email verification code | No |
| `auth_verify` | Exchange code for token (register / login) | No |
| `get_pricing` | Query current service rules and parameters | No |
| `search_books` | Keyword search | Yes |
| `get_book_detail` | Book details | Yes |
| `get_trial` | Episode 1 preview URL | Yes |
| `get_balance` | Account state and recent records | Yes |
| `list_purchases` | Owned titles | Yes |
| `redeem_card` | Redeem an access code | Yes |
| `get_book` | Retain a full book (issue or start packaging) | Yes |
| `get_download_url` | Get ZIP link for an owned book | Yes |

## Flow

```
search → details → preview episode 1 → (if you want the whole book) retain it → ZIP link
```

Preview delivery is deliberate: the raw signed URL is **never pasted into chat**. Instead the bundled
`player-template.html` is rendered into a player page by substituting `{{BOOK_TITLE}}`,
`{{EPISODE_LABEL}}`, `{{AUDIO_URL}}`, `{{VALID_MINUTES}}`, and `{{BOOK_PAGE_URL}}`. The footer
notice about non-profit archiving, copyright ownership, and takedown contact must stay intact.

## Usage rules

- **Preview**: episode 1 only; there is no in-browser playback for episode 2 onward — don't promise it.
- **Delivery**: whole book only (no single episodes); retained entitlements do not expire.
- **Link lifetime**: ZIP links are time-limited (currently 1 hour) and issuance is rate-limited
  per book per day (currently 3, resets daily).
- **Parameters may change**: the values above reflect the current configuration; always defer to the
  live response of `get_pricing` / `get_book_detail`.

## Copyright & disclaimer

- All content is copyright of the **original narrators / publishers**; this service archives and
  indexes only.
- For takedown requests, contact <kf@miaoking.com>.
- This repository contains **no audio files** — only the integration spec and workflow logic.

## Repository layout

```
zhenxi-audio/
├── SKILL.md               # Core skill: setup, usage rhythm, tone rules
├── player-template.html   # Preview player template (5 placeholders)
├── README.md              # English documentation (this file)
└── README.zh-CN.md        # Chinese documentation
```

## Version

`1.2.0` (2026-10-31). The authoritative version is the `version` field in `SKILL.md` frontmatter.
If your local copy is older, overwrite the whole package.

## License

Skill package text and template: **MIT**. Audio content served by the backend is **not** covered by
this license and remains with its respective rights holders.
