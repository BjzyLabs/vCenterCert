---
name: kubectl
description: Interacts with Kubernetes clusters for resource management, troubleshooting, deployments, scaling, and operational tasks including pods, services, configmaps, and secrets. Use when user mentions Kubernetes, k8s, pods, deployments, services, namespaces, or container orchestration.
---

# Kubernetes (kubectl) Integration

## Instructions

Use this skill to interact with Kubernetes clusters using the `kubectl` CLI tool. Kubernetes provides container orchestration, automated deployment, scaling, and management of containerized applications in the Home Lab.
**Always verify the current context and namespace before executing cluster operations.**

### Configuration

The preferred method of interaction is SSH to Kubernetes nodes or direct kubectl commands with proper kubeconfig. kubectl requires proper authentication and context configuration.

**Smart Access Pattern:**
- **Cluster Operations**: Use kubectl commands with proper context
- **Node Access**: SSH to master/control plane nodes for direct access
- **Configuration**: Verify kubeconfig and context before operations

#### Bjzy Labs defaults

- **Kubernetes Clusters**:
  - **Production**: 3-node cluster (control plane + workers)
  - **Development**: Single-node or 3-node cluster for testing
- **Common use cases**:
  - **Pod Management**: Deploy, scale, and troubleshoot pods
  - **Service Management**: Expose applications via services and ingress
  - **Configuration**: Manage ConfigMaps, Secrets, and CRDs
  - **Monitoring**: Check cluster health, resource usage, and events
- **Integration Points**:
  - **Docker Swarm**: Some workloads may run on both platforms
  - **Vault**: Kubernetes secrets integration
  - **Monitoring**: Prometheus/Grafana for cluster metrics
  - **Logging**: Loki for pod and application logs

### Environment and Guardrails (Bjzy Labs)

- **Managed Environments**:
  - **Production Cluster**:
    - **Control Plane**: kubernetes-control-plane nodes
    - **Worker Nodes**: kubernetes-worker nodes
    - **Network**: 192.168.70.0/24 (Kubernetes network)
    - **Pod Network**: 10.244.0.0/16 (Flannel/Calico)
  - **Development Cluster**:
    - **Control Plane**: dev-kubernetes-control-plane
    - **Worker Nodes**: dev-kubernetes-worker
    - **Network**: 192.168.71.0/24 (Dev Kubernetes network)
    - **Pod Network**: 10.245.0.0/16

- **Execution Rules**:
  - The agent must **ALWAYS** verify current context with `kubectl config current-context`
  - The agent must **NEVER** run `kubectl delete` on critical system namespaces without explicit confirmation
  - **SSH is allowed for direct node access** and kubeconfig management
  - **Namespace safety**: Always verify namespace before operations

- **Kubernetes Components**:
  - **API Server**: Kubernetes API endpoint (usually port 6443)
  - **etcd**: Distributed key-value store for cluster state
  - **kubelet**: Node agent running on each worker
  - **kube-proxy**: Network proxy for service communication
  - **Container Runtime**: containerd or Docker

### Standard Operating Procedure (SOP)

When asked to "Check Kubernetes status," "Deploy application," or "Troubleshoot pods":

1. **Verify Context**: Check current kubectl context and namespace
2. **Determine Environment**: Identify Production vs Development cluster
3. **Choose Method**:
   - **Cluster Operations**: Use kubectl commands directly
   - **Node Access**: SSH to specific nodes for direct access
4. **Execute & Monitor**: Run the operation and verify results
5. **Document**: If issues are found, consider enriching Keep alerts or updating documentation

## Examples

### 1. Check Cluster Status (Read-Only)

Use this to verify cluster health, node status, and overall cluster information.

- **Method:** kubectl commands
- **Command Pattern:**

```bash
# Check cluster info and current context
kubectl cluster-info
kubectl config current-context

# Check all nodes status
kubectl get nodes -o wide

# Check cluster component health
kubectl get componentstatuses
kubectl get cs

# Check cluster-wide events
kubectl get events --all-namespaces --sort-by='.lastTimestamp'
```

### 2. Inspect Pods and Deployments (Read-Only)

View pod status, deployments, and application health.

- **Method:** kubectl commands
- **Command Pattern:**

