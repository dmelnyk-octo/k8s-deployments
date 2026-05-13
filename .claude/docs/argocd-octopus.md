# Octopus Deploy — Argo CD Integration Reference

> Fetched from https://octopus.com/docs/argo-cd on 2026-05-12.
> Use this file as the authoritative reference instead of relying on model knowledge.

---

## Overview

Octopus makes it easy to improve Argo CD deployments with **environment modeling** and **deployment orchestration**. Argo CD treats each application as an independent entity with no codified relationship between staging and production — Octopus solves this via three mechanisms:

1. **Instances** — connections to running Argo CD deployments, with Applications cross-mapped to Octopus Projects
2. **Steps** — deployment steps that update Git repositories backing mapped Argo CD Applications
3. **Visibility** — dashboards and live status showing deployment results and resource health

---

## 1. Argo CD Instances

### Prerequisites

A **network gateway** must be installed in your Kubernetes cluster before connecting Octopus to Argo CD. The gateway establishes a TLS-encrypted, outgoing gRPC connection from the cluster to Octopus Server — no publicly accessible HTTP/gRPC endpoint required. One gateway is required per Argo CD instance.

### Adding an Instance

Navigate to **Infrastructure → Argo CD Instances → Add Argo CD Instance**.

| Field | Description |
|-------|-------------|
| Instance Name | Unique name; used to generate the Kubernetes namespace and Helm release name |
| Environment Selection | At least one environment the instance will service |
| Argo CD API Server URL | In-cluster URL of the Argo CD API Server service |
| Argo CD Web Frontend URL | Optional; enables links from Octopus to the Argo CD UI |
| JWT Authentication Token | Valid Argo CD JWT token; user needs read access to Application and Cluster resources |

After configuration, Octopus generates a Helm install command with a one-hour bearer token for initial gateway registration.

### TLS Options (Helm flags)

```
gateway.argocd.insecure="true"    # self-signed certificate
gateway.argocd.plaintext="true"   # no TLS at all
```

---

## 2. Authentication — Creating the Octopus User

### Step 1 — Create account in `argocd-cm`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  accounts.octopus: apiKey
  accounts.octopus.enabled: "true"
```

Or via kubectl patch:

```bash
kubectl patch configmap argocd-cm -n argocd --type merge -p \
  '{"data":{"accounts.octopus":"apiKey","accounts.octopus.enabled":"true"}}'
```

`apiKey` allows token generation but prevents web-UI login.

Verify: `argocd account list`

### Step 2 — Configure RBAC in `argocd-rbac-cm`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    p, octopus, applications, get, *, allow
    p, octopus, applications, sync, *, allow
    p, octopus, clusters, get, *, allow
    p, octopus, logs, get, */*, allow
```

Or via kubectl patch:

```bash
kubectl patch configmap argocd-rbac-cm -n argocd --type merge -p \
  '{"data":{"policy.csv":"p, octopus, applications, get, *, allow\np, octopus, applications, sync, *, allow\np, octopus, clusters, get, *, allow\np, octopus, logs, get, *\/*, allow\n"}}'
```

> **Note:** Insufficient permissions allow connection but result in an empty application list.

### Step 3 — Generate API Token

**CLI:**
```bash
argocd login <your-argo-web-ui>
argocd account generate-token --account octopus
```

**UI:** Settings → Accounts → octopus → Generate Token

### Permission Verification

```bash
argocd account can-i --auth-token <token> get clusters '*'        # expect: yes
argocd account can-i --auth-token <token> get applications '*'    # expect: yes
argocd account can-i --auth-token <token> get logs '*/*'          # expect: yes
argocd account can-i --auth-token <token> delete applications '*' # expect: no
```

---

## 3. Scoping Annotations

Annotations establish the relationship between an Argo CD Application Source and an Octopus Project + Environment (+ optional Tenant). They determine which sources are updated during deployments.

### Core Annotation Keys

| Annotation | Required | Purpose |
|-----------|----------|---------|
| `argo.octopus.com/project[.<source-name>]` | Yes | Octopus Project slug |
| `argo.octopus.com/environment[.<source-name>]` | Yes | Octopus Environment slug |
| `argo.octopus.com/tenant[.<source-name>]` | No | Octopus Tenant slug |

### Single Source — Unnamed

```yaml
metadata:
  annotations:
    argo.octopus.com/environment: development
    argo.octopus.com/project: argo-cd-guestbook
spec:
  source:
    repoURL: https://github.com/example-org/guestbook.git
    targetRevision: HEAD
    path: ./
```

### Single Source — Named

