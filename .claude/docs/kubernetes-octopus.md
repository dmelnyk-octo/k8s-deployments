# Octopus Deploy — Kubernetes Integration Reference

> Fetched from https://octopus.com/docs/kubernetes on 2026-05-12.
> Use this file as the authoritative reference instead of relying on model knowledge.

---

## Overview

Octopus Deploy manages Kubernetes resources whether you're starting simple or need complete control over a complex setup. Four deployment pathways are supported: Helm charts, YAML files, Kustomize, or the Octopus UI for configuration generation.

---

## 1. Deployment Targets

Octopus supports two kinds of Kubernetes targets.

### 1a. Kubernetes Agent (recommended)

A lightweight application installed inside the target cluster. Uses the same polling communication protocol as Octopus Tentacle — the agent initiates connections outward, bypassing firewall issues. No external credentials stored on the server.

#### System Requirements

| Component | Requirement |
|-----------|-------------|
| Kubernetes | 1.26–1.35 (agent 3.x requires 1.32–1.35) |
| Octopus Server | 2024.2.9396+ |
| Architecture | Linux AMD64 & ARM64 only |

#### How Task Execution Works

1. Tentacle pod maintains active connection to Octopus Server
2. Files and tools transferred to shared storage
3. New script pod created per deployment task
4. Script pod accesses shared storage, executes task, streams logs back
5. Script pod self-terminates on completion

#### Installation

Navigate to **Infrastructure → Deployment Targets → Add Deployment Target → KUBERNETES → Kubernetes Agent**.

| Field | Description |
|-------|-------------|
| Display Name | Unique identifier; generates namespace and Helm release name |
| Environment | Target deployment environment(s) |
| Target Tags | Organizational tags for step scoping |
| Default Namespace | Optional; namespace for resource deployment |
| Storage Class | Optional; defaults to cluster default with ReadWriteOnce |

Octopus generates a Helm install command with a 1-hour bearer token:

```bash
helm upgrade --install --atomic \
  --set agent.acceptEula="Y" \
  --set agent.targetName="<name>" \
  --set agent.serverUrl="<serverUrl>" \
  --set agent.serverCommsAddress="<serverCommsAddress>" \
  --set agent.space="Default" \
  --set agent.targetEnvironments="{<env1>,<env2>}" \
  --set agent.targetRoles="{<role1>,<role2>}" \
  --set agent.bearerToken="<token>" \
  --version "3.*.*" \
  --create-namespace --namespace <namespace> \
  <release-name> \
  oci://registry-1.docker.io/octopusdeploy/kubernetes-agent
```

#### Tenants (post-install or via Helm)

```bash
--set agent.tenants="{tenant1,tenant2}" \
--set agent.tenantTags="{tag1,tag2}" \
--set agent.tenantedDeploymentParticipation="TenantedOrUntenanted"
```

#### Certificate Configuration

```bash
# Self-signed / internal CA
--set global.serverCertificate="<base64-encoded-cert>"
# Or reference an existing secret
--set global.serverCertificateSecretName="octopus-server-certificate"
# gRPC TLS bridging
--set kubernetesMonitor.monitor.customCaCertificate="<base64-encoded-cert>"
```

#### Upgrades

- **Automatic** (default): controlled via Machine Policies
- **Manual**:
  ```bash
  helm upgrade --atomic --namespace NAMESPACE HELM_RELEASE_NAME \
    oci://registry-1.docker.io/octopusdeploy/kubernetes-agent
  ```
- **Rollback**: `helm rollback --namespace NAMESPACE HELM_RELEASE_NAME`

#### Permissions

Two-tier service account model:

| Account | Default Name | Scope |
|---------|-------------|-------|
| Agent Pod | `<agent-name>-tentacle` | Create/view/modify pods, logs, configmaps, secrets **in agent namespace only** |
| Script Pod | `<agent-name>-scripts` | Cluster-wide admin by default (all API groups, resources, verbs) |

Restrict script pod to specific namespaces:
```bash
--set scriptPods.serviceAccount.targetNamespaces="development,preproduction"
```

Custom RBAC rules:
```bash
--set scriptPods.serviceAccount.clusterRole.rules=...
```

#### Kubernetes Monitor

Runs alongside Tentacle; tracks health of resources deployed via Octopus.

- Communicates with Octopus Server over **gRPC port 8443** (outbound-only)
- Internally uses the Argo gitops engine to track cluster resources
- Requires a read-only ClusterRole (`get`, `watch`, `list` across all groups/resources) after registration
- Upgrade is tied to the Kubernetes agent upgrade process

