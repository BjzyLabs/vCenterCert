---
name: microceph
description: Manages MicroCeph distributed storage cluster for block storage, object storage, OSD operations, pool configuration, CRUSH map management, and cluster health monitoring. Use when user mentions Ceph, MicroCeph, storage cluster, OSDs, pools, CRUSH, or distributed storage.
---

# MicroCeph Distributed Storage Integration

## Instructions

Use this skill to interact with MicroCeph distributed storage clusters. MicroCeph provides distributed block storage (RBD) and file storage (CephFS) for Docker Swarm containers and other services in the Home Lab. **Always prefer using AWX Job Templates for cluster operations over manual microceph commands.**

### Configuration

The preferred method of interaction is SSH to MicroCeph nodes or using AWX job templates for orchestration tasks.

**Smart Authentication:**
MicroCeph requires sudo access on cluster nodes. AWX operations use AppRole authentication for Vault secrets.

```bash
# For AWX operations, verify token validity
# Note: Ensure AWX CA certificate is trusted in system trust store
if ! awx me > /dev/null 2>&1; then
  export TOWER_HOST=https://awx.bjzy.me
  export TOWER_OAUTH_TOKEN=$(vault kv get -field=api_token kvProd_v2/AWX/API)
fi

# For direct SSH operations, verify connectivity
ssh -o ConnectTimeout=5 ansible@huey.bjzy.me "sudo microceph.ceph -s" > /dev/null 2>&1 || echo "SSH connectivity issue"
```

### CLI vs API Decision Tree

**MicroCeph operations are CLI-first via microceph command wrapper.**

| Operation | Use SSH + microceph CLI | Use AWX |
|-----------|------------------------|---------|
| Check cluster status | ✅ `microceph.ceph -s` | ❌ |
| View OSD status | ✅ `microceph.ceph osd tree` | ❌ |
| Add OSD | ❌ Use AWX | ✅ Preferred |
| Create pools | ❌ Use AWX | ✅ Preferred |
| Remove OSD | ❌ Use AWX | ✅ Preferred |
| Check disk usage | ✅ `microceph.ceph df` | ❌ |
| Cluster bootstrap | ❌ Never manual | ✅ AWX only |

**Rule:** Read-only operations via SSH. Write operations through AWX.

**Smart Access Pattern:****
Before running commands, determine if the operation should be executed via AWX (cluster changes, OSD operations) or direct SSH (read-only verification, health checks).

#### Bjzy Labs defaults

- **MicroCeph Clusters**:
  - **Production**: 3-node cluster (Huey, Dewey, Louie)
  - **Development**: 3-node cluster (devHuey, devDewey, devLouie)
- **AWX Integration**:
  - Job Template: "MicroCeph Management" (playbook: `microceph-manage.yml`)
  - Operations: install, status, config, uninstall, uninstall-purge
- **Vault lookup (for credentials if needed)**:
  - Path: `kvProd_v2/MicroCeph/` (cluster secrets)

### Environment and Guardrails (Bjzy Labs)

- **Managed Environments**:
  - **Production Cluster**:
    - **Nodes**: Huey (192.168.60.81), Dewey (192.168.60.82), Louie (192.168.60.83)
    - **Hostnames**: huey.bjzy.me, dewey.bjzy.me, louie.bjzy.me
    - **Network**: 192.168.60.0/24 (smDSwitch-ProdLAN)
    - **Storage**: Each node has /dev/sda3 (140GB RAW) for OSD storage
    - **Pool**: `docker-volumes` (RBD, size=3, pg_num=64)
  - **Development Cluster**:
    - **Nodes**: devHuey (192.168.50.81), devDewey (192.168.50.82), devLouie (192.168.50.83)
    - **Hostnames**: devhuey.bjzy.me, devdewey.bjzy.me, devlouie.bjzy.me
    - **Network**: 192.168.50.0/24 (smDSwitch-DevLAN)
    - **Storage**: Each node has /dev/sda3 (140GB RAW) for OSD storage

- **Execution Rules**:
  - The agent must **NEVER** run `microceph cluster bootstrap` manually; use AWX job templates.
  - The agent must **NEVER** modify OSD configuration directly; changes must be made via AWX.
  - **SSH is allowed for read-only verification and troubleshooting**.
  - **Hostname safety**: `devHuey` != `Huey` (dev vs prod are different clusters).

