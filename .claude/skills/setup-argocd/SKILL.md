---
name: setup-argocd
description: "Installs ArgoCD into the argocd namespace of the current cluster and creates an Octopus Deploy service account with a scoped API token. Assumes a cluster is already reachable via kubectl. Usage: /setup-argocd"
allowed-tools: ["Bash"]
---

Install ArgoCD and configure an Octopus Deploy service account on the current cluster.

## Steps

### 1. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=available --timeout=300s deployment --all -n argocd
```

### 2. Create the octopus account

Patch `argocd-cm` to add an `octopus` user with `apiKey` capability only (no UI login):

```bash
kubectl patch configmap argocd-cm -n argocd --type merge -p \
  '{"data":{"accounts.octopus":"apiKey","accounts.octopus.enabled":"true"}}'
```

Verify the entry is present:

```bash
kubectl get configmap argocd-cm -n argocd -o jsonpath='{.data}' | grep octopus
```

### 3. Add RBAC permissions

Patch `argocd-rbac-cm` to grant the octopus user read access to applications, clusters and logs, plus sync permission on applications:

```bash
kubectl patch configmap argocd-rbac-cm -n argocd --type merge -p \
  '{"data":{"policy.csv":"p, octopus, applications, get, *, allow\np, octopus, applications, sync, *, allow\np, octopus, clusters, get, *, allow\np, octopus, logs, get, *\/*, allow\n"}}'
```

### 4. Generate an API token

Port-forward the ArgoCD server, log in as admin, and generate the token:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
PF_PID=$!
sleep 2

ADMIN_PASS=$(kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d)

argocd login localhost:8080 \
  --username admin \
  --password "$ADMIN_PASS" \
  --insecure

OCTOPUS_TOKEN=$(argocd account generate-token --account octopus)

kill $PF_PID 2>/dev/null
```

Print the token clearly:

```bash
echo ""
echo "=== Octopus ArgoCD API Token ==="
echo "$OCTOPUS_TOKEN"
echo "================================"
echo "Copy this token into Octopus Deploy > Infrastructure > Argo CD Accounts."
```

### 5. Verify permissions

Re-open the port-forward and run all four permission checks:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
PF_PID=$!
sleep 2

argocd account can-i --auth-token "$OCTOPUS_TOKEN" get clusters '*'         # expect: yes
argocd account can-i --auth-token "$OCTOPUS_TOKEN" get applications '*'     # expect: yes
argocd account can-i --auth-token "$OCTOPUS_TOKEN" get logs '*/*'           # expect: yes
argocd account can-i --auth-token "$OCTOPUS_TOKEN" delete applications '*'  # expect: no

kill $PF_PID 2>/dev/null
```

If any of the first three return `no`, or the last returns `yes`, report the failure and stop — do not proceed silently with incorrect permissions.

### 6. Report

Print all ArgoCD pod statuses and remind the user to save the token — it cannot be retrieved again.

```bash
kubectl get pods -n argocd
```
