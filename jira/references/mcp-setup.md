# MCP 接入方案配置（已按当前环境定制）

**当前环境**：内网私有化 Jira `https://jira.mychery.com`（Server / Data Center）
**结论**：必须使用**本地 mcp-atlassian**（官方远程 MCP 是 Atlassian 云托管，不支持内网私有化部署）。

## 前置条件

1. 本机可访问 `https://jira.mychery.com`（内网/VPN）
2. 已安装 Python 3.10+ 与 [uv](https://docs.astral.sh/uv/)（推荐），或 Node.js（用于 npx）
3. 已生成 **PAT（Personal Access Token）**，步骤见 `authentication.md`

## 环境变量（所有客户端共用）

把 PAT 放入环境变量，**不要写进任何配置文件明文**：

```bash
# ~/.zshrc 或 ~/.bashrc
export JIRA_URL="https://jira.mychery.com"
export JIRA_PERSONAL_TOKEN="粘贴你的PAT"
```

> 也可用 `direnv`/`.env`（确保 `.gitignore` 包含 `.env`）。

---

## 客户端 A：Claude Code（项目级 `.mcp.json`）

在项目根目录创建 `.mcp.json`：

```json
{
  "mcpServers": {
    "jira": {
      "command": "uvx",
      "args": ["mcp-atlassian"],
      "env": {
        "JIRA_URL": "${JIRA_URL}",
        "JIRA_PERSONAL_TOKEN": "${JIRA_PERSONAL_TOKEN}"
      }
    }
  }
}
```

或用户级（`~/.claude.json` 的 `mcpServers` 字段），或 `claude mcp add`：

```bash
claude mcp add jira --env JIRA_URL=${JIRA_URL} --env JIRA_PERSONAL_TOKEN=${JIRA_PERSONAL_TOKEN} -- uvx mcp-atlassian
```

## 客户端 B：Claude Desktop（`claude_desktop_config.json`）

macOS 路径 `~/Library/Application Support/Claude/claude_desktop_config.json`：

```json
{
  "mcpServers": {
    "jira": {
      "command": "uvx",
      "args": ["mcp-atlassian"],
      "env": {
        "JIRA_URL": "https://jira.mychery.com",
        "JIRA_PERSONAL_TOKEN": "${JIRA_PERSONAL_TOKEN}"
      }
    }
  }
}
```

## 客户端 C：DSH（`~/.dsh/profiles/web/cordis.patch.yml`）

DSH 通过 `@deepseek-ai/dsh-mcp-client` 插件接入，在 profile 的 patch 层追加：

```yaml
# ~/.dsh/profiles/web/cordis.patch.yml 的顶层数组追加：
- id: mcp-jira
  name: '@deepseek-ai/dsh-mcp-client'
  config:
    serverName: jira
    transport: stdio
    command: uvx
    args: ['mcp-atlassian']
    env:
      JIRA_URL: !!js process.env.JIRA_URL
      JIRA_PERSONAL_TOKEN: !!js process.env.JIRA_PERSONAL_TOKEN
```

- 工具以 `mcp__jira__*` 命名空间注册（如 `mcp__jira__jira_create_issue`）
- 保存后 HMR 热重载即生效，无需重启
- `serverName` 在同一 profile 内必须唯一（别与其他 MCP 插件重名）

## 无 uv 环境的备选（任意客户端）

把 `uvx mcp-atlassian` 换成：

```json
{
  "command": "npx",
  "args": ["-y", "@sooperset/mcp-atlassian"],
  "env": { "JIRA_URL": "...", "JIRA_PERSONAL_TOKEN": "..." }
}
```

或 Docker：`docker run -e JIRA_URL=... -e JIRA_PERSONAL_TOKEN=... sooperset/mcp-atlassian`

## 验证连通性

配置完成后做只读验证：

```
jira_search  jql: "project = PROJ ORDER BY created DESC"  maxResults: 1
```

（DSH 中为 `mcp__jira__jira_search`）

返回正常即接入成功。若失败：

| 现象 | 处理 |
|------|------|
| 连接超时 | 确认本机可达 `jira.mychery.com`（内网/VPN） |
| 401/403 | PAT 错误或权限不足，重新生成并确认 scopes |
| 版本不支持 | Server/DC 需 ≥ 8.14 |
| DSH 中工具未出现 | 查看 DSH 日志中 mcp-jira 插件激活信息；确认 `failOnStartupError` 或重连状态 |
