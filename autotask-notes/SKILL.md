---
name: autotask-notes
description: 在用户的 AutoTask 个人笔记（个人工作区默认知识库）中搜索、读取全文、创建和更新 Markdown 笔记。当用户要求"记个笔记/保存这段内容/整理成文档"、询问"我的笔记里有没有关于 X 的记录"、要求基于既有笔记做总结或修订时使用；当用户要求接入/配置/修复 AutoTask MCP 连接时也使用本技能的接入章节。触发词包括：笔记、记录、知识库、文档、保存下来、查一下我记过的、接入 autotask。
---

# AutoTask 个人笔记

通过 AutoTask MCP 服务端（`autotask` MCP server）的 `*_personal_note*` 工具，检索和读写用户在 AutoTask 个人版「笔记」（个人工作区默认知识库）里的 Markdown 文档。所有工具由服务端强制限定在调用者本人的个人工作区。

## 接入自检与自助配置

本技能是给 Agent 自己用的：不依赖人工读文档，Agent 按本节即可完成接入与修复。

**自检**：当前会话能调到 `search_personal_notes` 即已接入。调不到（或调用报连接错误）时按下面步骤配置，配置完成后需重启客户端或新开会话再验证。

1. **拿 token**（二选一，优先 A）：
   - A. 请用户在 AutoTask 网页 「设置 → API 令牌」 创建一个 API token，勾选 scope：`mcp:access`、`kb:read`、`kb:write`。token 只展示一次，请用户提供。
   - B. 用户机器装有 AutoTask CLI（`autotask` 命令）时，代跑 `autotask login --server <server>` 完成交互登录，之后可用 `autotask config setup codex|claude-code` 一键写入（stdio 桥，免手工管理 token）。
2. **写 MCP 配置**（端点 `<server>/mcp/stream`；自部署实例把 `https://app.autotask.run` 换成实际地址）。**合并写入，不得覆盖用户已有配置**：
   - ZCode：`~/.zcode/cli/config.json` 的 `mcp.servers` 下加
     `"autotask": {"type": "http", "url": "https://app.autotask.run/mcp/stream", "headers": {"Authorization": "Bearer <token>"}}`（工作区级则放 `<repo>/.zcode/config.json`）。
   - Codex：`~/.codex/config.toml` 加 `[mcp_servers.autotask]`，`url = "https://app.autotask.run/mcp/stream"`、`bearer_token_env_var = "AUTOTASK_TOKEN"`，并请用户在 shell 配置里 `export AUTOTASK_TOKEN=<token>`。
   - Claude Code：`claude mcp add --transport http autotask https://app.autotask.run/mcp/stream --header "Authorization: Bearer <token>"`。
3. **版本自检**：`tools/list` 里若没有 `get_personal_note` / `list_personal_notes`（2026-10 前的服务端版本），按降级模式工作——只用 `search_personal_notes` 的命中片段回答，并明确告知用户"片段可能有截断，完整内容请在 AutoTask 笔记页查看"；绝不在未读全文的情况下调用 `update_personal_note`。
4. **验证失败排查**：检查 token scope（`kb:read`/`kb:write`）、端点 URL、客户端 MCP 面板报错，不要反复重试连接。

token 只写入用户本机配置/环境变量，不得提交进任何仓库或写进对话产物文件。

## 工具速查

| 工具 | 用途 | 关键参数 / 返回 |
| --- | --- | --- |
| `search_personal_notes` | 按关键词搜标题+内容 | `query`(必填)、`limit`(≤50)；返回 `document_id`、`title`、**摘要片段**（非全文） |
| `list_personal_notes` | 按最近更新浏览（无需关键词） | `limit`(默认 20，≤50)；返回 `document_id`、`title`、开头 120 字 `excerpt` |
| `get_personal_note` | 读单篇**完整** Markdown | `document_id`(必填)；返回 `title`、`content`、`content_type`、`updated_at` |
| `create_personal_note` | 新建笔记 | `title`(必填)、`content`(Markdown 正文，自包含)、`auto_index`(默认 true) |
| `update_personal_note` | 更新标题或正文 | `document_id`(必填)、`title`、`content`(**整体替换**)、`auto_index` |

## 核心工作流

### 1. 查询 / 问答
「我之前记过 XX 吗」「我的笔记里关于 Y 的内容」：
1. `search_personal_notes`（关键词具体一点，匹配是大小写不敏感的子串匹配）。
2. 命中后**必须 `get_personal_note` 拿全文再回答或引用**——搜索结果只有 ±百字左右的命中片段，直接引用片段容易断章取义。
3. 没有关键词的浏览场景（「最近记了什么」）用 `list_personal_notes`。

### 2. 保存对话内容
「把刚才的结论存成笔记」「记录一下这个方案」：
- `create_personal_note`，`content` 写成**自包含**的 Markdown：读者不看对话也能看懂（背景 + 结论 + 关键细节），不要写「如上所述」「刚才提到的第二点」这类指代。
- 标题具体化（「氢能平台调研结论」优于「调研」）。

### 3. 修订既有笔记
「把这条笔记补充一下」「把过期的段落改掉」：
1. `search_personal_notes` / `list_personal_notes` 定位 `document_id`。
2. `get_personal_note` 取全文。
3. `update_personal_note` 提交**完整的修改后正文**——`content` 是整体替换，只传新增段落会把其余内容全部抹掉。只改标题时不传 `content`。

## 红线

- **先 get 再 update**：跳过全文读取直接更新是数据丢失的首要原因。
- 搜索摘要 ≠ 全文：引用给用户前必须 `get_personal_note` 核实上下文。
- Org 格式文档（`content_type: "org"`，如 Org-mode 同步的笔记）只可读、不可通过 MCP 更新，更新会报错；如需修改请引导用户去 AutoTask 笔记编辑器。
- 用户没有保存意图的对话内容不要擅自写笔记；保存前一句话确认标题和范围即可，不必逐字审阅。
- 笔记属于用户个人数据：工具没有也无法指定他人工作区，不要尝试传工作区参数。
