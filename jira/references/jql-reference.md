# JQL 参考（Jira Query Language）

> 提取并改编自 ClawHub `jira@1.3.3` 的 MCP 参考文档（JQL 部分）。
>
> **JQL 与后端无关**：无论用本地 `mcp-atlassian`（`jira_search` 工具）、官方远程 Rovo MCP，还是 `jira` CLI，JQL 语法完全一致。本文件用于补充主 `SKILL.md` 中偏薄的 JQL 内容，让 agent 能写出**正确且强**的查询，而不是只会 `get_issue` 一张张点。
>
> 用法：把下面任一条 JQL 串作为 `jira_search` 的 `jql` 参数传入即可（记得先 `maxResults=1` 小结果验证，再全量执行）。

## 基础语法

```
field operator value [AND|OR field operator value]
```

## 常用字段

| 字段 | 含义 | 示例 |
|------|------|------|
| `project` | 项目 key | `project = "PROJ"` |
| `issuetype` | 问题类型 | `issuetype = Bug` |
| `status` | 状态 | `status = "In Progress"` |
| `assignee` | 指派人 | `assignee = currentUser()` |
| `reporter` | 报告人 | `reporter = "jobarksdale"` |
| `priority` | 优先级 | `priority = High` |
| `labels` | 标签 | `labels = "backend"` |
| `component` | 组件 | `component = "API"` |
| `created` | 创建日期 | `created >= -30d` |
| `updated` | 最后更新 | `updated >= -7d` |
| `resolved` | 解决日期 | `resolved >= startOfMonth()` |
| `sprint` | 冲刺 | `sprint in openSprints()` |
| `epic` | 父 Epic | `"Epic Link" = PROJ-100` |
| `parent` | 父 issue | `parent = PROJ-50` |
| `text` | 全文搜索 | `text ~ "authentication"` |
| `summary` | 标题搜索 | `summary ~ "login"` |
| `description` | 描述搜索 | `description ~ "OAuth"` |

## 运算符

| 运算符 | 含义 | 示例 |
|--------|------|------|
| `=` | 精确匹配 | `status = Done` |
| `!=` | 不等于 | `status != Closed` |
| `~` | 包含（文本） | `summary ~ "auth*"` |
| `!~` | 不包含 | `summary !~ "test"` |
| `>` `>=` `<` `<=` | 比较 | `priority >= High` |
| `IN` | 多值匹配 | `status IN (Open, "In Progress")` |
| `NOT IN` | 排除值 | `status NOT IN (Done, Closed)` |
| `IS` | 空值检查 | `assignee IS EMPTY` |
| `IS NOT` | 非空 | `assignee IS NOT EMPTY` |
| `WAS` | 历史值 | `status WAS "In Progress"` |
| `CHANGED` | 字段变更 | `status CHANGED` |

## 函数

| 函数 | 含义 | 示例 |
|------|------|------|
| `currentUser()` | 当前登录用户 | `assignee = currentUser()` |
| `now()` | 当前时间戳 | `created <= now()` |
| `startOfDay()` | 今天零点 | `updated >= startOfDay()` |
| `startOfWeek()` | 本周起点 | `created >= startOfWeek()` |
| `startOfMonth()` | 本月起点 | `created >= startOfMonth()` |
| `endOfDay()` | 今天结束 | `due <= endOfDay()` |
| `openSprints()` | 进行中的冲刺 | `sprint in openSprints()` |
| `closedSprints()` | 已完成的冲刺 | `sprint in closedSprints()` |
| `linkedIssues()` | 关联 issue | `issue in linkedIssues("PROJ-123")` |

## 相对日期

```jql
# 天
created >= -7d    # 最近 7 天
updated >= -30d   # 最近 30 天

# 周
created >= -2w    # 最近 2 周

# 月
created >= -1M    # 上个月

# 具体日期
created >= "2024-01-01"
```

## 排序

```jql
# 按优先级降序
project = PROJ ORDER BY priority DESC

# 多字段排序
project = PROJ ORDER BY status ASC, created DESC
```

## 复杂查询示例

```jql
# 我未关闭的高优先级 issue
assignee = currentUser() AND status NOT IN (Done, Closed) AND priority >= High

# 本周新增的 Bug
issuetype = Bug AND created >= startOfWeek() ORDER BY priority DESC

# 进行中且含子任务的 Epic
issuetype = Epic AND status = "In Progress" AND issueFunction in hasLinks("is parent of")

# 我最近更新过的 issue
updatedBy = currentUser() AND updated >= -7d ORDER BY updated DESC

# 被阻塞的 issue
status = Blocked OR "Flagged" = "Impediment"

# 关联到指定 Epic 的 issue
"Epic Link" = PROJ-100 AND status != Done

# 当前冲刺待办
sprint in openSprints() AND status = "To Do" ORDER BY rank ASC

# 未指派的高优先级 Bug
issuetype = Bug AND assignee IS EMPTY AND priority >= High

# 我关注的 issue
watcher = currentUser()

# 团队最近解决的 issue
resolved >= -7d AND project = PROJ ORDER BY resolved DESC
```
