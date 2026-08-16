# Jira 操作指南（创建 & 操作 issue）

以下为本地 mcp-atlassian 的工具用法。官方远程 MCP Server 的工具名以官方文档为准（https://support.atlassian.com/rovo/docs/supported-tools/），但操作流程相同。

> **工具名前缀**：实际暴露的工具名带命名空间——Claude Code/Desktop 中为 `jira_*` 原始名；**DSH 中为 `mcp__jira__jira_*`**（如 `mcp__jira__jira_search`）。下文统一写原始名 `jira_*`，DSH 使用时自行加前缀。

## 通用规则

- **先只读后写入**：任何写操作前，先用搜索/详情确认目标 issue 与字段
- **小结果验证**：JQL 先加 `maxResults=1` 执行，确认语法与权限后再全量
- **写前确认**：创建/更新/流转/删除，先向用户展示操作内容并征得同意
- **回读验证**：写操作后用 `jira_get_issue` 确认结果

## 1. 搜索（JQL）

```
jira_search  jql="project = PROJ AND status = Open ORDER BY priority DESC, created ASC"  maxResults=50
```

常用 JQL 片段（字段/运算符/函数/相对日期/排序的完整参考见 `references/jql-reference.md`）：

```jql
-- 指派给我且未解决
assignee = currentUser() AND resolution = Unresolved

-- 某项目 7 天内的 Bug
project = PROJ AND issuetype = Bug AND created >= -7d

-- 进行中的 sprint
project = PROJ AND sprint in openSprints() AND status = "In Progress"

-- 文本搜索
project = PROJ AND text ~ "登录失败"

-- 按关键词排序（低优先级的 severity 文本字段排序）
project = PROJ ORDER BY created DESC, priority ASC
```

## 2. 查询 issue 详情

```
jira_get_issue  issue_key="PROJ-123"
```

用于查看：字段值、描述、评论、附件、状态流转历史。执行写操作前用它确认当前状态。

## 3. 创建 issue

```
jira_create_issue
  project_key="PROJ"
  issue_type="Bug"          -- 常见：Task / Story / Bug / Epic / Sub-task
  summary="登录页在 Safari 下报 500"
  description="复现步骤：\n1. 打开登录页\n2. 点击登录\n预期：跳转成功\n实际：500 错误"
  -- 可选：
  -- assignee="zhangsan"（或账号 id）
  -- priority="High"
  -- labels=["frontend","safari"]
  -- custom fields 按项目字段名传（如 sprint、epic link、story points）
```

创建要点：
- `project_key` 与 `issue_type` 必填；`summary` 必填
- 不确定可用 issue 类型时，先用 `jira_get_issue` 查同项目已有 issue，或询问用户
- 自定义字段（customfield_xxxxx）以项目实际配置为准

## 4. 更新 issue

```
jira_update_issue
  issue_key="PROJ-123"
  summary="新标题"
  description="新描述"
  -- 或更新 assignee / priority / labels / 自定义字段
```

## 5. 状态流转（transition）

```
-- 先查询该 issue 当前可用的流转（如 To Do → In Progress → Done）
jira_get_issue  issue_key="PROJ-123"   -- 查看状态

-- 执行流转
jira_transition_issue
  issue_key="PROJ-123"
  transition="In Progress"   -- 用可用流转的名称或 id
  -- 可选 comment 或 resolution 字段（如 Done 时设 resolution）
```

要点：
- 流转名称因工作流配置而异；先查可用流转再执行
- 流转到 Done 通常需要设置 resolution（如 Fixed / Won't Fix）

## 6. 评论

```
jira_add_comment  issue_key="PROJ-123"  comment="已修复，验证通过。"
```

## 7. 其他常用能力（mcp-atlassian 93 个工具）

- 附件：上传/下载附件
- 链接：issue 间关联（blocks / relates to / duplicates）
- 子任务与 Epic：创建子任务、设置 Epic Link
- Sprint/看板：查询 sprint、看板与 backlog（工具名以实际暴露为准，可用 `list_tools` 查看）
- 批量操作：谨慎，先确认再执行

## 8. 分页与限流

- 大结果集按 50–100 条/页分页拉取
- 遇到 429 限流：指数退避重试（1s → 2s → 4s…），尊重 `Retry-After` 头
- 网络失败：幂等操作可重试；非幂等（如创建）避免重复执行，先查询确认

## 9. 排障

| 现象 | 处理 |
|------|------|
| `maxResults=1` 也报 JQL 语法错 | 检查引号与 `currentUser()`/`openSprints()` 是否被正确转义 |
| 创建报"required fields missing" | 用 `jira_get_issue` 看同类型 issue 的必填字段 |
| 流转报"invalid transition" | 当前状态不支持该流转，先查可用流转 |
| 搜索为空 | 确认项目 key 与权限；试试 `project = PROJ ORDER BY created DESC` |
| 工具名不对 | 用 `list_tools` 列出实际暴露的工具名（远程/本地版本工具名可能有差异） |
