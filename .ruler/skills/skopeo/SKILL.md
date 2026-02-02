---
name: skopeo
description: Manages container registry operations for inspecting, copying, and managing container images without requiring Docker daemon access. Enables digest verification, metadata inspection, tag listing, and cross-registry image operations with secure credential handling via Vault.
---

# Skopeo Container Registry Skill

## Instructions

Use this skill to interact with container registries via `skopeo`. Skopeo enables inspection, copying, and management of container images without requiring Docker daemon access or pulling images locally. This is essential for registry operations, image verification, and cross-registry synchronization.

**Always retrieve credentials from Vault at runtime. NEVER hardcode registry credentials in code, logs, or environment files.**

### Configuration

The preferred method of interaction is the `skopeo` CLI tool (installed via Homebrew) with credentials securely retrieved from Vault.

**Smart Authentication:**
Before running commands, retrieve credentials from Vault for the target registry. Always use `--creds` flag to pass credentials securely.

#### Bjzy Labs Defaults

- **Container Registries**:
  - **Harbor (Bjzy Labs Private)**: `harbor.bjzy.me`
  - **Docker Hub**: `docker.io`
  - **Quay.io**: `quay.io`
  - **GitHub Container Registry**: `ghcr.io`

- **Vault Credential Paths**:
  - Harbor: `kvProd_v2/Harbor` - Fields: `username`, `password`
  - Docker Hub: `kvProd_v2/DockerHub` - Fields: `username`, `password`
  - Quay.io: `kvProd_v2/Quay` - Fields: `username`, `password`
  - GitHub Container Registry: `kvProd_v2/GitHub` - Fields: `username`, `token`

- **Harbor Specifics**:
  - **URL**: `harbor.bjzy.me`
  - **TLS**: Self-signed certificate, requires `--tls-verify=false`
  - **Projects**: `awx-custom`, `awx`, `infrastructure`, `applications`
  - **Notable Images**: `awx-custom-ee` (custom AWX Execution Environment)

### Environment and Guardrails (Bjzy Labs)

- **Registry Access**:
  - The agent must **NEVER** hardcode credentials in commands or scripts.
  - The agent must **ALWAYS** retrieve credentials from Vault using `vault kv get -field=`.
  - The agent must **NEVER** echo credentials to stdout or logs.
  - Credentials should be passed via `--creds` flag or environment variables.

- **TLS/Certificate Handling**:
  - **Harbor**: Use `--tls-verify=false` (self-signed cert).
  - **Public registries**: Use default `--tls-verify=true`.
  - Never disable certificate verification for production public registries.

- **Destructive Operations**:
  - Delete operations (`skopeo delete`) require explicit user confirmation.
  - Warn before deleting production image tags.
  - Log all delete operations for audit trail.

- **Credential Exposure Prevention**:
  - Always use `--creds` flag, not env vars in output.
  - Use `vault kv get -field=` to extract only needed fields.
  - Never store credentials in shell history.

- **Hostname Safety**:
  - Distinguish between dev and prod registries (e.g., `harbor.bjzy.me` for prod).
  - Verify registry URL before executing copy or delete operations.

### Standard Operating Procedure (SOP)

When asked to "Inspect an image," "Copy an image," "List tags," or "Verify a digest":

1. **Identify Registry**: Determine the target registry (Harbor, Docker Hub, etc.).
2. **Retrieve Credentials**: Use `vault kv get -field=` to fetch username and password from appropriate Vault path.
3. **Choose Operation**:
   - **Inspect**: Read-only, safe to execute immediately.
   - **List Tags**: Read-only, safe to execute immediately.
   - **Copy/Retag**: Requires confirmation, especially for production images.
   - **Delete**: Destructive, requires explicit user confirmation.
4. **Execute & Verify**: Run the operation and validate results.
5. **Document**: If performing production operations, document changes in Keep or ticket system.

## Examples

### 1. Inspect Image Metadata (Read-Only)

Get comprehensive metadata about a container image without pulling it locally.

