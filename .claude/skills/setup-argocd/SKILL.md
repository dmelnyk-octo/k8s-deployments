---
name: setup-argocd
description: "Installs ArgoCD into the argocd namespace of the current cluster and creates an Octopus Deploy service account with a scoped API token. Assumes a cluster is already reachable via kubectl. Usage: /setup-argocd"
allowed-tools: ["Bash"]
---

Install ArgoCD and configure an Octopus Deploy service account on the current cluster.

Assumes the cluster was created with `extraPortMappings` mapping host port `8080` → node port `30080` (see `bootstrap-kind` skill). ArgoCD will be permanently accessible at `http://localhost:8080` — no port-forward needed.

## Steps

### 1. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=available --timeout=300s deployment --all -n argocd
```

### 2. Expose ArgoCD via NodePort

Patch the `argocd-server` service to NodePort on 30443 (mapped to host port 8080 via `extraPortMappings`):

```bash
kubectl patch svc argocd-server -n argocd \
  -p '{"spec":{"type":"NodePort","ports":[{"name":"http","port":80,"targetPort":8080,"protocol":"TCP"},{"name":"https","port":443,"targetPort":8080,"nodePort":30443,"protocol":"TCP"}]}}'
```

ArgoCD is now available at `https://localhost:8080` (TLS is active, cert is self-signed).

### 3. Create the octopus account

Patch `argocd-cm` to add an `octopus` user with `apiKey` capability only (no UI login):

```bash
kubectl patch configmap argocd-cm -n argocd --type merge -p \
  '{"data":{"accounts.octopus":"apiKey","accounts.octopus.enabled":"true"}}'
```

### 4. Add RBAC permissions

```bash
kubectl patch configmap argocd-rbac-cm -n argocd --type merge -p \
  '{"data":{"policy.csv":"p, octopus, applications, get, *, allow\np, octopus, applications, sync, *, allow\np, octopus, clusters, get, *, allow\np, octopus, logs, get, *\/*, allow\n"}}'
```

### 5. Generate an API token

```bash
ADMIN_PASS=$(kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d)

argocd login localhost:8080 --username admin --password "$ADMIN_PASS" --insecure

OCTOPUS_TOKEN=$(argocd account generate-token --account octopus)

echo ""
echo "=== Octopus ArgoCD API Token ==="
echo "$OCTOPUS_TOKEN"
echo "================================"
echo "Copy this token into Octopus Deploy > Infrastructure > Argo CD Accounts."
```

### 6. Verify permissions

```bash
argocd account can-i --auth-token "$OCTOPUS_TOKEN" get clusters '*'         # expect: yes
argocd account can-i --auth-token "$OCTOPUS_TOKEN" get applications '*'     # expect: yes
argocd account can-i --auth-token "$OCTOPUS_TOKEN" get logs '*/*'           # expect: yes
argocd account can-i --auth-token "$OCTOPUS_TOKEN" delete applications '*'  # expect: no
```

If any of the first three return `no`, or the last returns `yes`, report the failure and stop.

### 7. Report

```bash
kubectl get pods -n argocd
```

Print all pod statuses and remind the user to save the token — it cannot be retrieved again.