- **Storage Architecture**:
  - **OSD Storage**: Raw partitions (/dev/sda3) - 140GB per node
  - **System Storage**: LVM partitions for OS, Docker, swap
  - **Replication**: 3x replication across all nodes
  - **Usable Capacity**: ~140GB total (with 3x replication)

### Standard Operating Procedure (SOP)

When asked to "Check MicroCeph status," "Manage storage," or "Troubleshoot cluster":

1. **Determine Environment**: Identify if this is Production or Development cluster.
2. **Choose Method**:
   - **Cluster Changes**: Use AWX Job Template "MicroCeph Management".
   - **Read-Only Checks**: SSH directly to any node.
3. **Verify Access**: For SSH operations, use `ssh ansible@<hostname>` (e.g., `ssh ansible@huey.bjzy.me`).
4. **Execute & Monitor**: Run the operation and verify results.
5. **Document**: If issues are found, consider enriching Keep alerts or updating documentation.

## Examples

### 1. Check Cluster Health (Read-Only)

Use this to verify cluster status, OSD health, and storage capacity.

- **Method:** SSH to any node
- **Command Pattern:**

```bash
# Check cluster status (run on any node)
ssh ansible@huey.bjzy.me "sudo microceph.ceph -s"

# Check detailed cluster health
ssh ansible@huey.bjzy.me "sudo microceph.eph health detail"

# Check OSD tree and status
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd tree"

# Check monitor status
ssh ansible@huey.bjzy.me "sudo microceph.ceph quorum_status"
```

### 2. Monitor Storage Usage (Read-Only)

View storage utilization and pool statistics.

- **Method:** SSH to any node
- **Command Pattern:**

```bash
# Check pool statistics
ssh ansible@huey.bjzy.me "sudo microceph.ceph df"

# Check OSD usage details
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd df"

# View pool configuration
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd pool ls detail"

# Check available storage
ssh ansible@huey.bjzy.me "sudo microceph.ceph df tree"
```

### 3. Inspect OSD Operations (Read-Only)

Monitor OSD performance and status.

- **Method:** SSH to any node
- **Command Pattern:**

```bash
# Check OSD performance
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd perf"

# View OSD dump (detailed configuration)
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd dump"

# Check OSD utilization histogram
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd utilization"

# Monitor OSD pings (latency)
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd ping"
```

### 4. Pool Management via AWX (Preferred Method)

Use AWX to manage storage pools and configurations.

- **Method:** AWX CLI
- **Command Pattern:**

```bash
# Launch MicroCeph management job
awx job_template launch "MicroCeph Management" \
  --extra_vars '{
    "operation": "status",
    "environment": "production"
  }' \
  --monitor

# Install or configure cluster
awx job_template launch "MicroCeph Management" \
  --extra_vars '{
    "operation": "install",
    "environment": "production"
  }' \
  --monitor
```

### 5. Check CephFS Status (Read-Only)

Verify CephFS file system status if configured.

- **Method:** SSH to any node
- **Command Pattern:**

```bash
# Check CephFS status
ssh ansible@huey.bjzy.me "sudo microceph.ceph fs ls"

# Check MDS status (if CephFS is used)
ssh ansible@huey.bjzy.me "sudo microceph.ceph mds stat"

# View CephFS clients
ssh ansible@huey.bjzy.me "sudo microceph.ceph fs dump"
```

### 6. Monitor Network and Performance (Read-Only)

Check cluster network performance and metrics.

- **Method:** SSH to any node
- **Command Pattern:**

```bash
# Check network connectivity between nodes
ssh ansible@huey.bjzy.me "sudo microceph.ceph network stat"

# Monitor cluster-wide performance
ssh ansible@huey.bjzy.me "sudo microceph.ceph perf dump"

# Check recovery operations
ssh ansible@huey.bjzy.me "sudo microceph.ceph pg dump"

# View placement group statistics
ssh ansible@huey.bjzy.me "sudo microceph.ceph pg stat"
```

### 7. Verify Docker Integration (Read-Only)

Check Docker RBD plugin and integration status.

