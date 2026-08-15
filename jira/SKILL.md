---
name: jira
description: 通过 Atlassian 官方 MCP server 或本地 mcp-atlassian 与 Jira 集成，支持公司内部（内网/私有化）Jira 地址与多种授权方式（API Token / PAT / OAuth）。用于创建、查询、更新、流转（transition）issue，添加评论、附件，编写和执行 JQL 查询，管理 sprint/看板，以及排障 Atlassian API 集成。当用户提到 Jira、jira issue、工单、ticket、JQL、缺陷单、sprint、看板、以及"创建/更新/关闭工单"等任务时使用本 skill。
license: MIT
metadata:
  version: "1.0.0"
  domain: platform
  triggers: Jira, jira, ticket, 工单, 缺陷, issue, JQL, sprint, backlog, 看板, Atlassian, MCP
  role: expert
  scope: implementation
  related-skills: mcp-builder, security-reviewer
---

# Jira 集成 Skill（官方 & 安全优先）

在动手操作 Jira 之前，先按本 skill 确定 **接入方式** 与 **授权方式**，再执行操作。所有写操作（创建/更新/流转/删除）必须先向用户确认，并优先用只读探测验证权限。

## 0. 当前环境默认配置（内网私有化）

- Jira 地址：`https://jira.mychery.com`（内网 Server/Data Center）
- 接入方案：**本地 mcp-atlassian**（官方远程 MCP 是云托管，不支持内网私有化）
- 授权方式：**PAT（Personal Access Token）**，经环境变量 `JIRA_PERSONAL_TOKEN` 注入
- 客户端：Claude Code / Claude Desktop / DSH 通用（具体配置文件见 `references/mcp-setup.md`）

> 若用户环境与上述不同（如 Jira Cloud），按下方决策表重新选择方案。

## 1. 选择接入方案（先做决定）

| 场景 | 推荐方案 | 说明 |
|------|----------|------|
| 公司用 **Jira Cloud**（`xxx.atlassian.net`），追求最官方 | **Atlassian Rovo MCP Server（官方远程）** | Atlassian 官方 org 出品，Apache-2.0，已 GA；OAuth 2.1 / API Token |
| 公司 Jira 在**内网/私有化（Server / Data Center）**，或 URL 是公司自有域名 | **本地 mcp-atlassian**（`uvx mcp-atlassian`） | 官方 README 推荐的社区标准实现（MIT，5.7k+ stars），直接连任意 Jira URL，含内网地址；支持 Server/DC v8.14+ |
| 需要同时操作 Jira + Confluence | 两者都支持；本地 mcp-atlassian 更全 | 93 个工具，覆盖 Jira 与 Confluence |

> **关键约束**：官方远程 MCP Server 是 **Atlassian 云托管**的服务，只支持 Jira Cloud，**不支持内网/私有化 Jira**（数据会经 Atlassian 云）。内网 Jira 必须用本地 mcp-atlassian。

详细配置见 `references/mcp-setup.md`。

## 2. 授权（Authentication）

按你的 Jira 部署类型选择：

- **Jira Cloud**：`JIRA_USERNAME`（邮箱）+ `JIRA_API_TOKEN`（在 https://id.atlassian.com/manage-profile/security/api-tokens 创建）；或官方远程 MCP 的 OAuth 2.1 授权流程
- **Jira Server / Data Center（内网）**：`JIRA_PERSONAL_TOKEN`（PAT，管理员在用户配置中生成）
- **官方远程 MCP Server**：支持 OAuth 2.1 或 API Token，两种方式都行

**安全铁律**：
- 凭据**绝不硬编码**进代码/配置文件，从环境变量或密钥管理器读取（如 `env` 中的 `${JIRA_API_TOKEN}` 引用）
- 最小权限原则：API Token / PAT 只授予操作所需的最低权限（如只读就只给 Read）
- 参考 `references/authentication.md`

## 3. 核心操作（Jira 常用工具）

本地 mcp-atlassian 工具名（官方远程 MCP 工具名略不同，以 `references/jira-operations.md` 为准）：

| 操作 | 工具 | 说明 |
|------|------|------|
| 搜索 | `jira_search` | JQL 查询；**先加 `maxResults=1` 验证再全量执行** |
| 查详情 | `jira_get_issue` | 获取 issue 字段、评论、附件 |
| 创建 | `jira_create_issue` | 创建前确认必填字段（project、issuetype、summary 等） |
| 更新 | `jira_update_issue` | 改字段、描述、assignee |
| 流转 | `jira_transition_issue` | 状态流转（如 To Do → In Progress → Done），先查可用流转 |
| 评论 | `jira_add_comment` | 添加评论 |

## 4. 标准工作流

1. **确认接入方案与授权**：问清 Jira 是 Cloud 还是内网 Server/DC、URL、授权方式
2. **只读探测**：先执行一个 `maxResults=1` 的搜索，确认连通性与权限范围
3. **设计查询**：写好 JQL，先小结果验证语法与权限
4. **写操作前确认**：创建/更新/流转/删除必须向用户展示将要执行的操作并征得同意
5. **执行与验证**：执行后用 `jira_get_issue` 回读确认结果
6. **错误处理**：429 限流用指数退避；网络失败重试；记录关键 API 调用便于审计

## 5. MUST / MUST NOT

### MUST DO
- 先确认 Jira 部署类型（Cloud vs 内网 Server/DC）再选方案
- 凭据走环境变量/密钥管理，不写死在文件里
- 写操作前向用户确认
- JQL 先用 `maxResults=1` 验证
- 大结果集分页（50–100 条/页）
- 操作后回读验证

### MUST NOT DO
- 不硬编码 API Token / PAT / OAuth 密钥
- 不在日志或报错信息中泄露 issue 敏感内容与凭据
- 不跳过必填字段校验直接创建 issue
- 不对生产数据执行未确认的批量/写操作
- 不混用多种认证方式（同一会话保持一致）
- 不把内网 Jira 数据发给云托管的远程 MCP（数据合规）

## 参考文档

- `references/mcp-setup.md` — 两种接入方案的完整配置（含内网 URL）
- `references/authentication.md` — API Token / PAT / OAuth 2.1 详细配置
- `references/jira-operations.md` — JQL 语法、issue CRUD、流转、评论、附件、sprint
- `references/security.md` — 安全与合规清单
