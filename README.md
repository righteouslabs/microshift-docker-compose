# MicroShift — Single-Node OpenShift via Docker Compose

Run a single-node [MicroShift](https://github.com/microshift-io/microshift) (OKD) cluster locally using Docker Compose. Includes an OpenShift web console accessible from your browser.

## Prerequisites

- Docker Desktop with Compose V2
- `kubectl` CLI installed ([install guide](https://kubernetes.io/docs/tasks/tools/))

## Quick Start

```bash
cd microshift

# Start the cluster (first run pulls images and takes ~2 minutes)
docker compose up -d

# Point kubectl at the auto-generated kubeconfig
export KUBECONFIG=$(pwd)/kubeconfig

# Verify the cluster is running
kubectl get nodes
```

Expected output:

```
NAME         STATUS   ROLES                         AGE   VERSION
microshift   Ready    control-plane,master,worker   2m    v1.34.2
```

## OpenShift Console (Dashboard)

The stack automatically deploys the OpenShift web console as a sidecar container.

Once `docker compose up -d` completes, open your browser to:

**http://localhost:9000**

No login is required — authentication is disabled for local development.

The console provides a full OpenShift UI for browsing pods, deployments, services, namespaces, RBAC, storage, and more.

> **Note:** Some OpenShift-specific menus (Operators, Builds, ImageStreams) will show errors since MicroShift is a minimal distribution that doesn't include the full OpenShift operator framework.

## Common kubectl Commands

```bash
# Set KUBECONFIG for the current shell session
export KUBECONFIG=$(pwd)/kubeconfig

# List all nodes
kubectl get nodes

# List all pods across all namespaces
kubectl get pods -A

# List services
kubectl get svc -A

# Deploy a test workload
kubectl create deployment hello --image=nginx
kubectl expose deployment hello --port=80 --type=NodePort

# View logs
kubectl logs -f deployment/hello

# Clean up
kubectl delete deployment hello
kubectl delete svc hello
```

## Architecture

The stack runs four containers:

| Container | Purpose |
|---|---|
| `microshift` | The MicroShift cluster (systemd, CRI-O, kubelet, API server) |
| `microshift-kubeconfig` | One-shot: extracts and patches kubeconfig for localhost access |
| `microshift-console-setup` | One-shot: creates RBAC and bearer token for the console |
| `openshift-console` | The OpenShift web console UI (port 9000) |

The `kubeconfig` and `console-setup` containers exit after completing their setup tasks. The `microshift` and `openshift-console` containers stay running.

## Ports

| Port | Service |
|---|---|
| `6443` | Kubernetes API server |
| `9000` | OpenShift web console |

## Stopping and Starting

```bash
# Stop the cluster (preserves data)
docker compose down

# Start again (data persists in the microshift-data volume)
docker compose up -d

# Full reset (destroys all cluster data)
docker compose down -v
```

## Files

```
microshift/
├── docker-compose.yml           # Stack definition
├── config/
│   ├── api_server.yaml          # MicroShift API server config (SAN names)
│   └── crio-runtime.conf        # CRI-O runtime config (for amd64/Rosetta)
├── manifests/
│   └── openshift-console.yaml   # Console RBAC (ServiceAccount + ClusterRoleBinding)
├── kubeconfig                   # Auto-generated (gitignored)
└── README.md
```

## Troubleshooting

**Node shows `NotReady` after start:**
MicroShift takes 1-2 minutes to fully initialize. Wait and retry `kubectl get nodes`.

**Console not loading at localhost:9000:**
Check if the console container is running: `docker ps -a --filter name=openshift-console`. View logs with `docker logs openshift-console`.

**`kubectl` returns TLS errors:**
Make sure you're using the auto-generated kubeconfig (not a stale copy). The generated kubeconfig uses `insecure-skip-tls-verify: true` to work around the self-signed CA.

**TopoLVM pods in CrashLoopBackOff:**
This is expected. TopoLVM requires LVM volumes on the host, which aren't available inside a container. It does not affect cluster functionality.

**Image pull failures inside MicroShift:**
CRI-O inside MicroShift runs on the host's native architecture. On Apple Silicon, only arm64 images can be pulled by pods. The OpenShift console runs as a Docker Compose sidecar (with Rosetta amd64 emulation) to work around this.
