---
name: haproxy
description: Manages HAProxy load balancer for high availability traffic distribution, SSL termination, and backend health monitoring including configuration validation, backend health checks, and routing verification. Use when user mentions load balancing, backends, SSL certificates, HAProxy, traffic distribution, or health checks.
---

# HAProxy Load Balancer Management

## Instructions

Use this skill to manage and monitor HAProxy, the high-performance TCP/HTTP load balancer used for distributing traffic across backend services, SSL termination, and ensuring high availability of critical infrastructure. HAProxy operates as the edge load balancer in the Home Lab architecture.

### Configuration

The preferred method of interaction is the `haproxyctl` CLI tool and direct HTTP API calls to HAProxy stats endpoints. HAProxy runs as a highly available service providing load balancing for web applications, databases, and internal services.

**Smart Authentication:**
HAProxy stats endpoint may require basic auth. Verify connectivity before operations.

```bash
# Check if HAProxy stats endpoint is accessible
if ! curl -fsS http://localhost:8400/stats > /dev/null 2>&1; then
  echo "HAProxy stats endpoint not accessible or requires auth"
  # Retrieve credentials from Vault if needed
  HAPROXY_USER=$(vault kv get -field=stats_user kvProd_v2/HAProxy/Application-Prod)
  HAPROXY_PASS=$(vault kv get -field=stats_password kvProd_v2/HAProxy/Application-Prod)
fi

# Verify HAProxy service is running
systemctl is-active --quiet haproxy || echo "HAProxy service not running"
```

### Certificate Handling (Homelab)

HAProxy handles SSL termination for backend services. Certificate files must be in PEM format (cert + key).

```bash
# HAProxy certificate format (combined PEM)
cat /etc/haproxy/certs/bjzy.me.crt /etc/haproxy/certs/bjzy.me.key > /etc/haproxy/certs/bjzy.me.pem

# Verify certificate validity
openssl x509 -in /etc/haproxy/certs/bjzy.me.pem -noout -dates

# Test SSL configuration
echo | openssl s_client -connect bjzy.me:443 -servername bjzy.me
```

#### Bjzy Labs defaults

- **HAProxy Deployment**: High availability configuration with Keepalived VIP
- **Common use cases**:
  - **Load Balancing**: Distribute traffic across multiple backend servers
  - **SSL Termination**: Handle HTTPS encryption at the edge
  - **Health Monitoring**: Continuous backend health checks
  - **Connection Routing**: Route traffic to appropriate services based on rules
- **Integration Points**:
  - **Traefik**: Routes to Docker Swarm services via Traefik mesh
  - **Vault**: Load balances Vault cluster endpoints
  - **Portainer**: Distributes traffic across Portainer instances
  - **Keepalived**: Provides VIP failover for HAProxy redundancy

### Environment and Guardrails (Bjzy Labs)

- **HAProxy Access**:
  - HAProxy runs on dedicated load balancer nodes
  - Stats interface available on management network
  - Configuration files in `/etc/haproxy/`
  - Runtime socket for management operations
- **Security Rules**:
  - Always validate configuration before reloading
  - Use read-only operations for health monitoring
  - Be cautious with backend server modifications (affect live traffic)
  - Test configuration changes in staging when possible
- **CLI Availability**:
  - `haproxyctl` tool for runtime management
  - `curl` for stats endpoint queries
  - Direct configuration file editing for persistent changes

### Standard Operating Procedure (SOP)

When asked to "Check HAProxy," "Validate configuration," or "Monitor backends":

1. **Verify Service:** Check HAProxy process and stats endpoint
2. **Identify Operation:** Determine if you need health checks, config validation, or routing verification
3. **Execute Query:** Use appropriate haproxyctl or curl command
4. **Verify Results:** Cross-check backend health and traffic distribution
5. **Document Changes:** Log any configuration modifications

## Examples

### 1. Check HAProxy Service Status

Verify HAProxy is running and accessible.

- **Method:** `haproxyctl` and system checks
- **Command Pattern:**

```bash
# Check HAProxy process status
systemctl status haproxy

# Verify HAProxy version and build info
haproxy -vv

# Check if stats socket is accessible
haproxyctl show info

# Test stats endpoint availability
curl -s http://localhost:8400/stats
```

### 2. Monitor Backend Health

Check the health status of all configured backend servers.

- **Method:** `haproxyctl` and stats endpoint
- **Command Pattern:**

```bash
# Show all backend server status
haproxyctl show servers

# Get detailed backend information
haproxyctl show backend

# Check specific backend health
curl -s "http://localhost:8400/stats;csv" | grep BACKEND

# Monitor real-time stats
watch -n 1 "curl -s http://localhost:8400/stats | grep -E '(UP|DOWN)'"
```