- **Method:** `skopeo inspect` with Vault credentials
- **Use Case**: Verify image layers, labels, creation date, digest
- **Command Pattern:**

```bash
# Retrieve credentials from Vault
HARBOR_USER=$(vault kv get -field=username kvProd_v2/Harbor)
HARBOR_PASS=$(vault kv get -field=password kvProd_v2/Harbor)

# Inspect image metadata (Harbor example with self-signed cert)
skopeo inspect --tls-verify=false \
  --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/awx-custom/awx-custom-ee:1.0.17

# Output includes:
# - Digest: sha256:e157bfadc19d46617ae46c7c819bd3d287923e7d63761bfbb09523211dc4f840
# - Layers: [list of layer digests]
# - Labels: [image labels]
# - Created: 2025-02-23T10:30:45Z
```

### 2. Verify Image Digest (Security Check)

Verify that an image tag points to the expected digest, critical for security and reproducibility.

- **Method:** `skopeo inspect` with digest extraction and comparison
- **Use Case**: Verify AWX EE image integrity, confirm version consistency
- **Command Pattern:**

```bash
# Retrieve credentials
HARBOR_USER=$(vault kv get -field=username kvProd_v2/Harbor)
HARBOR_PASS=$(vault kv get -field=password kvProd_v2/Harbor)

# Define expected digest
EXPECTED_DIGEST="sha256:e157bfadc19d46617ae46c7c819bd3d287923e7d63761bfbb09523211dc4f840"

# Get actual digest from registry
ACTUAL_DIGEST=$(skopeo inspect --tls-verify=false \
  --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/awx-custom/awx-custom-ee:1.0.17 | \
  jq -r '.Digest')

# Verify match
if [ "${ACTUAL_DIGEST}" = "${EXPECTED_DIGEST}" ]; then
  echo "✓ Digest matches: ${ACTUAL_DIGEST}"
else
  echo "✗ DIGEST MISMATCH!"
  echo "  Expected: ${EXPECTED_DIGEST}"
  echo "  Actual:   ${ACTUAL_DIGEST}"
  exit 1
fi
```

### 3. List All Available Tags for a Repository (Read-Only)

List all available tags for a repository to determine next version number or find available versions.

- **Method:** `skopeo list-tags` with Vault credentials
- **Use Case**: Find next version for AWX EE, check available versions
- **Command Pattern:**

```bash
# Retrieve credentials
HARBOR_USER=$(vault kv get -field=username kvProd_v2/Harbor)
HARBOR_PASS=$(vault kv get -field=password kvProd_v2/Harbor)

# List all tags for a repository
skopeo list-tags --tls-verify=false \
  --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/awx-custom/awx-custom-ee

# Output example:
# {
#   "Repository": "harbor.bjzy.me/awx-custom/awx-custom-ee",
#   "Tags": ["1.0.15", "1.0.16", "1.0.17", "latest", "dev"]
# }
```

### 4. Copy/Retag Image (Write Operation)

Copy an image from one tag to another within the same registry or between registries.

- **Method:** `skopeo copy` with credentials
- **Use Case**: Retag a version as `latest`, promote image between environments
- **Risk Level**: Write operation - requires confirmation
- **Command Pattern:**

```bash
# Retrieve credentials for source and destination registries
HARBOR_USER=$(vault kv get -field=username kvProd_v2/Harbor)
HARBOR_PASS=$(vault kv get -field=password kvProd_v2/Harbor)

# Copy image to new tag (same registry)
skopeo copy --tls-verify=false \
  --src-creds "${HARBOR_USER}:${HARBOR_PASS}" \
  --dest-creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/awx-custom/awx-custom-ee:1.0.17 \
  docker://harbor.bjzy.me/awx-custom/awx-custom-ee:latest

# Cross-registry copy (Harbor to Docker Hub example)
# Retrieve both credentials
DOCKER_HUB_USER=$(vault kv get -field=username kvProd_v2/DockerHub)
DOCKER_HUB_PASS=$(vault kv get -field=password kvProd_v2/DockerHub)

skopeo copy --tls-verify=false \
  --src-creds "${HARBOR_USER}:${HARBOR_PASS}" \
  --dest-creds "${DOCKER_HUB_USER}:${DOCKER_HUB_PASS}" \
  docker://harbor.bjzy.me/awx-custom/awx-custom-ee:1.0.17 \
  docker://docker.io/myorg/awx-custom-ee:1.0.17
```