---

### 1b. Kubernetes API Target

Defines the cluster endpoint, credentials, and namespace directly. Requires workers with `kubectl` installed (Linux workers also need `jq`, `xargs`, `base64`).

#### Configuration Fields

| Field | Required | Notes |
|-------|----------|-------|
| Display Name | Yes | |
| Environment | Yes | At least one |
| Target Tags | Yes | At least one |
| Authentication | Yes | See methods below |
| Kubernetes Cluster URL | Yes | |
| Kubernetes Namespace | Yes | Used by default; can be overridden per step |
| Cluster Certificate Authority | No | For self-signed certs |

#### Authentication Methods

| Method | Notes |
|--------|-------|
| Username/Password | From kubeconfig `username`/`password` |
| Token | From kubeconfig `token`; Kubernetes 1.24+ no longer auto-creates tokens — must be manually created |
| Azure Service Principal | For AKS; requires `kubelogin` CLI from Octopus 2023.3 when local accounts are disabled |
| AWS Account | For EKS; requires `aws cli` 1.16.156+ or `aws-iam-authenticator` on execution path |
| Google Cloud Account | For GKE; requires `gke-cloud-auth-plugin` from kubectl 1.26+ |
| Client Certificate | Base64-encoded cert + key combined into PFX format |

#### Kubeconfig Mapping

| kubeconfig field | Octopus field |
|-----------------|---------------|
| `server` | Kubernetes cluster URL |
| `certificate-authority-data` | Cluster certificate authority |
| `username` / `password` / `token` | Account credentials |

#### Target Discovery (2022.2+)

Automatic discovery of AKS and EKS clusters via cloud resource tags, enabling dynamic target provisioning per environment.

---

## 2. Deployment Steps

### 2a. Deploy Kubernetes YAML

Deploys arbitrary Kubernetes resources from YAML. Requires deeper Kubernetes knowledge but gives full control.

#### YAML Source Options

| Source | Notes |
|--------|-------|
| **Git Repository** | Clones repo at deployment time; commit hash saved at release creation — redeployments use that commit |
| **Package** | Traditional package feed approach; supports glob patterns (2023.3+) |
| **Inline YAML** | Simplest; YAML saved in Octopus project; not recommended for advanced use |

#### Glob Patterns (Git / Package sources)

```
# Multiple newline-separated paths (applied top-to-bottom)
deployments/apply-first.yaml
services/apply-second.yml

# Glob pattern
**/*.{yaml,yml}
```

Glob rules:
- Paths are always **relative** (no leading `/`, `./`, or `../`)
- Use forward slashes on all platforms
- `?` = single character, `*` = within one directory level, `**` = recursive

#### Variable Substitution

Full Octopus variable substitution in YAML content:

```yaml
image: "#{Octopus.Action.Package[nginx].PackageId}:#{Octopus.Action.Package[nginx].PackageVersion}"
```

---

### 2b. Deploy a Helm Chart

#### Chart Sources

| Source | Notes |
|--------|-------|
| **Helm Feed** | HTTP server hosting `index.yaml`; supports ChartMuseum, Artifactory, Cloudsmith |
| **OCI Registry** | OCI-based registry hosting packaged charts (2023.3.4127+) |
| **Git Repository** | Direct from Git; commit hash captured at release creation (2024.1+) |

#### Key Configuration

| Field | Notes |
|-------|-------|
| Kubernetes release name | Unique per cluster (not just namespace); defaults to project+environment |
| Reset values | Default: enabled (`--reset-values`); Octopus config is authoritative |
| Helm client tool | `helm` must be on workers; can specify full path or version override |

#### Values Sources (applied in reverse order — top = highest precedence)

1. Files in the chart (default: `./values.yaml`)
2. Files in a Git repository
3. Files in a package
4. Key/value pairs
5. Inline YAML

#### Image Updates Without Chart Versioning

Add Docker image references in the step, then reference in values YAML:

```yaml
image:
  repository: "#{Octopus.Action.Package[myimage].PackageId}"
  tag: "#{Octopus.Action.Package[myimage].PackageVersion}"
```

#### Known Limitations

| Issue | Details |
|-------|---------|
| Deployment cancellation | `--atomic` doesn't auto-rollback on cancellation; may need manual cleanup |
| Timeout interaction | If Octopus timeout < Helm timeout, partial deployments can occur; set Octopus timeout larger |
| Path length | Deployments fail with 100+ character tar.gz paths; use ZIP or shorter filenames |
| Provenance validation | Not automatically performed |

