# kind-cilium-gatewayapi-test

A minimal, reproducible local Kubernetes test environment that demonstrates a modern, production-pattern networking stack using entirely open-source tooling — with zero cloud infrastructure and zero DNS configuration required.

## What this does

Spins up a fully functional 3-node KinD cluster on your machine, replaces the default CNI and kube-proxy with Cilium, wires up both the Kubernetes Gateway API and the Ingress controller for ingress, and deploys two real test applications to validate both paths end-to-end.

**Traffic paths:**

```
# Gateway API
curl http://podinfo.172.18.50.1.nip.io
  → Docker bridge network
  → Cilium L2 ARP announcement (172.18.50.1)
  → Kubernetes Gateway API (Cilium GatewayClass)
  → HTTPRoute
  → podinfo-gwapi pod

# Ingress
curl http://podinfo.172.18.50.2.nip.io
  → Docker bridge network
  → Cilium L2 ARP announcement (172.18.50.2)
  → Cilium Ingress Controller (shared mode)
  → Ingress
  → podinfo-ingress pod
```

## Stack

| Tool | Role |
|------|------|
| [KinD](https://kind.sigs.k8s.io/) | Local multi-node Kubernetes cluster inside Docker |
| [Cilium](https://cilium.io/) | eBPF-based CNI; replaces kube-proxy, provides L2 LoadBalancer, Gateway API and Ingress support |
| [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) | Modern replacement for Ingress (v1.4.1, including experimental CRDs) |
| [Helm](https://helm.sh/) | Deploys the test applications |
| [Task](https://taskfile.dev/) | Task runner orchestrating all setup steps |
| [podinfo](https://github.com/stefanprodan/podinfo) | Lightweight Go microservice used as the test application |
| [nip.io](https://nip.io/) | Wildcard DNS — routes hostnames to LoadBalancer IPs, no DNS setup needed |
| [Hubble](https://docs.cilium.io/en/stable/observability/hubble/) | Cilium's built-in network observability layer |

## Cluster design

- **3 nodes**: 1 control-plane + 2 workers
- **No default CNI**: `kindnet` is disabled; Cilium fills the role
- **No kube-proxy**: Cilium handles all service routing via eBPF
- **Custom subnets**: pods on `10.10.0.0/16`, services on `10.11.0.0/16`
- **Two LoadBalancer IPs** allocated via `CiliumLoadBalancerIPPool`, advertised via ARP on the Docker bridge:
  - `172.18.50.1` — Gateway API
  - `172.18.50.2` — Ingress Controller (shared mode)

## Usage

### Full setup (single command)

```sh
task install
```

This runs the following steps in order:

1. `task kind` — creates the KinD cluster
2. `task gwapi` — installs Gateway API CRDs
3. `task cilium` — installs Cilium (kube-proxy replacement + Gateway API + Ingress + L2 announcements + Hubble)
4. `task lb` — applies the IP pool, L2 announcement policy, and Gateway object
5. `task podinfo-gwapi` — deploys podinfo via Gateway API
6. `task podinfo-ingress` — deploys podinfo via Ingress

### Individual tasks

```sh
task kind             # Create the cluster
task delete           # Tear down the cluster
task gwapi            # Install Gateway API CRDs
task cilium           # Install Cilium
task lb               # Apply Cilium networking manifests
task podinfo-gwapi    # Deploy test application via Gateway API
task podinfo-ingress  # Deploy test application via Ingress
task test-gwapi       # Curl loop to verify Gateway API load balancing
task test-ingress     # Curl loop to verify Ingress load balancing
task update-vips      # Sync all VIP references across manifests
```

### Observability

```sh
# Open Hubble UI in browser
cilium hubble ui

# Watch live flows on the CLI
cilium hubble port-forward &
hubble observe
```

## How the networking works

1. **Cilium L2 announcement** (`cilium/announcement.yaml`): Cilium answers ARP requests for both VIPs on `eth0`, making them reachable from the Docker host network without any static routes.
2. **IP pool** (`cilium/pool.yaml`): Defines two `/32` blocks — `.1` for Gateway API, `.2` for Ingress.
3. **Gateway** (`cilium/gateway.yaml`): A `Gateway` resource using `gatewayClassName: cilium` listens on port 80 and is pinned to `172.18.50.1` via the `lbipam.cilium.io/ips` annotation.
4. **Ingress Controller**: Cilium runs in shared mode — one `cilium-ingress` Service in `kube-system` handles all Ingress resources, pinned to `172.18.50.2` via `ingressController.service.loadBalancerIP`.
5. **HTTPRoute** (defined in `app/podinfo-gwapi.yaml` Helm values): Routes traffic from `podinfo.172.18.50.1.nip.io` to the `podinfo-gwapi` ClusterIP service.
6. **Ingress** (defined in `app/podinfo-ingress.yaml` Helm values): Routes traffic from `podinfo.172.18.50.2.nip.io` to the `podinfo-ingress` ClusterIP service.

## Applications

Two separate Helm releases of podinfo are deployed for comparison:

| Release | Path | VIP | Hostname |
|---|---|---|---|
| `podinfo-gwapi` | Gateway API → HTTPRoute | `172.18.50.1` | `podinfo.172.18.50.1.nip.io` |
| `podinfo-ingress` | Ingress Controller | `172.18.50.2` | `podinfo.172.18.50.2.nip.io` |

Both use 2 replicas, a `ClusterIP` service, and minimal resource requests (1m CPU, 16Mi memory). The `podinfo-ingress` release uses a distinct UI color and message to make the two visually distinguishable.