### 5. Get Image Layer Information (Read-Only)

Inspect detailed layer information for an image, useful for understanding image composition.

- **Method:** `skopeo inspect` with jq for layer parsing
- **Use Case**: Analyze image layers, understand base OS and packages
- **Command Pattern:**

```bash
# Retrieve credentials
HARBOR_USER=$(vault kv get -field=username kvProd_v2/Harbor)
HARBOR_PASS=$(vault kv get -field=password kvProd_v2/Harbor)

# Get layer information formatted
skopeo inspect --tls-verify=false \
  --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/awx-custom/awx-custom-ee:1.0.17 | \
  jq '.Layers[] | {size: .Size, digest: .Digest}' | head -20

# Get layer count
LAYER_COUNT=$(skopeo inspect --tls-verify=false \
  --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/awx-custom/awx-custom-ee:1.0.17 | \
  jq '.Layers | length')

echo "Image has ${LAYER_COUNT} layers"
```

### 6. Extract Image Labels (Read-Only)

Extract specific labels from an image to verify metadata like version, maintainer, build date.

- **Method:** `skopeo inspect` with jq for label extraction
- **Use Case**: Verify image labels match expected values
- **Command Pattern:**

```bash
# Retrieve credentials
HARBOR_USER=$(vault kv get -field=username kvProd_v2/Harbor)
HARBOR_PASS=$(vault kv get -field=password kvProd_v2/Harbor)

# Extract all labels
skopeo inspect --tls-verify=false \
  --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/awx-custom/awx-custom-ee:1.0.17 | \
  jq '.Labels'

# Extract specific label
VERSION_LABEL=$(skopeo inspect --tls-verify=false \
  --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/awx-custom/awx-custom-ee:1.0.17 | \
  jq -r '.Labels.version // "unknown"')

echo "Image version: ${VERSION_LABEL}"
```

### 7. Check Image Size Information (Read-Only)

Get image size and layer information without pulling the image.

- **Method:** `skopeo inspect` with jq for size calculation
- **Use Case**: Verify image size before pulling, understand bandwidth requirements
- **Command Pattern:**

```bash
# Retrieve credentials
HARBOR_USER=$(vault kv get -field=username kvProd_v2/Harbor)
HARBOR_PASS=$(vault kv get -field=password kvProd_v2/Harbor)

# Get compressed and uncompressed sizes
skopeo inspect --tls-verify=false \
  --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/awx-custom/awx-custom-ee:1.0.17 | \
  jq '{
    manifest_size: .Size,
    config_size: .Config.Size,
    layers: (.Layers | map(.Size) | add)
  }'

# Total uncompressed size calculation
TOTAL_SIZE=$(skopeo inspect --tls-verify=false \
  --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/awx-custom/awx-custom-ee:1.0.17 | \
  jq '.Layers | map(.Size) | add')

echo "Uncompressed image size: $(numfmt --to=iec-i --suffix=B ${TOTAL_SIZE} 2>/dev/null || echo ${TOTAL_SIZE})"
```

### 8. Delete Image Tag (Destructive Operation)

Delete a specific image tag from a registry. **Requires explicit confirmation.**

- **Method:** `skopeo delete` with credentials
- **Use Case**: Remove old versions, clean up development tags
- **Risk Level**: Destructive - **requires explicit user confirmation**
- **Command Pattern:**

