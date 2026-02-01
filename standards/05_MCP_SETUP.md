# MCP Server Configuration for Bjzy Labs

Model Context Protocol (MCP) servers extend AI assistant capabilities through standardized integrations.

## Overview

MCP servers provide AI assistants with access to:
- **Documentation** (Notion) - Workspace knowledge base
- **Version Control** (GitHub) - Repository, issues, PRs, actions
- **Library Docs** (Context7) - AI-powered documentation search
- **Databases** (PostgreSQL) - Patroni dev cluster queries
- **Communication** (Slack) - Workspace messaging & automation
- **Files** (Filesystem) - Local file operations

**Default State**: All MCP servers should be **disabled by default** and enabled only when needed (goal: 99% disabled).

## Transport Protocols

### Remote Servers (Streamable HTTP)
Modern HTTP-based transport for remote MCP services:
- **Notion**: `https://mcp.notion.com/mcp`
- **Context7**: `https://mcp.context7.com/mcp`

**Characteristics:**
- No long-lived connections
- Works through standard HTTP proxies
- OAuth/API key authentication via headers
- Recommended for production

### Local Servers (stdio)
Local processes via npx for sensitive/direct access:
- **GitHub**: `@modelcontextprotocol/server-github`
- **PostgreSQL**: `@modelcontextprotocol/server-postgres`
- **Slack**: `@modelcontextprotocol/server-slack`
- **Filesystem**: `@modelcontextprotocol/server-filesystem`

**Characteristics:**
- Process-level isolation
- Direct credential access via env vars
- Faster for local resources
- No network overhead

## Current MCP Configuration

### Agent Configuration (Cursor Example)

**File**: `~/.cursor/mcp.json`

```json
{
  "mcpServers": {
    "Notion": {
      "url": "https://mcp.notion.com/mcp",
      "description": "Official Notion MCP - Documentation & knowledge base"
    },
    "GitHub": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "<from-vault>"
      },
      "description": "GitHub MCP - Repository, issues, PRs, actions"
    },
    "Context7": {
      "url": "https://mcp.context7.com/mcp",
      "headers": {
        "CONTEXT7_API_KEY": "<from-vault>"
      },
      "description": "Context7 MCP - AI-powered documentation search"
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/b"],
      "description": "Local filesystem access (scoped to home directory)"
    },
    "postgres": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-postgres",
        "postgresql://postgres:<password>@haproxy.bjzy.me:5433/postgres"
      ],
      "description": "Patroni PostgreSQL Dev Cluster (via HAProxy)"
    },
    "slack": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-slack"],
      "env": {
        "SLACK_BOT_TOKEN": "<from-vault>",
        "SLACK_TEAM_ID": "T015SA9KKSN"
      },
      "description": "Slack workspace integration - bjzy.slack.com"
    }
  }
}
```

**Security Note**: Replace `<from-vault>` placeholders with actual credentials from HashiCorp Vault. **NEVER commit this file to version control!**

### Retrieving Credentials from Vault

```bash
# Set Vault address
export VAULT_ADDR="https://vault.bjzy.me:8200"

# Login (creates short-lived token in ~/.vault-token)
vault login -method=userpass username=myuser

# Retrieve credentials
vault kv get -field=token kvProd/GitHub/PAT
vault kv get -field=api_key kvProd/Context7/API_KEY
vault kv get -field=postgresql_superuser_password kvProd/Keep/PostgreSQL-Dev
vault kv get -field=bot_token kvProd/Slack/BotToken
```

## Hosting Custom MCP Servers

You can host your own MCP servers on Docker Swarm with Traefik.

**Template Available**: Use the standard MCP playbook template from the ansible repository:
- **Location**: `BjzyLabs/ansible/playbooks/TemplateMCP-manage.yml`
- **GitHub**: https://github.com/BjzyLabs/ansible/blob/develop/playbooks/TemplateMCP-manage.yml
- **Features**: Docker Swarm support, Vault integration, Slack notifications, standard operations

### Traefik Configuration for Streamable HTTP

Traefik requires special configuration to support streaming responses:

```yaml
# docker-compose.yml
services:
  my-mcp-server:
    image: my-mcp-server:latest
    networks:
      - traefik-public
    deploy:
      labels:
        # Enable Traefik
        - "traefik.enable=true"
        
        # Router configuration
        - "traefik.http.routers.mcp.rule=Host(`mcp.bjzy.me`)"
        - "traefik.http.routers.mcp.entrypoints=websecure"
        - "traefik.http.routers.mcp.tls.certresolver=letsencrypt"
        
        # CRITICAL: Disable buffering for streaming
        - "traefik.http.routers.mcp.middlewares=mcp-nobuffer@docker,mcp-auth@docker"
        
        # No-buffer middleware
        - "traefik.http.middlewares.mcp-nobuffer.buffering.maxRequestBodyBytes=0"
        - "traefik.http.middlewares.mcp-nobuffer.buffering.maxResponseBodyBytes=0"
        - "traefik.http.middlewares.mcp-nobuffer.buffering.memRequestBodyBytes=0"
        - "traefik.http.middlewares.mcp-nobuffer.buffering.memResponseBodyBytes=0"
        
        # Authentication (recommended)
        - "traefik.http.middlewares.mcp-auth.basicauth.users=user:$$apr1$$..."
        
        # Service configuration
        - "traefik.http.services.mcp.loadbalancer.server.port=8080"
        
        # High timeout for long operations (5 minutes)
        - "traefik.http.services.mcp.loadbalancer.server.responseTimeout=300s"
      
      # Resource limits (prevent exhaustion)
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M

networks:
  traefik-public:
    external: true
```

