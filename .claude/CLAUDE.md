# Octopus Deploy Reference Docs

For any questions about how Octopus integrates with **Argo CD** (instances, authentication, annotations, steps, troubleshooting), read `.claude/docs/argocd-octopus.md` first.

For any questions about **Kubernetes** support in Octopus (agents, API targets, YAML/Helm/Kustomize steps, deployment verification, live object status, troubleshooting), read `.claude/docs/kubernetes-octopus.md` first.

# Summary

This is repository that stores deployment examples that is used in order to develop Kubernetes and ArgoCD integration in Octopus Deploy.

# Folder Types

- plain manifests
- helm charts (if `chart` subfolder exists)
- ArgoCD application (if `application.yaml` exists)