```bash
# Retrieve credentials
HARBOR_USER=$(vault kv get -field=username kvProd_v2/Harbor)
HARBOR_PASS=$(vault kv get -field=password kvProd_v2/Harbor)

# Confirm target before deletion
echo "Deleting tag 'dev' from harbor.bjzy.me/awx-custom/awx-custom-ee"
echo "This action cannot be undone."
read -p "Continue? (type 'yes' to confirm): " CONFIRM

if [ "${CONFIRM}" = "yes" ]; then
  skopeo delete --tls-verify=false \
    --creds "${HARBOR_USER}:${HARBOR_PASS}" \
    docker://harbor.bjzy.me/awx-custom/awx-custom-ee:dev
  echo "✓ Tag deleted successfully"
else
  echo "✗ Deletion cancelled"
  exit 1
fi
```

### 9. Get Image Architecture and OS Information (Read-Only)

Verify image architecture (amd64, arm64) and operating system information.

- **Method:** `skopeo inspect --raw` with jq for manifest parsing
- **Use Case**: Verify image compatibility with target architecture
- **Command Pattern:**

```bash
# Retrieve credentials
HARBOR_USER=$(vault kv get -field=username kvProd_v2/Harbor)
HARBOR_PASS=$(vault kv get -field=password kvProd_v2/Harbor)

# Get architecture and OS
skopeo inspect --tls-verify=false \
  --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/awx-custom/awx-custom-ee:1.0.17 | \
  jq '{
    arch: .Architecture,
    os: .Os,
    os_version: .OsVersion,
    variant: .Variant
  }'

# Check for multi-arch manifests
skopeo inspect --tls-verify=false \
  --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/awx-custom/awx-custom-ee:latest | \
  jq '.Architectures // "Single-arch"'
```

### 10. Sync Repository Between Registries (Write Operation)

Synchronize all images in a repository between registries.

- **Method:** `skopeo sync` with credentials
- **Use Case**: Mirror Harbor repository to backup registry, promote to production
- **Risk Level**: Write operation - **requires confirmation**
- **Command Pattern:**

```bash
# Retrieve credentials for both registries
HARBOR_USER=$(vault kv get -field=username kvProd_v2/Harbor)
HARBOR_PASS=$(vault kv get -field=password kvProd_v2/Harbor)

DOCKER_HUB_USER=$(vault kv get -field=username kvProd_v2/DockerHub)
DOCKER_HUB_PASS=$(vault kv get -field=password kvProd_v2/DockerHub)

# Sync specific repository (Harbor to Docker Hub)
echo "Syncing harbor.bjzy.me/awx-custom to Docker Hub"
read -p "Continue? (type 'yes' to confirm): " CONFIRM

if [ "${CONFIRM}" = "yes" ]; then
  skopeo sync --tls-verify=false \
    --src-creds "${HARBOR_USER}:${HARBOR_PASS}" \
    --dest-creds "${DOCKER_HUB_USER}:${DOCKER_HUB_PASS}" \
    --src docker --dest docker \
    harbor.bjzy.me/awx-custom docker.io/myorg/awx-custom
  echo "✓ Repository synced successfully"
fi
```

### 11. Get Image Manifest (Read-Only)

Retrieve the raw manifest of an image for detailed analysis.

- **Method:** `skopeo inspect --raw` with optional jq formatting
- **Use Case**: Analyze image structure, verify manifest format
- **Command Pattern:**

```bash
# Retrieve credentials
HARBOR_USER=$(vault kv get -field=username kvProd_v2/Harbor)
HARBOR_PASS=$(vault kv get -field=password kvProd_v2/Harbor)

# Get raw manifest (pretty-printed)
skopeo inspect --tls-verify=false \
  --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  --raw docker://harbor.bjzy.me/awx-custom/awx-custom-ee:1.0.17 | jq '.'

# Check manifest media type
MANIFEST_TYPE=$(skopeo inspect --tls-verify=false \
  --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  --raw docker://harbor.bjzy.me/awx-custom/awx-custom-ee:1.0.17 | \
  jq -r '.mediaType')

echo "Manifest type: ${MANIFEST_TYPE}"
```

