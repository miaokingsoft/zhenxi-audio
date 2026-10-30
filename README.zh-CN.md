# 珍惜音频资源服务 · Agent Skill

[English](README.md) | **简体中文**

> 让 AI 智能体接入「珍惜音频」的 Agent Skill：1600+ 部评书 / 有声小说 / 广播剧，共 5.4 万集。
> 报书名就搜、先听第 1 集、喜欢再取整部打包下载。

技能本身不含任何音频内容，只负责调用远端 MCP 服务的工具。

## 能力一览

| 能力 | 说明 |
| --- | --- |
| 搜书 | 按书名、作者、演播者关键字检索书库 |
| 详情 | 集数、简介、整部获取条件 |
| 试听 | 每部书第 1 集在线试听直链 |
| 整部下载 | 签发 ZIP 直链，打包中可轮询获取 |
| 账户 | 账户状态、最近记录、已获取书目 |
| 规则查询 | 实时查询服务端当前的规则与参数 |

## 工作方式：MCP over Streamable HTTP

服务端是一个**远程无状态** MCP 服务器，令牌只通过请求头传递。

```
Endpoint : https://zhenxi.miaoking.com/mcp.php
Transport: Streamable HTTP (stateless)
Auth     : Authorization: Bearer <接入令牌>
```

> 令牌**仅支持**通过 Header 传递，不支持 URL 参数。任何把令牌拼进 URL 的做法都不被接受。

## 安装

### 方式一：作为技能安装（推荐）

把整个目录复制到你的技能目录：

```bash
# WorkBuddy
git clone <this-repo> ~/.workbuddy/skills/zhenxi-audio

# Claude Code / 兼容 Agent Skills 规范的客户端
git clone <this-repo> ~/.claude/skills/zhenxi-audio
```

重启客户端，此后提到「评书」「听书」「搜书名」「有声小说」「整部下载」等词时技能即会被触发。

### 方式二：只配 MCP，不用技能

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

## 取得接入令牌

两条路，优先第一条。

### 1. 配对码（免注册、免验证）

打开[服务网页](https://zhenxi.miaoking.com) 的「MCP服务」页，点「复制一句话」，把带 `PX-` 开头
配对码的整句话发给智能体，10 分钟内一次性有效。智能体收到后会调用 `auth_pair` 直接换回令牌。

### 2. 邮箱验证码

没有配对码时：

```
auth_send   { email: "you@example.com", intent: "register" }  # 发送验证码，返回 ticket
auth_verify { ticket, mail_code }                             # 完成注册 → member_id + token
```

已有账号在新设备上换令牌，把 `intent` 改成 `"login"` 即可取回同一枚令牌。

拿到令牌后写入 MCP 配置的 Header，**请自行保存好**——丢失可在网页令牌页重新生成。

## 工具列表

| 工具 | 用途 | 需鉴权 |
| --- | --- | --- |
| `auth_pair` | 配对码换令牌 | 否 |
| `auth_send` | 发送邮箱验证码 | 否 |
| `auth_verify` | 验证码换令牌（注册 / 登录） | 否 |
| `get_pricing` | 查询当前服务规则与参数 | 否 |
| `search_books` | 按关键字搜书 | 是 |
| `get_book_detail` | 书目详情 | 是 |
| `get_trial` | 第 1 集试听直链 | 是 |
| `get_balance` | 账户状态与最近记录 | 是 |
| `list_purchases` | 已获取书目 | 是 |
| `redeem_card` | 兑换访问码 | 是 |
| `get_book` | 获取整部（签发或触发打包） | 是 |
| `get_download_url` | 已获取书取 ZIP 直链 | 是 |

## 使用流程

```
搜书 → 看详情 → 试听第 1 集 → （用户想留全本）取整部 → 拿 ZIP 直链
```

试听交付有讲究：**不把原始签名长链贴进对话**，而是用随包的 `player-template.html` 生成一张
播放页，替换 `{{BOOK_TITLE}}` / `{{EPISODE_LABEL}}` / `{{AUDIO_URL}}` / `{{VALID_MINUTES}}` /
`{{BOOK_PAGE_URL}}` 五个占位符后整页交付。页脚的「非盈利存档 / 版权归属 / 联系删除」声明
必须原样保留。

## 使用规则

- **试听**：仅每部书第 1 集提供在线试听，第 2 集起没有在线播放入口，不要承诺。
- **交付**：只提供整部交付（无单集）；已获取的权益长期有效。
- **直链时效**：ZIP 直链限时有效（当前 1 小时），每部每日限制签发次数（当前 3 次，次日恢复）。
- **参数会变**：以上为当前配置，如有调整以 `get_pricing` / `get_book_detail` 的实时返回为准。

## 版权与免责

- 本站内容版权归**原演播者 / 出版方**所有，本站仅作存档与索引。
- 如内容侵犯您的权利，请联系 <kf@miaoking.com> 删除，我们会尽快处理。
- 本项目（技能包）只是调用接口的说明与流程编排，**不包含任何音频文件**。

## 目录结构

```
zhenxi-audio/
├── SKILL.md               # 技能主体：接入配置、使用节奏、话术约束
├── player-template.html   # 试听播放页模板（5 个占位符）
├── README.md              # English documentation
└── README.zh-CN.md        # 中文文档（本文件）
```

## 版本

当前 `1.2.0`（2026-10-31）。版本以 `SKILL.md` frontmatter 的 `version` 为准；本地副本版本更低时，
请整包覆盖更新。

## 许可

技能包文本与模板采用 **MIT** 许可；后端提供的音频内容不在本许可范围内，版权归各自权利人所有。
