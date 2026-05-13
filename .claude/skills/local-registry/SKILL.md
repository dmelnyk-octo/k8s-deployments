---
name: local-registry
description: "Manages the local OCI registry and its UI. Usage: /local-registry [start|stop] — defaults to 'start'."
allowed-tools: ["Bash"]
---

Manage the local OCI registry (`local-registry`) and its UI (`registry-ui`).

## Parsing args

- First positional arg = action: `start` or `stop`. Default: `start`.

## Stop

```bash
docker stop local-registry registry-ui
```

Confirm both containers are stopped. Done — skip remaining steps.

## Start

### 1. Start the registry

If the container already exists (just stopped), start it:

```bash
docker start local-registry
```

If it does not exist, create it:

```bash
docker run -d -p 5000:5000 --name local-registry --restart no \
  -v local-registry-data:/var/lib/registry \
  -e REGISTRY_HTTP_HEADERS_Access-Control-Allow-Origin='["http://localhost:8090"]' \
  -e REGISTRY_HTTP_HEADERS_Access-Control-Allow-Methods='["HEAD","GET","OPTIONS","DELETE"]' \
  -e REGISTRY_HTTP_HEADERS_Access-Control-Allow-Headers='["Authorization","Accept","Cache-Control"]' \
  -e REGISTRY_HTTP_HEADERS_Access-Control-Expose-Headers='["Docker-Content-Digest"]' \
  registry:2
```

### 2. Start the registry UI

If the container already exists, start it:

```bash
docker start registry-ui
```

If it does not exist, create it:

```bash
docker run -d -p 8090:80 --name registry-ui --restart no \
  -e SINGLE_REGISTRY=true \
  -e REGISTRY_URL=http://host.docker.internal:5000 \
  joxit/docker-registry-ui:latest
```

### 3. Confirm

Verify both containers are running:

```bash
docker ps --filter name=local-registry --filter name=registry-ui --format "table {{.Names}}\t{{.Status}}"
```

Report:
- Registry: `http://localhost:5000`
- UI: `http://localhost:8090`