- **Method:** SSH to any node
- **Command Pattern:**

```bash
# Check Docker RBD plugin status
ssh ansible@huey.bjzy.me "docker plugin ls | grep rbd"

# Verify Ceph configuration files
ssh ansible@huey.bjzy.me "ls -la /etc/ceph/"

# Check admin keyring
ssh ansible@huey.bjzy.me "sudo cat /etc/ceph/ceph.client.admin.keyring"

# Test Ceph connectivity
ssh ansible@huey.bjzy.me "sudo microceph.ceph -s"
```

### 8. Cluster Configuration Verification (Read-Only)

Verify cluster configuration and settings.

- **Method:** SSH to any node
- **Command Pattern:**

```bash
# View cluster configuration
ssh ansible@huey.bjzy.me "sudo microceph.ceph config dump"

# Check monitor configuration
ssh ansible@huey.bjzy.me "sudo microceph.ceph mon dump"

# View cluster map
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd map docker-volumes test-object"

# Check authentication status
ssh ansible@huey.bjzy.me "sudo microceph.ceph auth list"
```

### 9. Troubleshooting OSD Issues (Read-Only)

Diagnose OSD problems and failures.

- **Method:** SSH to any node
- **Command Pattern:**

```bash
# Check OSD logs for errors
ssh ansible@huey.bjzy.me "sudo journalctl -u microceph -n 100"

# Check specific OSD status
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd status 0"

# View OSD utilization and rebalancing
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd utilization"

# Check for down OSDs
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd tree | grep -i down"
```

### 10. Storage Validation (Read-Only)

Validate storage layout and device status.

- **Method:** SSH to specific node
- **Command Pattern:**

```bash
# Check partition layout (verify /dev/sda3 is raw)
ssh ansible@huey.bjzy.me "lsblk"

# Verify /dev/sda3 has no filesystem
ssh ansible@huey.bjzy.me "sudo blkid /dev/sda3"

# Check LVM layout
ssh ansible@huey.bjzy.me "sudo vgs && sudo lvs"

# Verify MicroCeph can access OSD devices
ssh ansible@huey.bjzy.me "sudo microceph.ceph-volume lvm list"
```

### 11. OSD Management Operations

Manage individual OSDs (Object Storage Daemons).

- **Method:** SSH to node + sudo microceph commands
- **Purpose:** Maintain and troubleshoot storage daemons

**Command Pattern:**
```bash
# List all OSDs with status
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd tree"

# Check specific OSD details
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd dump | grep 'osd.0'"

# View OSD performance stats
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd perf"

# Mark OSD down (maintenance)
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd down 0"

# Mark OSD out (rebalance data)
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd out 0"

# Mark OSD back in
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd in 0"

# Check OSD smart data
ssh ansible@huey.bjzy.me "sudo smartctl -a /dev/sda"
```

### 12. Pool Operations

Create and manage Ceph storage pools.

- **Method:** SSH to any node
- **Purpose:** Organize storage with different replication/performance characteristics

**Command Pattern:**
```bash
# List all pools
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd pool ls detail"

# Get pool statistics
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd pool stats"

# Check pool usage
ssh ansible@huey.bjzy.me "sudo microceph.ceph df"

# View pool configuration
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd pool get docker-volumes all"

# Set pool parameters (use AWX for production)
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd pool set docker-volumes size 3"
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd pool set docker-volumes min_size 2"

# Check PG (placement group) status
ssh ansible@huey.bjzy.me "sudo microceph.ceph pg dump pgs"

# View pool autoscaler status
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd pool autoscale-status"
```

## Troubleshooting

### Cluster Health Warnings

```bash
# Check detailed health warnings
ssh ansible@huey.bjzy.me "sudo microceph.ceph health detail"

# Check for placement group issues
ssh ansible@huey.bjzy.me "sudo microceph.ceph pg dump | grep -i stuck"

# Monitor recovery progress
ssh ansible@huey.bjzy.me "sudo microceph.ceph -w"

# Check cluster-wide events
ssh ansible@huey.bjzy.me "sudo microceph.ceph log last 100"
```

### OSD Down or Out