### Why These Labels Matter

| Label | Purpose | Impact if Missing |
|-------|---------|-------------------|
| `buffering.*Bytes=0` | Disable response buffering | Streaming responses won't work |
| `responseTimeout=300s` | Allow long operations | Premature timeouts |
| `basicauth.*` | Protect endpoint | Unauthorized access |
| `memory limits` | Prevent exhaustion | Resource hogging |

**Important**: These settings only affect MCP services. Your other Traefik services keep their normal buffering/timeout settings.

### Testing Your MCP Server

```bash
# 1. Check service is running
docker service ls | grep mcp

# 2. Test endpoint connectivity
curl -I https://mcp.bjzy.me/health

# 3. Test streaming (should not buffer)
curl -N https://mcp.bjzy.me/mcp

# 4. Monitor logs
docker service logs my-mcp-server -f

# 5. Check for connection issues
docker service ps my-mcp-server --no-trunc
```

## Managing MCP Servers

### Enable/Disable in Cursor

1. Open Cursor Settings (`Cmd+,`)
2. Navigate to **Features** → **Model Context Protocol**
3. Toggle servers on/off as needed
4. No restart required

**Best Practice**: Keep servers disabled except when actively needed.

### Verification

```bash
# List enabled MCP servers (in Cursor)
# Cursor UI: Settings → MCP → View active servers

# Test specific MCP tool
# Ask AI: "Can you list the available GitHub MCP tools?"

# Check network connectivity for remote servers
curl -I https://mcp.notion.com/mcp
curl -I https://mcp.context7.com/mcp
```

## Troubleshooting

### Local Servers Won't Start

**Problem**: `npx` command fails or times out

**Solutions**:
```bash
# Check Node.js is installed
node --version  # Should be v18+

# Clear npx cache
npm cache clean --force

# Test npx directly
npx -y @modelcontextprotocol/server-github --version

# Check environment variables
echo $GITHUB_PERSONAL_ACCESS_TOKEN
```

### Remote Servers Timeout

**Problem**: Streamable HTTP connections hang

**Solutions**:
- Verify network connectivity: `curl -I <mcp-url>`
- Check firewall rules
- Confirm API keys are valid
- Review Traefik logs: `docker service logs traefik -f`

### Credentials Not Working

**Problem**: "Authentication failed" errors

**Solutions**:
```bash
# Re-authenticate with Vault
vault login -method=userpass username=myuser

# Verify secret exists
vault kv get kvProd/GitHub/PAT

# Check token hasn't expired
vault token lookup

# Regenerate token if needed (at source: GitHub, Slack, etc.)
```

### High Memory Usage

**Problem**: MCP server consuming excessive RAM

**Solutions**:
- Set resource limits in Docker Swarm (see config above)
- Disable unused servers
- Check for memory leaks in custom MCP code
- Monitor with: `docker stats`

## Security Checklist

- [ ] All credentials stored in HashiCorp Vault
- [ ] MCP endpoints use HTTPS only
- [ ] Authentication enabled on hosted MCP servers
- [ ] Filesystem MCP scoped to specific directory
- [ ] PostgreSQL user has minimal required permissions
- [ ] Slack bot has only necessary scopes
- [ ] GitHub PAT has limited repository access
- [ ] MCP servers disabled when not needed
- [ ] No credentials in version control

## Future Improvements

### Planned Enhancements

1. **Ruler Integration**: Extend Ruler to manage MCP configurations centrally
2. **Global Configuration**: Promote trialed MCP servers to global config
3. **MCP 2.0**: Transition to local TypeScript functions (per Anthropic announcement)
4. **Automated Credentials**: Vault agent for automatic credential injection
5. **Custom MCP Library**: Build home lab-specific MCP servers for:
   - vSphere management
   - Patroni cluster operations
   - Custom monitoring queries

### Configuration Management Vision

```
Future State:
  Ruler → Generates MCP configs → ~/.cursor/mcp.json (Cursor)
                                 → ~/.continue/config.json (Continue)
                                 → Other AI tools...

Current State:
  Manual configuration per tool
```

## Additional Resources

- **MCP Specification**: https://modelcontextprotocol.io/
- **Anthropic MCP Servers**: https://github.com/modelcontextprotocol
- **Cursor MCP Guide**: https://docs.cursor.com/advanced/model-context-protocol
- **Traefik Streaming**: https://doc.traefik.io/traefik/middlewares/http/buffering/
- **Home Lab Docs**: `.ruler/docs/standards/06_MCP_SERVERS_REFERENCE.md`

---

**Remember**: MCP servers are powerful tools. Keep them disabled by default and enable only when needed.
