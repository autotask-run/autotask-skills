# AutoTask 外部 Skill（个人待办 / 个人笔记）

面向外部 CLI Agent（Codex、ZCode、Claude Code 等）的两个技能包，让它们通过 AutoTask MCP 服务端直接读写用户在 AutoTask 个人版的**待办**与**笔记**：

| 目录 | Skill | 覆盖的 MCP 工具 |
| --- | --- | --- |
| `autotask-todo/` | 个人待办管理 | `list_personal_todos`、`create_personal_todo`、`update_personal_todo`、`delete_personal_todo`、`list_personal_todo_lists`、`batch_update_personal_todos`、`personal_todo_review` |
| `autotask-notes/` | 个人笔记读写 | `search_personal_notes`、`list_personal_notes`、`get_personal_note`、`create_personal_note`、`update_personal_note` |

服务端强制把所有工具限定在调用者本人的个人工作区（个人待办 / 默认笔记知识库），token 泄露也只会暴露该用户自己的数据。

## 0. 给外部 Agent 的自助接入（推荐分发方式）

两个 SKILL.md 都是**写给 Agent 自己看的**：接入自检、token 获取、各客户端 MCP 配置写法、失败排查全部内嵌在技能正文里，Agent 拿到技能包后不需要人工文档即可完成接入和修复。给用户的最短上手流程：

1. 在 AutoTask 网页 「设置 → API 令牌」创建一个 API token（scope 见下文「前置条件」）。
2. 把本仓库的 git URL（见「发布」）和 token 交给你的 Agent（zcode / codex / claude-code），说一句"按这个技能仓库接入 AutoTask 待办/笔记"。
3. Agent 会自行：克隆仓库 → 把技能目录装进自己的 skills 目录 → 写 MCP 配置（合并写入，不覆盖已有配置）→ 重启验证。

## 1. 前置条件

1. AutoTask 账号（线上 `https://app.autotask.run`，自部署则用你的 server 地址）。
2. **API token**：Web 端 「设置 → API 令牌」创建（`POST /api/v1/ln`）。所需权限 scope：
   - 公共：`mcp:access`
   - 待办：`autotask:read` + `todo:write`
   - 笔记：`kb:read` + `kb:write`
   - 只装其中一个 skill 时可以只给对应 scope。
3. **版本注意**：`list_personal_notes` / `get_personal_note` 需要包含 2026-10 personal-notes MCP 改动的服务端版本；旧版本服务端上笔记技能只有 `search/create/update` 三个工具可用（技能正文已写明降级行为）。

## 2. MCP 接入

MCP 端点为 `<server>/mcp/stream`（Streamable HTTP，认证 `Authorization: Bearer <API token>`；也支持 `GET /mcp/sse`）。任选一条路径：

### 方式 A：官方 CLI 桥（Codex / Claude Code，token 自动管理）

```bash
autotask login --server https://app.autotask.run
autotask config setup codex        # 或 claude-code / all
```

未设置 `AUTOTASK_TOKEN` 时写入的是 stdio 桥（`autotask mcp serve`），CLI 用本地 `auth.json` 自动换发 API token，无需手工管理。

### 方式 B：远程 HTTP 直连（所有客户端通用；ZCode 走这条）

**Codex**（`~/.codex/config.toml`，设置 `AUTOTASK_TOKEN` 环境变量后 `autotask config setup codex` 会自动生成，也可手写）：

```toml
[mcp_servers.autotask]
url = "https://app.autotask.run/mcp/stream"
bearer_token_env_var = "AUTOTASK_TOKEN"
```

**ZCode**（`~/.zcode/cli/config.json`，也可以放工作区 `<repo>/.zcode/config.json`）：

```json
{
  "mcp": {
    "servers": {
      "autotask": {
        "type": "http",
        "url": "https://app.autotask.run/mcp/stream",
        "headers": { "Authorization": "Bearer <你的 API token>" }
      }
    }
  }
}
```

**Claude Code**：

```bash
claude mcp add --transport http autotask https://app.autotask.run/mcp/stream \
  --header "Authorization: Bearer <你的 API token>"
```

> token 只保存在客户端本地配置/环境变量里，不要提交进任何仓库。

## 3. 安装 Skill

各 CLI 的用户级技能目录：Codex `~/.codex/skills/`、ZCode `~/.zcode/skills/`（跨工具共享可用 `~/.agents/skills/`）、Claude Code `~/.claude/skills/`。技能目录必须直接位于技能目录下（`<skills>/<skill-name>/SKILL.md`）：

```bash
git clone https://github.com/autotask-run/autotask-skills.git /tmp/autotask-skills
mkdir -p ~/.codex/skills
cp -r /tmp/autotask-skills/autotask-todo /tmp/autotask-skills/autotask-notes ~/.codex/skills/
```

想跟随仓库更新可以用软链代替拷贝：

```bash
git clone https://github.com/autotask-run/autotask-skills.git ~/skills-repos/autotask-skills
ln -s ~/skills-repos/autotask-skills/autotask-todo ~/.codex/skills/autotask-todo
ln -s ~/skills-repos/autotask-skills/autotask-notes ~/.codex/skills/autotask-notes
```

## 4. 发布（维护者）

同一份内容发布到两个 git 托管，安装者任选其一克隆：

### 4.1 AutoTask 托管（hosted skill source）

服务端原生托管技能 git 仓库（`server/internal/services/skill_source_service.go`），创建后得到形如
`https://app.autotask.run/git/skill-sources/<id>/skills-source-<id>.git` 的克隆地址（认证：HTTP Basic，密码位填 API token，或 `Authorization: Bearer`）：

```bash
curl -s -X POST https://app.autotask.run/api/v1/skill-sources \
  -H "Authorization: Bearer <API token>" -H "Content-Type: application/json" \
  -d '{"name":"autotask-skills","type":"hosted"}'    # 记下返回的 git_http_url

# 把本目录内容作为仓库根推送（从 monorepo 发布）
cd <monorepo>
git subtree split -P packages/skills/external -b autotask-skills-pub
git push https://<user>:<API token>@app.autotask.run/git/skill-sources/<id>/skills-source-<id>.git \
  autotask-skills-pub:master
git branch -D autotask-skills-pub
```

### 4.2 GitHub 托管

公开仓库 `github.com/autotask-run/autotask-skills` 已创建；后续更新从 monorepo 重新拆分推送即可：

```bash
git subtree split -P packages/skills/external -b autotask-skills-pub
git push https://github.com/autotask-run/autotask-skills.git autotask-skills-pub:master
git branch -D autotask-skills-pub
```

GitHub 仓库建议公开（SKILL.md 无任何机密）；它同时能被 AutoTask 内置 `skill-provider-github` 插件扫描导入 Skills Store（配置 `repo_url` + `subdir`）。

### 4.3 版本化

- 两个 skill 的 `description` 与工具参数表必须与服务端 `server_personal_tools.go` 的工具签名保持同步；改工具时同一次提交里更新技能正文。
- 破坏性变更（工具改名/参数删除）在 README 顶部记录「最低服务端版本」。