### 12. Compare Two Image Digests (Read-Only)

Compare digests of the same image tag across different registries to verify consistency.

- **Method:** Multiple `skopeo inspect` calls with digest comparison
- **Use Case**: Verify image consistency between Harbor and backup registry
- **Command Pattern:**

```bash
# Retrieve credentials
HARBOR_USER=$(vault kv get -field=username kvProd_v2/Harbor)
HARBOR_PASS=$(vault kv get -field=password kvProd_v2/Harbor)

DOCKER_HUB_USER=$(vault kv get -field=username kvProd_v2/DockerHub)
DOCKER_HUB_PASS=$(vault kv get -field=password kvProd_v2/DockerHub)

# Get digest from Harbor
HARBOR_DIGEST=$(skopeo inspect --tls-verify=false \
  --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/awx-custom/awx-custom-ee:1.0.17 | \
  jq -r '.Digest')

# Get digest from Docker Hub
DOCKER_HUB_DIGEST=$(skopeo inspect \
  --creds "${DOCKER_HUB_USER}:${DOCKER_HUB_PASS}" \
  docker://docker.io/myorg/awx-custom-ee:1.0.17 | \
  jq -r '.Digest')

# Compare
if [ "${HARBOR_DIGEST}" = "${DOCKER_HUB_DIGEST}" ]; then
  echo "✓ Images are identical"
  echo "  Digest: ${HARBOR_DIGEST}"
else
  echo "✗ Image digests do not match!"
  echo "  Harbor: ${HARBOR_DIGEST}"
  echo "  Docker Hub: ${DOCKER_HUB_DIGEST}"
fi
```

### 13. Public Registry Operations (Docker Hub, Quay.io)

Access public registries without authentication or with personal tokens.

- **Method:** `skopeo inspect` without credentials (public) or with credentials (private)
- **Use Case**: Check public image availability, inspect base images
- **Command Pattern:**

```bash
# No credentials needed for public images
skopeo inspect docker://docker.io/library/centos:stream9

# With authentication (if using private Docker Hub repos)
DOCKER_HUB_USER=$(vault kv get -field=username kvProd_v2/DockerHub)
DOCKER_HUB_PASS=$(vault kv get -field=password kvProd_v2/DockerHub)

skopeo inspect \
  --creds "${DOCKER_HUB_USER}:${DOCKER_HUB_PASS}" \
  docker://docker.io/myorg/private-image:latest

# Quay.io (Red Hat registry)
QUAY_USER=$(vault kv get -field=username kvProd_v2/Quay)
QUAY_PASS=$(vault kv get -field=password kvProd_v2/Quay)

skopeo inspect \
  --creds "${QUAY_USER}:${QUAY_PASS}" \
  docker://quay.io/myorg/image:latest
```

## Troubleshooting

### Authentication Failed

```bash
# Verify credentials from Vault
vault kv get kvProd_v2/Harbor

# Test credential validity
HARBOR_USER=$(vault kv get -field=username kvProd_v2/Harbor)
HARBOR_PASS=$(vault kv get -field=password kvProd_v2/Harbor)

# Try with verbose error output
skopeo inspect --tls-verify=false \
  --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/awx-custom/awx-custom-ee:latest

# If auth fails: check if credentials are valid in Vault
# Contact Harbor admin if credentials are expired
```

### Certificate Verification Errors

```bash
# Harbor uses self-signed cert, always use --tls-verify=false
skopeo inspect --tls-verify=false \
  --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/awx-custom/awx-custom-ee:latest

# For public registries, do NOT disable cert verification
# If cert issues occur on public registries, system CA trust may need update
```

### Image Not Found

```bash
# Verify registry URL and path are correct
skopeo list-tags --tls-verify=false \
  --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/awx-custom/awx-custom-ee

# If repository not found, check permissions
# Robot accounts may have restricted project access
```

### Network Connectivity Issues