```bash
# List all pods across all namespaces
kubectl get pods --all-namespaces -o wide

# Check pods in specific namespace
kubectl get pods -n <namespace> -o wide

# Get detailed pod information
kubectl describe pod <pod-name> -n <namespace>

# Check deployment status
kubectl get deployments -n <namespace>
kubectl describe deployment <deployment-name> -n <namespace>

# Check replica sets
kubectl get replicasets -n <namespace>
```

### 3. Monitor Resource Usage (Read-Only)

Check resource consumption and utilization across the cluster.

- **Method:** kubectl commands
- **Command Pattern:**

```bash
# Check resource usage for nodes
kubectl top nodes

# Check resource usage for pods
kubectl top pods --all-namespaces

# Check resource quotas
kubectl get resourcequota -n <namespace>

# Check limit ranges
kubectl get limitrange -n <namespace>

# View node resource allocation
kubectl describe node <node-name>
```

### 4. Service and Network Operations (Read-Only)

Verify services, endpoints, and network connectivity.

- **Method:** kubectl commands
- **Command Pattern:**

```bash
# List all services
kubectl get services --all-namespaces -o wide

# Check service endpoints
kubectl get endpoints --all-namespaces

# Check ingress resources
kubectl get ingress --all-namespaces

# Test service connectivity
kubectl run test-pod --image=busybox --rm -it --restart=Never -- wget -qO- <service-name>.<namespace>.svc.cluster.local

# Check network policies
kubectl get networkpolicies --all-namespaces
```

### 5. Configuration and Secrets (Read-Only)

Inspect ConfigMaps, Secrets, and application configuration.

- **Method:** kubectl commands
- **Command Pattern:**

```bash
# List ConfigMaps
kubectl get configmaps --all-namespaces

# View ConfigMap contents
kubectl get configmap <configmap-name> -n <namespace> -o yaml

# List Secrets (basic info only)
kubectl get secrets --all-namespaces

# Check CRDs (Custom Resources)
kubectl get crds

# View cluster roles and bindings
kubectl get clusterroles
kubectl get clusterrolebindings
```

### 6. Logs and Debugging (Read-Only)

Access pod logs and debug application issues.

- **Method:** kubectl commands
- **Command Pattern:**

```bash
# View pod logs
kubectl logs <pod-name> -n <namespace>

# Follow logs in real-time
kubectl logs -f <pod-name> -n <namespace>

# Get logs from previous container instance
kubectl logs <pod-name> -n <namespace> --previous

# Get logs from all pods in a deployment
kubectl logs -l app=<app-label> -n <namespace>

# Execute commands in running pod
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash
```

### 7. Scaling and Updates (With Caution)

Manage application scaling and rolling updates.

- **Method:** kubectl commands
- **Command Pattern:**

```bash
# Scale deployment
kubectl scale deployment <deployment-name> --replicas=<count> -n <namespace>

# Check rollout status
kubectl rollout status deployment/<deployment-name> -n <namespace>

# View rollout history
kubectl rollout history deployment/<deployment-name> -n <namespace>

# Rollback deployment
kubectl rollout undo deployment/<deployment-name> -n <namespace>

# Update deployment image
kubectl set image deployment/<deployment-name> <container>=<new-image> -n <namespace>
```

### 8. Namespace and RBAC Management (Read-Only)

Check namespaces, roles, and access controls.

- **Method:** kubectl commands
- **Command Pattern:**

```bash
# List all namespaces
kubectl get namespaces

# Check current user permissions
kubectl auth can-i --list --as=system:serviceaccount:<namespace>:<serviceaccount>

# View role bindings
kubectl get rolebindings --all-namespaces

# Check service accounts
kubectl get serviceaccounts --all-namespaces

# View pod security policies
kubectl get psp --all-namespaces
```

### 9. Storage and Persistent Volumes (Read-Only)

Check storage classes, PVCs, and volume status.

- **Method:** kubectl commands
- **Command Pattern:**

```bash
# List storage classes
kubectl get storageclasses

# Check persistent volume claims
kubectl get pvc --all-namespaces

# Check persistent volumes
kubectl get pv --all-namespaces

# Check volume status details
kubectl describe pvc <pvc-name> -n <namespace>

# View storage usage
kubectl get pv -o custom-columns=NAME:.metadata.name,CAPACITY:.spec.capacity.storage,STATUS:.status.phase
```

### 10. Cluster Diagnostics (Read-Only)