### 3. Validate Configuration

Test HAProxy configuration syntax before applying changes.

- **Method:** `haproxy` config validation
- **Command Pattern:**

```bash
# Validate configuration file syntax
haproxy -f /etc/haproxy/haproxy.cfg -c

# Check specific configuration section
haproxy -f /etc/haproxy/haproxy.cfg -c -f <section>

# Test configuration with dry run
haproxy -f /etc/haproxy/haproxy.cfg -d

# Show parsed configuration
haproxy -f /etc/haproxy/haproxy.cfg -f -vv
```

### 4. Reload Configuration

Apply configuration changes without dropping connections.

- **Method:** `haproxyctl` and systemd
- **Command Pattern:**

```bash
# Graceful reload (maintains connections)
haproxyctl reload

# Systemd reload method
systemctl reload haproxy

# Force restart (drops connections)
systemctl restart haproxy

# Check reload success
systemctl status haproxy
```

### 5. Query Traffic Statistics

Get detailed traffic and performance metrics.

- **Method:** Stats endpoint and `haproxyctl`
- **Command Pattern:**

```bash
# Get comprehensive stats in CSV format
curl -s "http://localhost:8400/stats;csv"

# Get human-readable stats
curl -s http://localhost:8400/stats | python3 -m json.tool

# Show specific backend stats
curl -s "http://localhost:8400/stats;csv" | grep <backend_name>

# Monitor connection rates
haproxyctl show stat | grep -E "(scur|scon)"
```

### 6. Backend Server Management

Manage backend server availability and weight.

- **Method:** `haproxyctl` runtime commands
- **Command Pattern:**

```bash
# Enable backend server
haproxyctl enable server <backend>/<server>

# Disable backend server (drain connections)
haproxyctl disable server <backend>/<server>

# Set server weight
haproxyctl set weight <backend>/<server> <weight>

# Put server in maintenance mode
haproxyctl set server <backend>/<server> state maint
```

### 7. SSL Certificate Management

Monitor and manage SSL termination configuration.

- **Method:** Configuration file checks and stats
- **Command Pattern:**

```bash
# Check SSL certificate expiration
openssl x509 -in /etc/haproxy/certs/<cert>.pem -noout -dates

# Verify SSL certificate chain
openssl verify -CAfile /etc/haproxy/ca-bundle.crt /etc/haproxy/certs/<cert>.pem

# Check SSL session statistics
curl -s "http://localhost:8400/stats;csv" | grep ssl

# Test SSL configuration
haproxy -f /etc/haproxy/haproxy.cfg -c | grep ssl
```

### 8. Health Check Configuration

Monitor and troubleshoot health check behavior.

- **Method:** Configuration and runtime monitoring
- **Command Pattern:**

```bash
# Show health check configuration
grep -A 10 "option healthchk" /etc/haproxy/haproxy.cfg

# Monitor health check results
watch -n 2 "haproxyctl show servers | grep -E '(check|status)'"

# Test health check endpoint manually
curl -I http://<backend_server>:<port>/health

# Check health check intervals
curl -s "http://localhost:8400/stats;csv" | grep -E "(check|inter)"
```

### 9. Connection Routing Verification

Verify traffic is being routed correctly to backends.

- **Method:** Stats and manual testing
- **Command Pattern:**

```bash
# Check active connections per backend
curl -s "http://localhost:8400/stats;csv" | grep -E "(scur|scon)"

# Test routing to specific backend
curl -H "Host: <domain>" http://<haproxy_ip>:<port>/path

# Check connection distribution
haproxyctl show stat | grep -E "(rate|count)"

# Verify stickiness tables
haproxyctl show table | grep <table_name>
```

### 10. Keepalived Integration

Monitor VIP failover and HAProxy redundancy.

- **Method:** Keepalived and network checks
- **Command Pattern:**

```bash
# Check Keepalived status
systemctl status keepalived

# Verify VIP is assigned
ip addr show | grep <vip_address>

# Check HAProxy on both nodes
ssh <haproxy_node2> "systemctl status haproxy"

# Monitor failover logs
journalctl -u keepalived -f
```

### 11. Backend Health Check Configuration

Configure and verify health checks for backend servers.

- **Method:** HAProxy configuration
- **Purpose:** Ensure traffic only goes to healthy backends