```yaml
metadata:
  annotations:
    argo.octopus.com/environment.guestbook-source: development
    argo.octopus.com/project.guestbook-source: argo-cd-guestbook
spec:
  source:
    name: guestbook-source
    repoURL: https://github.com/example-org/guestbook.git
```

### Multiple Sources (requires Argo CD 2.14.0+)

All sources must be named; each source gets distinct annotations:

```yaml
metadata:
  annotations:
    argo.octopus.com/environment.service-1: development
    argo.octopus.com/project.service-1: argo-cd-service-1
    argo.octopus.com/environment.service-2: development
    argo.octopus.com/project.service-2: argo-cd-service-2
spec:
  sources:
    - name: service-1
      repoURL: https://github.com/example-org/service-1.git
    - name: service-2
      repoURL: https://github.com/example-org/service-2.git
```

### Generating Annotations via UI

**Infrastructure → Argo CD Instances → [instance] → Generate Scoping Annotations** — choose Project, Environment, and optional Tenant to auto-generate YAML.

---

## 4. Cluster Annotations

If the cluster's default container registry differs from `docker.io`, add this annotation to the cluster object in Argo CD (**Settings → Clusters → Edit**):

```
argo.octopus.com/default-container-registry : my-company-registry.com
```

Only needed if the cluster default has been changed from standard Kubernetes defaults.

---

## 5. Deployment Steps

Octopus provides two built-in Argo CD steps.

### Common Configuration

**Git Commit Settings:**
- **Commit Message** — summary/description; auto-populated if left empty; cannot reference Sensitive Variables
- **Git Commit Method** — direct commit to branch, or merge via pull request
  - PR support: GitHub, GitLab, Azure DevOps
  - PRs can be scoped per environment

**Step Verification (from 2026.1):**

| Option | Behavior |
|--------|----------|
| Direct commit or pull request created | Ensures Git commit succeeded; no further checks |
| Pull request merged (2026.2+) | Pauses until all PRs merge; polls every 60s |
| Argo CD Application is healthy | Waits for apps to be in-sync and healthy; polls every 30s |

**Trigger Sync:** Forces Argo CD to explicitly sync applications after commit. Configurable per environment.

**Output Variables:**

| Variable | Content |
|----------|---------|
| `PullRequest.Title` | PR title (empty if no PR) |
| `PullRequest.Number` | PR identifier (empty if no PR) |
| `PullRequest.Url` | PR URL (empty if no PR) |

---

### 5a. Update Argo CD Application Image Tags

Updates container image tags in the Application's repository to match Octopus release package versions.

**Configuration:**
- Add a package reference per container image to update
- Unreferenced images are left unchanged
- For Helm sources: set **Helm image tag path** (e.g. `agent.image.tag`)

**Helm tag path validation:**
- "Only the tag" → replaced with package version
- "Tag + ImageName + repository" → validated against step package properties before replacement; mismatched namespace/repo skips the replacement

**Update behaviour by source type:**

| Source type | What is updated |
|-------------|----------------|
| Kubernetes YAML | Image fields in known resource types (not CRDs) |
| Helm Charts | `values.yaml` tag fields matching Helm Annotations |
| Kustomize | Only `newTag` fields in Kustomize files |

> **Note:** Helm Annotations are only processed when **no** `helm-image-tag-paths` are configured on the step directly.

---

### 5b. Update Argo CD Application Manifests

Populates an Argo CD Application's Git repository with content generated from Octostache templates substituted with Octopus Variables.

**Template input sources:**
- Git repository (URL, credentials, branch)
- Feed packages (zip, NuGet, etc.)

**Directory handling:**
- With `path`: templates copy under the application's path
- With `ref` only: templates copy into repository root
- With both `ref` and `path`: step takes no action (ambiguous)
- Override output path: `argo.octopus.com/path.<source-name>` annotation

**Purge Argo CD Source Folder:** Clears the Path directory before writing templates — useful when removed resources must also be removed from the target repo.

**Variable substitution syntax:** `#{VARIABLE_NAME}`

```yaml
# Example template
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config
data:
  directory: "#{Octopus.Environment.Name}"
  database_url: "jdbc:postgresql://mydb.example.com:5432/#{DB_NAME}"
  deployment_created_at: "#{Octopus.Deployment.Created}"
```

> **Security constraint:** Deployments fail if a template references a Sensitive Variable (prevents secrets persisting in Git).

---

## 6. Helm Image Tags Annotations (Advanced)

Use when direct Helm image path configuration in the step is insufficient.

### Annotation Key

```
argo.octopus.com/image-replace-paths[.<helm-source-name>]
```

Comma-delimited Helm-template style string building fully qualified image names. Octopus requires full registry + image to avoid accidental matches.