```bash
# Check OSD status
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd tree"

# Check specific OSD details
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd dump | grep -A 10 'osd.0'"

# View OSD logs
ssh ansible@huey.bjzy.me "sudo journalctl -u snap.microceph.daemon-osd.0 -n 50"

# Check device status
ssh ansible@huey.bjzy.me "sudo lsblk /dev/sda3"
```

### Network Connectivity Issues

```bash
# Check monitor connectivity
ssh ansible@huey.bjzy.me "sudo microceph.ceph quorum_status"

# Test network between nodes
ssh ansible@huey.bjzy.me "ping -c 3 dewey.bjzy.me"
ssh ansible@huey.bjzy.me "ping -c 3 louie.bjzy.me"

# Check firewall status
ssh ansible@huey.bjzy.me "sudo ufw status"

# Monitor network stats
ssh ansible@huey.bjzy.me "sudo microceph.ceph network stat"
```

### Storage Capacity Issues

```bash
# Check overall storage usage
ssh ansible@huey.bjzy.me "sudo microceph.ceph df"

# Check OSD utilization
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd df"

# Monitor pool usage
ssh ansible@huey.bjzy.me "sudo microceph.ceph rbd du docker-volumes"

# Check for near-full OSDs
ssh ansible@huey.bjzy.me "sudo microceph.ceph health detail | grep -i near"
```

### Docker Integration Issues

```bash
# Check Docker RBD plugin
ssh ansible@huey.bjzy.me "docker plugin inspect rbd"

# Verify Ceph configuration
ssh ansible@huey.bjzy.me "sudo cat /etc/ceph/ceph.conf"

# Test RBD volume creation
ssh ansible@huey.bjzy.me "sudo microceph.rbd create test --size 1G --pool docker-volumes"

# Clean up test volume
ssh ansible@huey.bjzy.me "sudo microceph.rbd rm test --pool docker-volumes"
```

## Quick Reference Commands

### Cluster Status (Read-Only)
```bash
# Overall cluster health
ssh ansible@huey.bjzy.me "sudo microceph.ceph -s"

# Detailed health
ssh ansible@huey.bjzy.me "sudo microceph.ceph health detail"

# OSD tree
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd tree"

# Pool statistics
ssh ansible@huey.bjzy.me "sudo microceph.ceph df"
```

### Storage Operations (Read-Only)
```bash
# Pool details
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd pool ls detail"

# OSD utilization
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd df"

# Storage usage tree
ssh ansible@huey.bjzy.me "sudo microceph.ceph df tree"
```

### Performance Monitoring (Read-Only)
```bash
# Performance dump
ssh ansible@huey.bjzy.me "sudo microceph.ceph perf dump"

# OSD performance
ssh ansible@huey.bjzy.me "sudo microceph.ceph osd perf"

# Network statistics
ssh ansible@huey.bjzy.me "sudo microceph.ceph network stat"
```

### Docker Integration (Read-Only)
```bash
# Docker plugin status
ssh ansible@huey.bjzy.me "docker plugin ls | grep rbd"

# Ceph config files
ssh ansible@huey.bjzy.me "ls -la /etc/ceph/"

# Test connectivity
ssh ansible@huey.bjzy.me "sudo microceph.ceph -s"
```

## Important Notes

- **Never run destructive commands** (`microceph cluster destroy`, `osd out`, `osd destroy`) without explicit user confirmation.
- **Always verify environment** (Production vs Development) before executing commands.
- **Use AWX for cluster changes** - Direct microceph commands should only be used for read-only verification.
- **Check Keep alerts first** - Many MicroCeph issues may already have alerts in Keep AIOps.
- **Document findings** - If you discover issues, enrich Keep alerts or update Notion documentation.

## Related Documentation

- **Notion**: [MicroCeph Distributed Storage — Ubuntu Installation Guide](https://www.notion.so/2823569aa25581479351cadcbdd3d68a)
- **Notion**: [MicroCeph Cluster Implementation - Feature Branch Development](https://www.notion.so/2863569aa25581f89b12ce42812f8c5e)
- **GitHub Repo**: `homelab-playbooks` (BjzyLabs/ansible-homelab)
- **Playbook**: `playbooks/microceph-manage.yml`
- **Local Docs**: `docs/MICROCEPH_IMPLEMENTATION_GUIDE.md`
