# kind-cilium-gatewayapi-test

A minimal, reproducible local Kubernetes test environment that demonstrates a modern, production-pattern networking stack using entirely open-source tooling — with zero cloud infrastructure and zero DNS configuration required.

## What this does

Spins up a fully functional 3-node KinD cluster on your machine, replaces the default CNI and kube-proxy with Cilium, wires up the Kubernetes Gateway API for ingress, and deploys a real test application to validate the entire path end-to-end.

**Full traffic path:**

```
curl http://podinfo.172.18.50.1.nip.io
  → Docker bridge network
  → Cilium L2 ARP announcement (172.18.50.1)
  → Kubernetes Gateway API (Cilium GatewayClass)
  → HTTPRoute
  → podinfo pod
```

## Stack

| Tool | Role |
|------|------|
| [KinD](https://kind.sigs.k8s.io/) | Local multi-node Kubernetes cluster inside Docker |
| [Cilium](https://cilium.io/) | eBPF-based CNI; replaces kube-proxy, provides L2 LoadBalancer and Gateway API support |
| [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) | Modern replacement for Ingress (v1.4.1, including experimental CRDs) |
| [Helm](https://helm.sh/) | Deploys the test application |
| [Task](https://taskfile.dev/) | Task runner orchestrating all setup steps |
| [podinfo](https://github.com/stefanprodan/podinfo) | Lightweight Go microservice used as the test application |
| [nip.io](https://nip.io/) | Wildcard DNS — routes `podinfo.172.18.50.1.nip.io` to the LoadBalancer IP, no DNS setup needed |

## Cluster design

- **3 nodes**: 1 control-plane + 2 workers
- **No default CNI**: `kindnet` is disabled; Cilium fills the role
- **No kube-proxy**: Cilium handles all service routing via eBPF
- **Custom subnets**: pods on `10.10.0.0/16`, services on `10.11.0.0/16`
- **LoadBalancer IP**: `172.18.50.1` allocated via `CiliumLoadBalancerIPPool` and advertised via ARP on the Docker bridge, making it reachable from the host

## Usage

### Full setup (single command)

```sh
task install
```

This runs the following steps in order:

1. `task kind` — creates the KinD cluster
2. `task gwapi` — installs Gateway API CRDs
3. `task cilium` — installs Cilium (kube-proxy replacement + Gateway API + L2 announcements)
4. `task lb` — applies the IP pool, L2 announcement policy, and Gateway object
5. `task podinfo` — deploys podinfo via Helm
6. `task test` — runs a continuous curl loop against the live endpoint

### Individual tasks

```sh
task kind       # Create the cluster
task delete     # Tear down the cluster
task gwapi      # Install Gateway API CRDs
task cilium     # Install Cilium
task lb         # Apply Cilium networking manifests
task podinfo    # Deploy the test application
task test       # Curl loop to verify load balancing
```

## How the networking works

1. **Cilium L2 announcement** (`cilium/announcement.yaml`): Cilium answers ARP requests for `172.18.50.1` on `eth0`, making the IP reachable from the Docker host network without any static routes.
2. **IP pool** (`cilium/pool.yaml`): Assigns `172.18.50.1` as the single LoadBalancer IP that Cilium allocates to the `Gateway`.
3. **Gateway** (`cilium/gateway.yaml`): A `Gateway` resource using `gatewayClassName: cilium` listens on port 80. Cilium claims it and binds the LoadBalancer IP.
4. **HTTPRoute** (defined in `app/podinfo.yaml` Helm values): Routes traffic from `podinfo.172.18.50.1.nip.io` to the podinfo `ClusterIP` service.

## Application

`podinfo` is deployed with:
- 2 replicas
- `ClusterIP` service (traffic enters via Gateway, not NodePort/LoadBalancer)
- HTTPRoute attached to the `gateway` Gateway
- Minimal resource requests (1m CPU, 16Mi memory)
