---
name: bootstrap-kind
description: "Creates a local kind Kubernetes cluster. Usage: /bootstrap-kind [cluster-name] — defaults to 'dev' if no name is given."
allowed-tools: ["Bash"]
---

Create a local kind Kubernetes cluster.

## Parsing args

- First positional arg = cluster name. Default: `dev`.

## Steps

### 1. Check for an existing cluster

```bash
kind get clusters
```

If a cluster with the target name already exists, ask the user whether to delete and recreate it. Do not delete without explicit confirmation.

```bash
kind delete cluster --name <cluster-name>
```

### 2. Create the cluster

```bash
kind create cluster --name <cluster-name>
```

### 3. Verify connectivity

```bash
kubectl cluster-info --context kind-<cluster-name>
kubectl get nodes
```

Report the active context and node status. Done.