### Examples

```yaml
# Separate registry and tag
argo.octopus.com/image-replace-paths: "docker.io/{{ .Values.agent.image.repository }}:{{ .Values.agent.image.tag }}"

# Combined repository and tag
argo.octopus.com/image-replace-paths: "{{ .Values.global.image.registry }}/{{ .Values.global.image.repositoryAndTag }}"

# Multiple images
argo.octopus.com/image-replace-paths: "{{ .Values.image.name}}:{{ .Values.image.version}}, {{ .Values.another-image.name }}"

# Named Helm source
argo.octopus.com/image-replace-paths.helm-source: "{{ .Values.image.name}}:{{ .Values.image.version}}"
```

> If **any** package/container in a step uses annotations, **all** packages must use annotations.

---

## 7. Argo CD Applications View (Step)

Provides visibility into which Argo CD Applications will be updated before deployment runs.

| State | Behaviour |
|-------|-----------|
| No gateway/instance registered | Button to initiate gateway registration |
| No scoping annotations | Prompts to add annotations; helper drawer available |
| Annotations present | "Deployment Preview" shows mapped applications |
| Missing Git credentials | Warning triangle; "Connect Git Credential" button |
| All preconditions met | Clean preview; deployment can proceed |

> Argo CD Instances are **not** Deployment Targets and use annotations (not Target Tags) for scoping.

---

## 8. Live Object Status

Shows live status of Kubernetes resources deployed by mapped Argo CD Applications.

**Requirements:** Octopus 2025.4+ (2026.1+ for Sync Status), registered Argo CD Instance, scoping annotations, at least one Argo CD step in the deployment process.

| Category | States |
|----------|--------|
| Project Health | Progressing, Healthy, Unknown, Degraded, Missing, Unavailable, Waiting, Stale |
| Project Sync | In Sync, Out of Sync, Git Drift, Unknown, Unavailable, Waiting |
| Object Health | Progressing, Healthy, Unknown, Degraded, Missing, Suspended, Stale |
| Object Sync | In Sync, Out of Sync, Git Drift, Unknown |

Expandable drawers show Summary, Events, Logs, and Kubernetes YAML manifest (fetched on-demand from the Argo instance).

> Manifest diffs show repository state on the **left** and live cluster manifest on the **right** (opposite of Argo CD's native UI).

---

## 9. Supported Use Cases & Constraints

| Constraint | Detail |
|-----------|--------|
| `targetRevision` | Cannot update pinned revisions; Octopus treats them as branches and will fail pushing back |
| Helm from Helm repo / OCI feed | Read-only; Octopus cannot update these sources |
| Helm charts as directories in repo | Fully supported |
| Pull requests | GitHub, GitLab, Azure DevOps only |
| Multi-source applications | Requires Argo CD 2.14.0+ (named sources) |

---

## 10. Troubleshooting

### Gateway Installation

| Symptom | Cause | Fix |
|---------|-------|-----|
| Dialog stuck "in progress" | Gateway can't reach Octopus Server URL | Verify `registration.octopus.serverApiUrl` is reachable from within the cluster; use `host.docker.internal` for local clusters |
| Helm command halts after 5+ min | Expired API token | Rerun installation within token lifetime |
| `CrashLoopBackoff` + "error validating connection to Argo CD" | Wrong gRPC endpoint / TLS misconfiguration | Confirm `gateway.argocd.serverGrpcUrl`; set `insecure: true` or `plaintext: true` as appropriate; delete gateway and reinstall |

### Application / Project Mapping

| Symptom | Cause | Fix |
|---------|-------|-----|
| Octopus shows no applications (Argo CD has them) | Token lacks permissions | Add correct RBAC entries for the octopus account |
| "No instances available" in step | No registered instances in current space | Register instance via Infrastructure → Argo CD Instances |

### Step Execution

| Symptom | Cause | Fix |
|---------|-------|-----|
| Deployment completes with warnings; "No annotated Argo CD applications could be found" | Missing/wrong annotations | Review and update annotations to match project/environment/tenant |
| "Could not find a Git Credential associated with" | No Git Credential for the repo | Add credential with correct repository allowlist URL |
| "Failed to clone Git repository" | Source is Helm/OCI (not Git), or insufficient Git privileges | Ensure Git credentials have read/write permissions |
| `http status code: 403` | Git credential lacks repo permissions | Create appropriate credential in Git provider; store in Octopus |

### Dashboard

| Symptom | Cause | Fix |
|---------|-------|-----|
| Live Status Panel absent | Feature disabled | Enable "Live Status" toggle at dashboard top |
