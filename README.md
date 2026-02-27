# MicroShift — Single-Node OpenShift via Docker Compose

Run a single-node [MicroShift](https://github.com/microshift-io/microshift) (OKD) cluster locally using Docker Compose. Includes the OpenShift web console and RabbitMQ operators matching common OpenShift custom resource controllers in a production namespace.

## Prerequisites

- Docker Desktop with Compose V2
- `kubectl` CLI installed ([install guide](https://kubernetes.io/docs/tasks/tools/))

## Quick Start

```bash
cd microshift

# Start the cluster (first run pulls images and takes ~5 minutes)
docker compose up -d

# Point kubectl at the auto-generated kubeconfig
export KUBECONFIG=$(pwd)/kubeconfig

# Verify the cluster is running
kubectl get nodes
```

Expected output:

```
NAME         STATUS   ROLES                         AGE   VERSION
microshift   Ready    control-plane,master,worker   5m    v1.34.2
```

## What Gets Installed

On startup, the `cluster-setup` container automatically installs:

| Component | Purpose |
|---|---|
| **TopoLVM CSI** | Default StorageClass for PVC dynamic provisioning (backed by loopback LVM) |
| **Prometheus Operator CRDs** | `PrometheusRule`, `ServiceMonitor`, `PodMonitor`, etc. (CRDs only, no full stack) |
| **cert-manager** | TLS certificate management (prerequisite for topology operator) |
| **RabbitMQ Cluster Operator** | Manages `RabbitmqCluster` custom resources |
| **RabbitMQ Messaging Topology Operator** | Manages `Binding`, `Exchange`, `Queue`, `User`, `Permission`, `Policy`, `Vhost`, etc. |
| **OpenShift Console** | Web UI dashboard at http://localhost:9000 |

These match the custom resource controllers deployed on a typical OpenShift namespace.

### Verify Operators

```bash
export KUBECONFIG=$(pwd)/kubeconfig

# Check operator pods are running
kubectl get pods -n cert-manager
kubectl get pods -n rabbitmq-system

# List available RabbitMQ CRDs
kubectl api-resources --api-group=rabbitmq.com
```

Expected RabbitMQ CRDs:

```
bindings, exchanges, federations, operatorpolicies, permissions,
policies, queues, rabbitmqclusters, schemareplications, shovels,
superstreams, topicpermissions, users, vhosts
```

### Create a RabbitMQ Cluster

```bash
# Create a single-node RabbitMQ cluster
kubectl apply -f - <<EOF
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: my-rabbitmq
spec:
  replicas: 1
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
EOF

# Watch it come up
kubectl get rabbitmqclusters
kubectl get pods -l app.kubernetes.io/component=rabbitmq

# Create a queue via the Topology Operator
kubectl apply -f - <<EOF
apiVersion: rabbitmq.com/v1beta1
kind: Queue
metadata:
  name: my-queue
spec:
  name: my-queue
  rabbitmqClusterReference:
    name: my-rabbitmq
EOF

kubectl get queues.rabbitmq.com
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

# List RabbitMQ resources
kubectl get rabbitmqclusters,queues.rabbitmq.com,exchanges.rabbitmq.com,bindings.rabbitmq.com -A

# View logs
kubectl logs -f deployment/rabbitmq-cluster-operator -n rabbitmq-system
```

## Architecture

The stack runs five containers:

| Container | Purpose |
|---|---|
| `microshift-lvm-setup` | One-shot: creates a loopback LVM volume group for TopoLVM (matches upstream `cluster_manager.sh`) |
| `microshift` | The MicroShift cluster (systemd, CRI-O, kubelet, API server) |
| `microshift-kubeconfig` | One-shot: extracts and patches kubeconfig for localhost access |
| `microshift-cluster-setup` | One-shot: installs Prometheus CRDs, cert-manager, RabbitMQ operators, and console RBAC |
| `openshift-console` | The OpenShift web console UI (port 9000) |

The `lvm-setup`, `kubeconfig`, and `cluster-setup` containers exit after completing their tasks. The `microshift` and `openshift-console` containers stay running.

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

Operator manifests (cert-manager, RabbitMQ Cluster Operator, RabbitMQ Messaging Topology Operator) are downloaded from their GitHub releases at startup via `curl`. Versions are pinned in the `cluster-setup` service's environment variables in `docker-compose.yml`:

```yaml
environment:
  - PROMETHEUS_OPERATOR_VERSION=v0.82.2
  - CERT_MANAGER_VERSION=v1.19.4
  - RABBITMQ_CLUSTER_OPERATOR_VERSION=v2.19.1
  - RABBITMQ_TOPOLOGY_OPERATOR_VERSION=v1.18.3
```

### Resource Limits

The MicroShift container is configured with resource limits suitable for running on a laptop:

```yaml
cpus: 2
mem_limit: 4g
```

To upgrade an operator, bump the version and restart: `docker compose up -d`.

## Troubleshooting

**Node shows `NotReady` after start:**
MicroShift takes 1-2 minutes to fully initialize. Wait and retry `kubectl get nodes`.

**Console not loading at localhost:9000:**
Check if the console container is running: `docker ps -a --filter name=openshift-console`. View logs with `docker logs openshift-console`.

**`kubectl` returns TLS errors:**
Make sure you're using the auto-generated kubeconfig (not a stale copy). The generated kubeconfig uses `insecure-skip-tls-verify: true` to work around the self-signed CA.

**Operator pods stuck in `ImagePullBackOff`:**
CRI-O inside MicroShift enforces fully-qualified image names. If updating operator manifests, ensure all `image:` references include the registry prefix (e.g., `docker.io/rabbitmqoperator/...` not just `rabbitmqoperator/...`).

**TopoLVM pods in CrashLoopBackOff:**
TopoLVM requires an LVM volume group. The `lvm-setup` init container creates a 1GB sparse-file-backed loopback VG (`myvg1`) before MicroShift starts, matching the upstream `cluster_manager.sh` approach. If the VG is missing (e.g., after a Docker Desktop restart that clears loopback devices), do a full reset: `docker compose down -v && docker compose up -d`.

**Image pull failures inside MicroShift:**
CRI-O inside MicroShift runs on the host's native architecture. On Apple Silicon, only arm64 images can be pulled by pods. The OpenShift console runs as a Docker Compose sidecar (with Rosetta amd64 emulation) to work around this.