Comprehensive cluster health and diagnostics.

- **Method:** kubectl commands
- **Command Pattern:**

```bash
# Check cluster health summary
kubectl get componentstatuses
kubectl get nodes -o wide

# Check for failing pods
kubectl get pods --all-namespaces --field-selector=status.phase!=Running

# View recent cluster events
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -20

# Check API server connectivity
kubectl get --raw /api/v1/namespaces

# Verify etcd health (if accessible)
kubectl get --raw /healthz
```

## Troubleshooting

### Pod Won't Start

```bash
# Check pod status and events
kubectl describe pod <pod-name> -n <namespace>

# View pod logs
kubectl logs <pod-name> -n <namespace>

# Check resource constraints
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "Limits\|Requests"

# Check image pull issues
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Events"
```

### Service Not Accessible

```bash
# Check service endpoints
kubectl get endpoints <service-name> -n <namespace>

# Verify service configuration
kubectl describe service <service-name> -n <namespace>

# Test pod-to-service connectivity
kubectl run test-pod --image=busybox --rm -it --restart=Never -- nslookup <service-name>.<namespace>.svc.cluster.local

# Check network policies
kubectl get networkpolicies -n <namespace>
```

### Node Issues

```bash
# Check node status and conditions
kubectl describe node <node-name>

# View node resource usage
kubectl top node <node-name>

# Check kubelet logs (requires SSH)
ssh ansible@<node-name> "sudo journalctl -u kubelet -n 50"

# Check for taints and tolerations
kubectl describe node <node-name> | grep -A 10 "Taints"
```

### Resource Constraints

```bash
# Check resource quotas
kubectl describe resourcequota -n <namespace>

# View limit ranges
kubectl describe limitrange -n <namespace>

# Check pod resource requests vs limits
kubectl get pods -n <namespace> -o custom-columns=NAME:.metadata.name,CPU:.spec.containers[*].resources.requests.cpu,MEMORY:.spec.containers[*].resources.requests.memory
```

### Storage Issues

```bash
# Check PVC status
kubectl describe pvc <pvc-name> -n <namespace>

# Verify storage class
kubectl describe storageclass <storage-class-name>

# Check volume binding
kubectl get pv -o wide

# Check for volume issues
kubectl get events -n <namespace> --field-selector=involvedObject.kind=PersistentVolumeClaim
```

## Quick Reference Commands

### Cluster Status (Read-Only)
```bash
# Cluster overview
kubectl cluster-info && kubectl get nodes -o wide

# Current context
kubectl config current-context && kubectl config view

# Component health
kubectl get componentstatuses && kubectl get events --all-namespaces
```

### Pod Management (Read-Only)
```bash
# List pods
kubectl get pods --all-namespaces -o wide

# Pod details
kubectl describe pod <pod-name> -n <namespace>

# Pod logs
kubectl logs <pod-name> -n <namespace> --tail=100
```

### Service Management (Read-Only)
```bash
# List services
kubectl get services --all-namespaces -o wide

# Service details
kubectl describe service <service-name> -n <namespace>

# Endpoints
kubectl get endpoints --all-namespaces
```

### Resource Monitoring (Read-Only)
```bash
# Node resources
kubectl top nodes

# Pod resources
kubectl top pods --all-namespaces

# Resource quotas
kubectl get resourcequota --all-namespaces
```

## Important Notes

- **Always verify context** with `kubectl config current-context` before executing commands
- **Never delete system namespaces** (kube-system, kube-public) without explicit confirmation
- **Check resource quotas** before scaling deployments
- **Use namespaces** to organize and isolate applications
- **Monitor resource usage** to avoid cluster overload
- **Document changes** in Keep alerts or Notion for team visibility
- **Test in development** before applying changes to production

## Related Documentation

- **Kubernetes Documentation**: [https://kubernetes.io/docs/](https://kubernetes.io/docs/)
- **kubectl Cheat Sheet**: [https://kubernetes.io/docs/reference/kubectl/cheatsheet/](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- **Notion**: [Kubernetes Cluster Management — Home Lab Guide](https://www.notion.so/kubernetes-cluster-management)
- **GitHub Repo**: `homelab-playbooks` (BjzyLabs/ansible-homelab)
- **Local Docs**: `docs/KUBERNETES_DEPLOYMENT.md`, `docs/KUBERNETES_OPERATIONS.md`