```bash
# Verify connectivity to registry
curl -k https://harbor.bjzy.me/api/v2/health

# Check if Harbor is accessible
ssh ansible@huey.bjzy.me "curl -k https://harbor.bjzy.me/api/v2/health"

# Check DNS resolution
nslookup harbor.bjzy.me
```

### Copy Operation Fails

```bash
# Verify source image exists
skopeo inspect --tls-verify=false \
  --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/awx-custom/awx-custom-ee:1.0.17

# Verify destination credentials have write permission
# Check that destination repository exists

# Retry with verbose output
skopeo copy --debug --tls-verify=false \
  --src-creds "${HARBOR_USER}:${HARBOR_PASS}" \
  --dest-creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/awx-custom/awx-custom-ee:1.0.17 \
  docker://harbor.bjzy.me/awx-custom/awx-custom-ee:latest
```

### Skopeo Command Not Found

```bash
# Install skopeo on macOS
brew install skopeo

# Verify installation
skopeo --version
# Should show: skopeo version 1.x.x or higher
```

## Quick Reference Commands

### Inspection (Read-Only, Safe)
```bash
# Inspect image
HARBOR_USER=$(vault kv get -field=username kvProd_v2/Harbor)
HARBOR_PASS=$(vault kv get -field=password kvProd_v2/Harbor)
skopeo inspect --tls-verify=false --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/project/image:tag

# List tags
skopeo list-tags --tls-verify=false --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/project/image

# Get digest
skopeo inspect --tls-verify=false --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/project/image:tag | jq -r '.Digest'
```

### Copy/Retag (Write, Requires Confirmation)
```bash
# Retag image
skopeo copy --tls-verify=false \
  --src-creds "${HARBOR_USER}:${HARBOR_PASS}" \
  --dest-creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/project/image:old-tag \
  docker://harbor.bjzy.me/project/image:new-tag
```

### Delete (Destructive, Requires Confirmation)
```bash
# Delete tag
skopeo delete --tls-verify=false \
  --creds "${HARBOR_USER}:${HARBOR_PASS}" \
  docker://harbor.bjzy.me/project/image:tag
```

## Important Notes

- **Never run destructive commands** (`skopeo delete`) without explicit user confirmation.
- **Always verify registry and image names** before executing copy or delete operations.
- **Use `--tls-verify=false` only for Harbor**; do not disable cert verification for public registries.
- **Retrieve credentials from Vault**, never hardcode them in scripts or environment.
- **Document all production operations** for audit trail and troubleshooting.
- **Test with non-production images first** before performing operations on production images.

## Related Documentation

- **Skopeo Documentation**: [containers/skopeo GitHub](https://github.com/containers/skopeo)
- **Harbor API**: [Bjzy Labs Harbor Registry](https://harbor.bjzy.me)
- **Vault Skill**: See `vault-cli` skill for Vault operations
- **Docker Swarm Skill**: See `docker-swarm` skill for container orchestration
- **GitHub Issue**: [#2 - Skopeo Skill Implementation](https://github.com/BjzyLabs/ansible_builder/issues/2)

## Dependencies

### Required Tools
- `skopeo` (installed via `brew install skopeo`)
- `vault` CLI (for credential retrieval)
- `jq` (for JSON parsing)
- `curl` (for registry API calls)

### Version Requirements
- skopeo 1.x+
- Vault CLI with valid authentication
- jq 1.6+

## Context from EE Upgrade

This skill was requested during the AWX Execution Environment upgrade (v1.0.16 → v1.0.17). Operations that motivated this skill:

1. **Digest Verification**: Verify that image tags in AWX match digests in Harbor registry
2. **Tag Listing**: Determine available versions for planning next release
3. **Image Metadata**: Inspect base OS, layers, creation date without pulling image
4. **Image Retagging**: Promote images between tags (e.g., version tag → `latest`)
5. **Cross-Registry Operations**: Mirror images between Harbor and other registries

---

**Labels**: enhancement, skill, infrastructure, container-registry
**Related Skill**: `vault-cli` (credential management), `docker-swarm` (container orchestration)
