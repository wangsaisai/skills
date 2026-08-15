# 授权方式（Authentication）

按 Jira 部署类型选择对应的授权方式。同一会话内**只使用一种认证方式**，不要混用。

## 1. Jira Cloud — API Token

1. 在 https://id.atlassian.com/manage-profile/security/api-tokens 创建 API Token
2. 配置本地 mcp-atlassian：

```json
{
  "mcpServers": {
    "mcp-atlassian": {
      "command": "uvx",
      "args": ["mcp-atlassian"],
      "env": {
        "JIRA_URL": "https://your-company.atlassian.net",
        "JIRA_USERNAME": "your.email@company.com",
        "JIRA_API_TOKEN": "${JIRA_API_TOKEN}"
      }
    }
  }
}
```

- `JIRA_USERNAME` 是 Atlassian 账号邮箱，不是昵称
- Token 泄露后立即在 Atlassian 账号安全页吊销

## 2. Jira Server / Data Center（内网）— PAT（当前环境：jira.mychery.com）

1. 浏览器登录 `https://jira.mychery.com`
2. 右上角头像 → **Profile**（个人资料）→ 左侧 **Personal Access Tokens** 页签
   - 旧版路径：`https://jira.mychery.com/secure/ViewProfile.jspa` → Personal Access Tokens
3. 点击 **Create token**，填写名称（如 `dsh-mcp`），设置过期时间
4. 生成后**立即复制保存**（只显示一次），放入环境变量：

```bash
export JIRA_URL="https://jira.mychery.com"
export JIRA_PERSONAL_TOKEN="粘贴PAT"
```

5. 配置 mcp-atlassian（三种客户端完整配置见 `mcp-setup.md`）：

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

- 内网 Jira 用 PAT，**不要**用 API Token（API Token 仅用于 Cloud）
- Server/DC 版本需 ≥ 8.14
- PAT 支持按需吊销（Profile → Personal Access Tokens → Revoke），泄露时立即吊销

## 3. 官方远程 MCP Server（Jira Cloud）— OAuth 2.1 / API Token

### OAuth 2.1（推荐）

1. 在 MCP 客户端添加远程服务器：`https://mcp.atlassian.com/v1/mcp`
2. 首次调用时自动打开浏览器完成 Atlassian 账号授权
3. 授权页面会列出请求的权限范围（scopes），如 `read:jira-work`、`write:jira-work`，只授予需要的范围

### API Token（备用）

在环境变量中提供 API Token 跳过交互式授权（具体变量名以官方文档为准：https://support.atlassian.com/rovo/docs/getting-started-with-the-atlassian-remote-mcp-server/）。

## 4. 权限范围（最小权限原则）

- 只读任务：只授权 `read:jira-work` 级别权限
- 需要创建/更新：才加 `write:jira-work`
- 不需要 Confluence 就别配 Confluence 凭据
- 定期轮换 Token；离职/不再使用时吊销

## 5. 安全铁律

- 凭据从环境变量或密钥管理（Vault/1Password/.env 且 .gitignore）读取，**绝不硬编码**
- 不在日志、报错信息、聊天输出中打印完整 Token
- 不把 Token 提交进 git；检查 `.gitignore` 是否忽略 `.env`
- Token 意外泄露 → 立即吊销并重新生成

## 常见问题

| 现象 | 原因 | 处理 |
|------|------|------|
| 401 Unauthorized | Token 错误/过期/无权限 | 检查 Token、权限范围；重新生成 |
| 403 Forbidden | 权限不足 | 联系管理员授予所需范围 |
| 404 | 站点/项目不可见 | 确认 `JIRA_URL` 与项目权限 |
| Server/DC 不支持 | 版本 < 8.14 | 升级或联系管理员 |
