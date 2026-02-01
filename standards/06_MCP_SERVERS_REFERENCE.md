# MCP Servers Reference Guide

**Note**: This guide covers MCP servers currently trialed in Cursor. Future goal is to promote these to a centralized global configuration managed by Ruler.

**Current Configuration**: `~/.cursor/mcp.json` (Cursor-specific, manually managed)

## Active Servers (Configured)

### ✅ **Notion** (Remote - Working)
- **Type**: Remote (Streamable HTTP)
- **URL**: https://mcp.notion.com/mcp
- **Setup**: OAuth via Notion app (already completed)
- **Use Cases**: 
  - Search workspace content
  - Create/update pages and databases
  - Manage comments and teams
  - Access based on your permissions

### ✅ **GitHub** (Local - Ready)
- **Type**: Local stdio via `npx`
- **Package**: `@modelcontextprotocol/server-github`
- **Setup**: GitHub Personal Access Token added
- **Use Cases**:
  - Repository management (clone, create, search)
  - Issues & pull requests (create, update, comment)
  - Code search & file operations
  - Actions & workflows
  - Security scanning

### ✅ **Context7** (Remote - Ready)
- **Type**: Remote (HTTP)
- **URL**: https://mcp.context7.com/mcp
- **Setup**: API key added
- **Use Cases**:
  - AI-powered documentation search
  - Version-specific code examples
  - Search across 90+ frameworks
  - Real-time documentation access

---

## Official Anthropic Servers (Added - Need Setup)

### 📁 **Filesystem** (Under Evaluation)
- **Type**: Local stdio
- **Package**: `@modelcontextprotocol/server-filesystem`
- **Scope**: `/Users/b` (home directory)
- **Setup**: ✅ No credentials needed
- **Status**: May be deprecated if benefit (multi-file reads) isn't compelling
- **Use Cases**:
  - Read multiple files simultaneously
  - Directory operations
  - File search
  - Safe scoped access

### 🐘 **PostgreSQL**
- **Type**: Local stdio
- **Package**: `@modelcontextprotocol/server-postgres`
- **Setup**: ✅ Connected to Patroni Dev Cluster
- **Connection**: `haproxy.bjzy.me:5433` (via HAProxy)
- **Database**: `postgres` (superuser access to all DBs)
- **Use Cases**:
  - Query all databases on cluster
  - Schema inspection across DBs
  - Database size analysis
  - Monitor cluster health
  - Access Keep, Portainer, and other app databases

### 💬 **Slack**
- **Type**: Local stdio
- **Package**: `@modelcontextprotocol/server-slack`
- **Setup**: ✅ Configured for bjzy.slack.com
- **Team ID**: T015SA9KKSN
- **Bot User**: cursor_bot
- **Use Cases**:
  - Read/send messages
  - Channel management
  - File sharing
  - Workflow automation

---

## Other Popular MCP Servers (Not Yet Added)

### **Project Management**
- **Linear** - Issue tracking
- **Jira** - Project management
- **Asana** - Task management

### **Development Tools**
- **Figma** - Design files & collaboration
- **Sentry** - Error tracking
- **Playwright** - Browser automation
- **GitLab** - DevSecOps platform

### **Cloud Platforms**
- **Vercel** - Deployment management
- **Netlify** - Web hosting
- **Heroku** - App deployment
- **Supabase** - Backend-as-a-service

### **Analytics**
- **PostHog** - Product analytics
- **Google Analytics** - Web analytics

---

## Configuration Management

### **Current Approach**: Manual Per-Agent Configuration
- **Cursor Config**: `~/.cursor/mcp.json` (manually created and managed)
- **Future Vision**: Ruler extension to write agent MCP configs automatically
- **Goal**: Project-specific configs augment or override global configs

### **Configuration Files**
```
~/.cursor/mcp.json              ← Cursor-specific (manual, with real credentials)
~/.continue/config.json         ← Continue-specific (future)
Other AI tool configs...        ← Future automation target
```

### **Enable/Disable Servers**
**Best Practice**: Create servers in **disabled state**, enable only when needed (goal: 99% disabled).

1. Open Cursor settings (`Cmd+,`)
2. Navigate to **Features** → **Model Context Protocol**
3. Toggle servers on/off as needed
4. **Always disable when done** to reduce resource usage and attack surface