**Command Pattern:**
```bash
# View current health check configuration
grep -A 10 "backend" /etc/haproxy/haproxy.cfg | grep -E "check|health"

# Example health check config - Use Ansible/templating for idempotent management
# For one-time setup, verify backend doesn't exist first:
if ! grep -q "backend api_servers" /etc/haproxy/haproxy.cfg; then
  sudo tee -a /etc/haproxy/haproxy.cfg > /dev/null <<'EOF'
backend api_servers
  mode http
  balance roundrobin
  option httpchk GET /health HTTP/1.1
  http-check expect status 200
  server api1 192.168.60.10:8080 check inter 5s fall 3 rise 2
  server api2 192.168.60.11:8080 check inter 5s fall 3 rise 2
  server api3 192.168.60.12:8080 check inter 5s fall 3 rise 2
EOF
fi

# Validate configuration
haproxy -c -f /etc/haproxy/haproxy.cfg

# Reload HAProxy to apply changes
systemctl reload haproxy

# Verify health check status
echo "show stat" | socat stdio /var/run/haproxy/admin.sock | grep -i check

# Check specific backend health
curl -s http://localhost:8400/stats | grep api_servers
```

## Troubleshooting

### HAProxy Service Issues

```bash
# Check if HAProxy is running
ps aux | grep haproxy

# Check system logs for errors
journalctl -u haproxy -n 50

# Verify configuration syntax
haproxy -f /etc/haproxy/haproxy.cfg -c

# Check resource limits
ulimit -n
```

### Backend Server Down

```bash
# Check backend server status
haproxyctl show servers | grep <backend>

# Test backend connectivity manually
curl -I http://<backend_ip>:<port>/health

# Check health check configuration
grep -A 5 "server <backend>" /etc/haproxy/haproxy.cfg

# Verify network connectivity
ping -c 3 <backend_ip>
```

### Configuration Reload Failures

```bash
# Check configuration syntax
haproxy -f /etc/haproxy/haproxy.cfg -c

# Look for specific errors
haproxy -f /etc/haproxy/haproxy.cfg -d 2>&1 | head -20

# Check file permissions
ls -la /etc/haproxy/haproxy.cfg

# Verify socket permissions
ls -la /var/lib/haproxy/
```

### High Connection Rates

```bash
# Check connection statistics
haproxyctl show stat | grep -E "(scur|scon|rate)"

# Monitor system resources
top -p $(pgrep haproxy)

# Check connection limits
ulimit -n

# Adjust max connections if needed
echo "global maxconn 10000" >> /etc/haproxy/haproxy.cfg
```

### SSL Issues

```bash
# Check certificate validity
openssl x509 -in /etc/haproxy/certs/cert.pem -noout -dates

# Verify certificate chain
openssl verify -CAfile /etc/haproxy/ca-bundle.crt /etc/haproxy/certs/cert.pem

# Test SSL handshake
openssl s_client -connect <haproxy_ip>:443 -servername <domain>

# Check SSL stats
curl -s "http://localhost:8400/stats;csv" | grep ssl
```

### Performance Issues

```bash
# Check response times
curl -s "http://localhost:8400/stats;csv" | grep -E "(time|rt)"

# Monitor CPU usage
iostat -x 1 | grep haproxy

# Check memory usage
ps aux | grep haproxy

# Analyze slow connections
haproxyctl show stat | grep -E "(slow|timeout)"
```

## Common Service Patterns

### Traefik Integration

HAProxy routes to Docker Swarm services via Traefik:

```bash
# Check Traefik backend health
haproxyctl show servers | grep traefik

# Verify routing to Swarm services
curl -H "Host: <service_domain>" http://<haproxy_ip>/path

# Monitor Traefik connection distribution
curl -s "http://localhost:8400/stats;csv" | grep traefik
```

### Database Load Balancing

HAProxy distributes database connections:

```bash
# Check database backend health
haproxyctl show servers | grep postgresql

# Monitor database connection pool
curl -s "http://localhost:8400/stats;csv" | grep -E "(db|sql)"

# Verify read/write splitting
grep -A 10 "backend db" /etc/haproxy/haproxy.cfg
```

### Multi-Service Routing

Different domains route to different backend pools:

```bash
# List all frontend configurations
grep "frontend" /etc/haproxy/haproxy.cfg

# Check ACL rules for routing
grep -A 5 "acl" /etc/haproxy/haproxy.cfg

# Monitor traffic per service
curl -s "http://localhost:8400/stats;csv" | grep FRONTEND
```

### Maintenance Operations

Safely perform maintenance on backend servers:

```bash
# Drain connections from backend
haproxyctl set server <backend>/<server> state drain

# Wait for connections to drop
watch -n 1 "haproxyctl show servers | grep <server>"

# Disable backend completely
haproxyctl disable server <backend>/<server>

# Re-enable after maintenance
haproxyctl enable server <backend>/<server>
```
