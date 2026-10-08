---
name: autotask-todo
description: 管理用户的 AutoTask 个人待办（个人工作区「捕获清单/待办页」同源数据）。当用户要求记录待办、记一下某件事、稍后提醒、安排日程、询问"今天/本周有什么事"、汇报完成某事、整理或改期任务、做日/周复盘时使用。触发词包括：待办、任务清单、提醒、日程、改期、复盘、完成/勾掉某事。
---

# AutoTask 个人待办

通过 AutoTask MCP 服务端（`autotask` MCP server）的 `*_personal_todo*` 工具，直接读写用户在 AutoTask 个人版的待办。数据与 AutoTask 待办页实时同源；所有工具都由服务端强制限定在调用者本人的个人工作区，无法（也无需）指定工作区。

## 前置条件

1. 客户端已配置 `autotask` MCP server 并成功连接（`tools/list` 里能看到 `list_personal_todos` 等工具）。未配置时按技能仓库 README 的「MCP 接入」一节操作。
2. API token 需要 `autotask:read` + `todo:write` 权限；只读场景（查询/复盘）仅需 `autotask:read`。

## 工具速查

| 工具 | 用途 | 关键参数 |
| --- | --- | --- |
| `list_personal_todos` | 按视图列出待办 | `view`: `inbox`(未安排)/`today`(默认)/`upcoming`/`completed`；`local_date` (YYYY-MM-DD)、`timezone`、`search`、`list_name`、`tag_name`、`page`、`page_size`(≤100) |
| `create_personal_todo` | 新建待办 | `title`(必填)、`note`、`planned_date`、`remind_at`、`priority`(low/normal/high)、`repeat_rule`(daily/weekly/monthly，需 `planned_date`)、`list_name`、`tag_names[]`、`parent_id`、`confirmed` |
| `update_personal_todo` | 改字段 / 完成 / 重开 | `id`(必填)、`title`、`note`、`planned_date`(空串清除)、`remind_at`(空串清除)、`priority`、`repeat_rule`、`list_name`、`tag_names[]`、`status`(`completed`/`open`) |
| `delete_personal_todo` | 软删除（进回收站可恢复） | `id`(必填) |
| `list_personal_todo_lists` | 列出清单及未完成计数 | 无 |
| `batch_update_personal_todos` | 批量整理（1-50 条） | `ids[]`(必填) + `planned_date` / `priority` / `list_id`(0=收件箱) / `delete`(布尔) 之一，互斥 |
| `personal_todo_review` | 日/周复盘统计 | `period`(day/week)、`local_date`、`timezone` |

## 核心工作流

### 1. 捕获行动项
用户说「帮我记一下……」「这事下周三要办」时：
- 解析出标题（动词开头的一句话最理想）、计划日期（日历日 YYYY-MM-DD，不是时间戳）、优先级、清单。
- 直接调 `create_personal_todo`。若返回 `status: "proposal_required"`，说明用户开启了「AI 创建待办需确认」：把 `proposal` 内容原样展示给用户，等用户确认后**带相同参数 + `confirmed: true` 重调**；返回 `status: "ai_todo_disabled"` 时告知用户已在设置中关闭该能力，不要重试。
- 清单名先参考 `list_personal_todo_lists`；传不存在的 `list_name` 会自动建清单，谨慎使用。

### 2. 每日/每周规划
「我今天有什么事」「帮我安排一下」→ `list_personal_todos`（`view=today`，客户端时区可传 `timezone`）；再视需要看 `upcoming`。给建议时结合 `personal_todo_review` 的逾期/未安排统计。

### 3. 完成与改期
「X 做完了」「把 Y 挪到周五」→ 先 `list_personal_todos` + `search` 定位 ID（不要猜 ID），再 `update_personal_todo`（`status: "completed"` 或 `planned_date`）。多条同类操作用 `batch_update_personal_todos`。

### 4. 复盘
「总结一下今天/本周的完成情况」→ `personal_todo_review`（`period=day|week`），输出完成数、改期数、逾期数和最近完成项。

## 红线

- **未经用户意图不创建待办**：闲聊中提到的口号、愿望不算行动项；拿不准就先问。
- `planned_date` 是本地日历日；`remind_at` 是 RFC3339 且必须晚于当前时间，过期的提醒会被拒绝。
- 周期规则（`repeat_rule`）必须搭配 `planned_date`；时区缺省按 Asia/Shanghai。
- 删除是软删除（回收站可恢复），但批量 `delete` 仍需用户明确同意后再执行。
- 需要的是「让 Agent 稍后自动执行某项工作」而非记录待办时，不要用这些工具，向用户说明并改用 AutoTask 的会话调度能力。