### **Credential Management**
All credentials stored in **HashiCorp Vault** (primary secrets management for home lab).

**Secure Access** (DO NOT use root token in shell!):

```bash
# Set Vault address
export VAULT_ADDR="https://vault.bjzy.me:8200"

# Login with userpass (creates short-lived token in ~/.vault-token)
vault login -method=userpass username=myuser

# Retrieve credentials for MCP configuration
vault kv get -field=token kvProd/GitHub/PAT
vault kv get -field=api_key kvProd/Context7/API_KEY
vault kv get -field=postgresql_superuser_password kvProd/Keep/PostgreSQL-Dev
vault kv get -field=bot_token kvProd/Slack/BotToken

# Token auto-expires; re-login when needed
```

**For Automation**: Use AppRole authentication instead of user tokens (see `.ruler/docs/standards/04_SECURITY.md`).

### **Adding Credentials**

**For env vars** (like Brave Search):
```json
"env": {
  "BRAVE_API_KEY": "your-key-here"
}
```

**For headers** (like Context7):
```json
"headers": {
  "CONTEXT7_API_KEY": "your-key-here"
}
```

**For connection strings** (like PostgreSQL):
```json
"args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://user:pass@host:5432/dbname"]
```

---

## Security Notes

⚠️ **Critical Security Requirements:**

- **HashiCorp Vault**: Central role in home lab security - ALL secrets stored here
- **Never commit credentials**: Keep `~/.cursor/mcp.json` local only, add to `.gitignore`
- **Vault Authentication**: Use userpass or AppRole, NEVER store root token on disk
- **Scoped Permissions**: 
  - GitHub PAT: Minimal repo access only
  - PostgreSQL: Create limited users for specific tasks
  - Slack Bot: Only necessary scopes
  - Filesystem: Restrict to specific directories
- **Rotate Regularly**: Quarterly rotation for all tokens/credentials
- **Disable by Default**: Keep MCP servers off 99% of the time

---

## Testing New Servers

1. Add server config to agent MCP config file (e.g., `~/.cursor/mcp.json` for Cursor)
2. Restart the agent/tool
3. Check MCP status (in Cursor: green dot in settings = working)
4. Ask AI to test: "Can you list available [server] tools?"
5. **Immediately disable if not actively needed** (keep disabled 99% of the time)

---

## Troubleshooting

**Server won't connect:**
- Check credentials are correct
- Verify API keys haven't expired
- Ensure `npx` is installed (for local servers)
- Check network connectivity (for remote servers)

**Duplicate servers:**
- Check both global and project configs
- Disable in one location

**Performance issues:**
- Disable unused servers (primary recommendation)
- Use remote servers when possible (less local resource usage)
- Limit filesystem scope to specific directories
- Monitor with: `top`, `htop`, Activity Monitor

---

## Future Plans

### MCP 2.0 Transition
- Anthropic announced MCP 2.0 with local TypeScript functions
- May eliminate need for always-on MCP servers
- Evaluating migration path once stable

### Ruler Integration
- Extend Ruler to manage MCP server configurations
- Write to default global paths for each agent/tool
- Project-specific MCP configs augment or override global configs
- Centralized credential management via Vault plugin

### Custom MCP Servers
Potential home lab-specific MCP servers:
- **vSphere MCP**: VM management, resource monitoring
- **Patroni MCP**: Cluster health, failover operations
- **AWX MCP**: Job templates, inventory management
- **Observability MCP**: Unified Loki/Mimir/Grafana queries

### Global Configuration Target
Promote trialed servers to centralized global config:
- Notion ✅
- GitHub ✅
- Context7 ✅
- PostgreSQL ✅
- Slack ✅
- Filesystem ❓ (under evaluation)

---

## References

- Official MCP Docs: https://modelcontextprotocol.io/
- Anthropic Servers: https://github.com/modelcontextprotocol
- Cursor MCP Guide: https://docs.cursor.com/advanced/model-context-protocol
- Notion MCP: https://developers.notion.com/docs/mcp
- GitHub MCP: https://github.com/github/github-mcp-server
- Context7: https://context7.com/docs/installation