---

### 2c. Deploy with Kustomize

Sources Kustomize files from Git, performs variable substitution, applies to cluster.

#### Configuration Fields

| Field | Notes |
|-------|-------|
| Kustomization File Directory | Path to directory containing `kustomization.yaml`; relative to repo root; Linux paths are case-sensitive |
| Substitute Variables in Files | Target paths relative to repo root; supports glob patterns |
| Container Image References (v2.0.2+) | Add package references; use `#{Octopus.Action.Package[name].PackageVersion}` |

#### Overlay Patterns

- **Multiple overlays** — `.env` files with Octopus variable substitution feeding `secretGenerator`/`configMapGenerator`; static `kustomization.yaml`
- **Single Octopus overlay** — inject variable substitution directly into `kustomization.yaml`; suits multi-env/multi-tenant
- **Mixed** — hardcoded environment overlays + Octopus substitution for tenant-specific properties

---

### 2d. Configure and Apply Kubernetes Resources (UI-driven)

Deploys Deployment + Service + Ingress through a unified UI. Renamed from "Deploy Kubernetes containers" in 2024.1.

#### Deployment Settings

| Field | Default | Notes |
|-------|---------|-------|
| Replicas | 1 | |
| Progression deadline | 600s | Max time before failure |
| Revision history limit | — | Number of revisions kept |
| Pod termination grace period | — | |

#### Deployment Strategies

| Strategy | Zero Downtime | Notes |
|----------|---------------|-------|
| Recreate | No | Terminates all existing pods first |
| Rolling Update | Yes* | Incremental replacement |
| Blue/Green | Yes | Creates new Deployment, switches Service only after new Deployment is ready |

Blue/Green phases:
1. Existing green Deployment + Service running
2. New blue Deployment created; Service still points to green
3. Waits for blue readiness (`kubectl rollout status`)
4. Service updated to blue; old resources deleted

#### Container Configuration

**Image Pull Policy:**
- `If Not Present` — pull only if locally absent
- `Always` — pull on every pod start
- `Never` — assume local existence
- Default: `Always` for `latest` tag; `If Not Present` for specific tag

**Health Probes** (Liveness, Readiness, Startup):

| Type | Available on |
|------|-------------|
| Command | All |
| HTTP | All |
| TCP Socket | All |

Readiness probes not supported on Init Containers.

**Volume Types:** ConfigMap, Secret, EmptyDir, HostPath, PersistentVolumeClaim, Raw YAML (for EBS, GCE PD, etc.)

**Environment Variables:** plain key/value, ConfigMap reference, or Secret reference

#### Service Types

| Type | Access |
|------|--------|
| Cluster IP | In-cluster only |
| Node Port | Cluster IP + node-level port |
| Load Balancer | Cluster IP + node ports + cloud load balancer |

#### ConfigMap and Secret Lifecycle

New resources created per deployment with unique names (deployment ID appended). Old resources cleaned up automatically after new Deployment succeeds.

Custom name templates via variables:
- `Octopus.Action.KubernetesContainers.ConfigMapNameTemplate`
- `Octopus.Action.KubernetesContainers.SecretNameTemplate`

#### Automatic Labels Applied

- `Octopus.Deployment.Id`
- `Octopus.Step.Id`
- `Octopus.Environment.Id`
- `Octopus.Deployment.Tenant.Id`
- `Octopus.Kubernetes.DeploymentName`

---

## 3. Deployment Verification

Octopus compares deployed resource actual state against desired state; step completes only when states match.

Applies to: Deploy Kubernetes YAML, Deploy Helm Chart, Deploy with Kustomize, Configure and apply Kubernetes resources (except Blue/Green), ConfigMap/Secret/Ingress/Service steps.

**Disabled by default for pre-existing steps; enabled by default for new steps.**

### Configuration

| Setting | Notes |
|---------|-------|
| Verify desired state | Toggle on/off |
| Step Timeout | Seconds before step fails; disabled = indefinite |
| Wait for Jobs | On: waits for job completion; Off: succeeds once jobs are created |

### Helm Steps

Uses Helm's built-in `--wait` parameter (+ `--wait-for-jobs` optional).

### Other Steps

Octopus identifies objects, continuously polls status including child objects (ReplicaSets, Pods). Stand-alone pods/jobs fail early if reaching unrecoverable states.

### Object Status Display

