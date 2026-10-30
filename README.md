# 珍惜音频资源服务 · Agent Skill

> 一个让 AI 智能体接入「珍惜音频」的 Agent Skill：1600+ 部评书 / 有声小说 / 广播剧，共 5.4 万集。
> 报书名就搜、想听先听第 1 集、喜欢再整部打包下载。
>
> An Agent Skill that connects AI assistants to **Zhenxi Audio** — 1,600+ Chinese audiobooks
> (storytelling, audio novels, radio dramas) across 54,000+ episodes. Search by title, listen to
> episode 1 for free, then download whole books as ZIP archives.

[中文](#中文) · [English](#english)

---

## 中文

### 这是什么

`zhenxi-audio` 是一个遵循 Agent Skills 规范（`SKILL.md` + 可选资源文件）的技能包。装上它之后，
你的 AI 助手就学会了如何通过 MCP 工具去「珍惜音频资源服务」里找书、试听、取整部，并且知道
**该先让用户体验内容、再谈价格**，不会一上来就推销。

技能本身不含任何音频内容，只负责调用远端 MCP 服务的工具。

### 能力一览

| 能力 | 说明 |
| --- | --- |
| 搜书 | 按书名、作者、演播者关键字检索书库 |
| 详情 | 集数、简介、整部价与封顶抵扣明细 |
| 试听 | 每部书第 1 集在线试听直链（首听扣 1 积分，重复免费） |
| 整部下载 | 扣整部积分后签发 ZIP 直链，打包中可轮询获取 |
| 账户 | 余额、最近流水、已获取书目 |
| 积分 | 兑换码即时入账；规则与汇率实时查询 |

### 工作方式：MCP over Streamable HTTP

服务端是一个**远程无状态** MCP 服务器，令牌只通过请求头传递。

```
Endpoint : https://zhenxi.miaoking.com/mcp.php
Transport: Streamable HTTP (stateless)
Auth     : Authorization: Bearer <接入令牌>
```

> 令牌**仅支持**通过 Header 传递，不支持 URL 参数。任何把令牌拼进 URL 的做法都不被接受。

### 安装

**方式一：作为技能安装（推荐）**

把整个目录复制到你的技能目录：

```bash
# WorkBuddy
git clone <this-repo> ~/.workbuddy/skills/zhenxi-audio

# Claude Code / 兼容 Agent Skills 规范的客户端
git clone <this-repo> ~/.claude/skills/zhenxi-audio
```

重启客户端，此后提到「评书」「听书」「搜书名」「有声小说」「整部下载」等词时技能即会被触发。

**方式二：只配 MCP，不用技能**

在 MCP 配置里加一条即可（Claude Desktop / Cursor / 其他支持 Streamable HTTP 的客户端同理）：

```json
{
  "mcpServers": {
    "zhenxi-audio": {
      "type": "http",
      "url": "https://zhenxi.miaoking.com/mcp.php",
      "headers": {
        "Authorization": "Bearer <你的接入令牌>"
      }
    }
  }
}
```

### 取得接入令牌

两条路，优先第一条：

**1. 配对码（免注册、免验证）** —— 打开 [服务网页](https://zhenxi.miaoking.com) 的「MCP服务」页，
点「复制一句话」，把带 `PX-` 开头配对码的整句话发给智能体，10 分钟内一次性有效。
智能体收到后会调用 `auth_pair` 直接换回令牌。

**2. 邮箱验证码** —— 没有配对码时：

```
auth_send   { email: "you@example.com", intent: "register" }  # 发送验证码，返回 ticket
auth_verify { ticket, mail_code }                             # 完成注册 → member_id + token
```

老会员在新设备上换令牌，把 `intent` 改成 `"login"` 即可取回同一枚令牌。

拿到令牌后写入 MCP 配置的 Header，**请自行保存好**——丢失可在网页令牌页重新生成。

### 工具列表

| 工具 | 用途 | 需鉴权 |
| --- | --- | --- |
| `auth_pair` | 配对码换令牌 | 否 |
| `auth_send` | 发送邮箱验证码 | 否 |
| `auth_verify` | 验证码换令牌（注册 / 登录） | 否 |
| `get_pricing` | 实时计费规则与汇率 | 否 |
| `search_books` | 按关键字搜书 | 是 |
| `get_book_detail` | 书目详情与整部价 | 是 |
| `get_trial` | 第 1 集试听直链 | 是 |
| `get_balance` | 余额 + 最近 10 条流水 | 是 |
| `list_purchases` | 已获取书目 | 是 |
| `redeem_card` | 兑换码入账 | 是 |
| `get_book` | 获取整部（扣分 / 触发打包） | 是 |
| `get_download_url` | 已获取书取 ZIP 直链 | 是 |

### 使用流程

```
搜书 → 看详情 → 试听第 1 集 → （用户想收藏）获取整部 → 拿 ZIP 直链
```

试听交付有讲究：**不把原始签名长链贴进对话**，而是用随包的 `player-template.html` 生成一张
播放页，替换 `{{BOOK_TITLE}}` / `{{EPISODE_LABEL}}` / `{{AUDIO_URL}}` / `{{VALID_MINUTES}}` /
`{{BOOK_PAGE_URL}}` 五个占位符后整页交付。页脚的「非盈利存档 / 版权归属 / 联系删除」声明
必须原样保留。

### 计费与规则（以工具返回为准）

以下是**当前配置快照**，服务端可能随时调整。对外报出的任何价格、汇率、限额，都以
`get_pricing` / `get_book_detail` / `search_books` 的实时返回为准。

- **计价**：整部价 = 每集单价累加，单部封顶（当前 每集 20 积分、单部 2000 封顶）
- **汇率**：捐助到账按积分/元兑换（当前 100 积分 = 1 元）；注册赠送积分（当前 100）
- **试听**：仅每部书第 1 集可在线试听，首听扣 1 积分，重复免费
- **交付**：只提供整部交付（无单集）；已获取权益永久有效，ZIP 直链限时有效（当前 1 小时），
  每部每日限次签发（当前 3 次，次日恢复）
- **积分**：封闭体系，不可提现、不可转赠、不设退款

### 关于捐助

本站是**非盈利存档站点**。5.4 万集音频的云存储与带宽开销相当可观，捐助全部用于补贴这些成本，
让内容能长期保存下去。技能被明确要求：先算清差额再建议金额，不引导超额捐助，余额够用时不得
引导捐助。捐助页：<https://j.wwzu.com/p/psvip>（10 元起，自定义金额，付款后发放兑换码）。

### 版权与免责

- 本站内容版权归**原演播者 / 出版方**所有，本站仅作存档与索引。
- 如内容侵犯您的权利，请联系 <kf@miaoking.com> 删除，我们会尽快处理。
- 本项目（技能包）只是调用接口的说明与流程编排，**不包含任何音频文件**。

### 目录结构

```
zhenxi-audio/
├── SKILL.md               # 技能主体：接入配置、使用节奏、计费口径、话术约束
├── player-template.html   # 试听播放页模板（5 个占位符）
└── README.md              # 本文件
```

### 版本

当前 `1.2.0`（2026-10-31）。版本以 `SKILL.md` frontmatter 的 `version` 为准；本地副本版本更低时，
请整包覆盖更新。

---

## English

### What this is

`zhenxi-audio` is a skill package following the Agent Skills convention (`SKILL.md` plus optional
resources). Once installed, your AI assistant knows how to search, preview, and retrieve audiobooks
from the **Zhenxi Audio** service through MCP tools — and, just as importantly, knows to let you
*experience the content first* before mentioning price.

The skill ships **no audio at all**. It only orchestrates calls to a remote MCP server.

### Capabilities

| Capability | Description |
| --- | --- |
| Search | Query the library by title, author, or narrator |
| Details | Episode count, synopsis, full-book price with cap breakdown |
| Preview | Signed streaming URL for episode 1 (1 credit on first play, free afterwards) |
| Full download | Deducts credits and issues a ZIP link; poll while packaging |
| Account | Balance, recent ledger, purchased titles |
| Credits | Instant redemption codes; live pricing lookup |

### How it works — MCP over Streamable HTTP

The server is a **remote, stateless** MCP endpoint. Tokens travel in the request header only.

```
Endpoint : https://zhenxi.miaoking.com/mcp.php
Transport: Streamable HTTP (stateless)
Auth     : Authorization: Bearer <access-token>
```

> Tokens are accepted **only** via the `Authorization` header. Passing a token as a URL parameter
> is not supported.

### Install

**Option 1 — install as a skill (recommended)**

```bash
# WorkBuddy
git clone <this-repo> ~/.workbuddy/skills/zhenxi-audio

# Claude Code or any Agent Skills-compatible client
git clone <this-repo> ~/.claude/skills/zhenxi-audio
```

Restart your client. The skill activates on phrases like "audiobook", "storytelling", "search a
title", or "download the whole book".

**Option 2 — MCP configuration only**

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

### Getting an access token

**1. Pairing code (no signup, no email)** — open the [service site](https://zhenxi.miaoking.com),
go to the "MCP Service" page, click "Copy a sentence", and send the whole sentence (it contains a
`PX-` prefixed pairing code) to your assistant. Valid for 10 minutes, single use. The assistant
calls `auth_pair` to exchange it for a token.

**2. Email verification code** — if you have no pairing code:

```
auth_send   { email: "you@example.com", intent: "register" }  # sends code, returns ticket
auth_verify { ticket, mail_code }                             # → member_id + token
```

Existing members on a new device: switch `intent` to `"login"` to recover the same token.

Store the token in your MCP config header. Keep it safe — it can be regenerated from the token page
on the website.

### Tools

| Tool | Purpose | Auth |
| --- | --- | --- |
| `auth_pair` | Exchange pairing code for token | No |
| `auth_send` | Send email verification code | No |
| `auth_verify` | Exchange code for token (register / login) | No |
| `get_pricing` | Live pricing rules and exchange rate | No |
| `search_books` | Keyword search | Yes |
| `get_book_detail` | Book details and full-book price | Yes |
| `get_trial` | Episode 1 preview URL | Yes |
| `get_balance` | Balance + last 10 ledger entries | Yes |
| `list_purchases` | Purchased titles | Yes |
| `redeem_card` | Redeem a credit code | Yes |
| `get_book` | Purchase full book (deduct / start packaging) | Yes |
| `get_download_url` | Get ZIP link for an owned book | Yes |

### Flow

```
search → details → preview episode 1 → (if you want to keep it) purchase → ZIP link
```

Preview delivery is deliberate: the raw signed URL is **never pasted into chat**. Instead the bundled
`player-template.html` is rendered into a player page by substituting `{{BOOK_TITLE}}`,
`{{EPISODE_LABEL}}`, `{{AUDIO_URL}}`, `{{VALID_MINUTES}}`, and `{{BOOK_PAGE_URL}}`. The footer
notice about non-profit archiving, copyright ownership, and takedown contact must stay intact.

### Pricing & rules (always defer to tool output)

The figures below are a **snapshot of the current configuration** and may change at any time.
Any price, exchange rate, or limit quoted to a user must come from the live response of
`get_pricing` / `get_book_detail` / `search_books`.

- **Pricing**: full-book price = per-episode price × episodes, capped per book (currently 20 credits
  per episode, 2,000 per book)
- **Rate**: donations convert at credits-per-yuan (currently 100 credits = ¥1); signup bonus credits
  (currently 100)
- **Preview**: episode 1 only; first play costs 1 credit, repeats are free
- **Delivery**: whole-book only (no single episodes); entitlements never expire; ZIP links are
  time-limited (currently 1 hour) and rate-limited per book per day (currently 3, resets daily)
- **Credits**: closed system — no cash-out, no transfers, no refunds

### About donations

This is a **non-profit archive**. Hosting and bandwidth for 54,000+ episodes is expensive, and
donations go entirely toward covering those costs so the archive can stay online. The skill is
explicitly instructed to compute the shortfall first, suggest the minimum sufficient amount, never
push overspending, and never solicit donations when the balance already covers the book.
Donation page: <https://j.wwzu.com/p/psvip> (from ¥10, custom amount, code issued after payment).

### Copyright & disclaimer

- All content is copyright of the **original narrators / publishers**; this service archives and
  indexes only.
- For takedown requests, contact <kf@miaoking.com>.
- This repository contains **no audio files** — only the integration spec and workflow logic.

### Repository layout

```
zhenxi-audio/
├── SKILL.md               # Core skill: setup, usage rhythm, pricing policy, tone rules
├── player-template.html   # Preview player template (5 placeholders)
└── README.md              # This file
```

### Version

`1.2.0` (2026-10-31). The authoritative version is the `version` field in `SKILL.md` frontmatter.
If your local copy is older, overwrite the whole package.

---

## License / 许可

Skill package text and template: **MIT**. Audio content served by the backend is **not** covered by
this license and remains with its respective rights holders.

技能包文本与模板采用 **MIT** 许可；后端提供的音频内容不在本许可范围内，版权归各自权利人所有。