| Status | Meaning |
|--------|---------|
| In progress | Moving toward desired state |
| Success | Desired state achieved |
| Error | Failed |
| Timed out while in progress | Timeout reached before success |

---

## 4. Live Object Status

Post-deployment real-time monitoring of Kubernetes objects directly in Octopus.

**Requirements:** Octopus 2025.3+, Kubernetes Agent target, projects with Kubernetes steps.

### Health Status

| Status | Meaning |
|--------|---------|
| Progressing | Moving toward desired state |
| Healthy | Resources match last deployment |
| Unknown | Status unavailable |
| Degraded | Errors post-deployment |
| Missing | Resources absent from cluster |
| Unavailable | Deployment failure prevented tracking |
| Waiting | Pending deployment completion |
| Stale | No data received in last 10 minutes |

### Sync Status

| Status | Meaning |
|--------|---------|
| In Sync | Cluster matches last deployment |
| Out of Sync | Discrepancies detected |
| Unknown | Cannot retrieve sync info |
| Unavailable | Failed deployment |
| Waiting | Awaiting deployment completion |

### Reporting Manifests from kubectl Script Steps

```bash
# Bash
report_kubernetes_manifest "$manifest"
report_kubernetes_manifest_file "$path"
```

```powershell
# PowerShell
Report-KubernetesManifest -manifest $manifest
Report-KubernetesManifestFile -path $path
```

Available in Octopus 2025.4.10333+ and 2026.1.4557+.

### Security

- Octopus variables redacted from manifests, logs, events
- Kubernetes secrets hashed with SHA256 (Octopus cannot decrypt unknown secrets)
- Requires `DeploymentView` permission to access live status data

### Limitations

- Objects from excluded deployment steps won't display live status
- Runbook-modified objects not monitored (use deployments instead)
- Log tailing not yet available

---

## 5. Troubleshooting

### Kubernetes Agent Installation

| Symptom | Cause | Fix |
|---------|-------|-----|
| `context deadline exceeded` during Helm install | Resources didn't start within 5-min `--atomic` timeout | Run `kubectl describe pods` + `kubectl logs` to find root cause; check network, token validity, storage class |
| Helm install dialog stuck / no connection | Invalid server URL, expired bearer token, network issue | Verify `registration.octopus.serverApiUrl` reachable from cluster; rerun within token lifetime |
| `CrashLoopBackoff` + SSL handshake failure | SHA1RSA cert on Windows Server 2012 R2 | Refer to SHA1 certificate incompatibility guide |
| Service account annotation config (EKS IAM) | Dots in annotation key need escaping | `--set scriptPods.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:..."` |

### Script Execution

| Symptom | Cause | Fix |
|---------|-------|-----|
| "Unexpected Script Pod log line number" | Container logs rotated before Octopus read them | Increase log retention; deployment fails intentionally to prevent hidden changes |
| "The Script Pod could not be found" | Script pod evicted/terminated by Kubernetes (e.g. NFS restart, storage quota) | Monitor evictions; increase storage quotas |
| `Exec format error` / architecture mismatch | Script pod scheduled on different arch node than Tentacle | Set node affinity for both Tentacle and script pods to same arch via Helm values YAML |

### Health Checks and Upgrades

| Symptom | Cause | Fix |
|---------|-------|-----|
| "service account octopus-agent-auto-upgrader not found" | Octopus Server 2024.3.11946+ / 2024.4 requires auto-upgrader account added in agent 1.16.0 (V1) / 2.2.0 (V2) | Manually upgrade agent: `helm upgrade --atomic --namespace NAMESPACE --version "2.*.*" RELEASE oci://...` |

### Live Object Status

| Symptom | Cause | Fix |
|---------|-------|-----|
| gRPC port 8443 connection failure | Firewall blocking outbound; misconfigured port | Allow outbound 8443; verify `grpcListenPort=8443` on Octopus Server |
| Certificate error on gRPC | Self-signed cert; load balancer without TLS passthrough | Check agent custom certificate config; consult LB docs for gRPC |
| "Couldn't find Kubernetes monitor associated with target" | Monitor registration lost | Delete + reinstall agent, OR delete `<agent-name>-kubernetesmonitor-authentication` secret and restart monitor pod |
| Stale / slow status updates | Conservative rate limits | Expected during early access; rate limits may be raised |
| Objects Out of Sync | Manual updates outside Octopus; controllers modifying objects | Ensure Octopus is sole modification entity; validate manifests have no invalid fields |
| Live Status Panel absent from dashboard | Feature disabled | Enable "Live Status" toggle at dashboard top |
