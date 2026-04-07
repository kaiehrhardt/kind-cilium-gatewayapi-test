# kind-cilium-gatewayapi-test

A minimal, reproducible local Kubernetes test environment that demonstrates a modern, production-pattern networking stack using entirely open-source tooling — with zero cloud infrastructure and zero DNS configuration required.

## What this does

Spins up a fully functional 3-node KinD cluster on your machine, replaces the default CNI and kube-proxy with Cilium, wires up both the Kubernetes Gateway API and the Ingress controller for ingress, installs cert-manager with a self-signed CA, and deploys two real test applications to validate both HTTP and HTTPS paths end-to-end.

**Traffic paths:**

```
# Gateway API (HTTP)
curl http://podinfo.172.18.50.1.nip.io
  → Docker bridge network
  → Cilium L2 ARP announcement (172.18.50.1)
  → Kubernetes Gateway API (Cilium GatewayClass)
  → HTTPRoute
  → podinfo-gwapi pod

# Gateway API (HTTPS — TLS terminated at Gateway)
curl https://podinfo.172.18.50.1.nip.io
  → Docker bridge network
  → Cilium L2 ARP announcement (172.18.50.1)
  → Kubernetes Gateway API — TLS terminated with cert-manager certificate
  → HTTPRoute
  → podinfo-gwapi pod

# Gateway API (TLS Passthrough — TLS terminated at Pod)
curl https://podinfo-passthrough.172.18.50.1.nip.io:8443
  → Docker bridge network
  → Cilium L2 ARP announcement (172.18.50.1)
  → Kubernetes Gateway API — TLSRoute, traffic forwarded encrypted
  → podinfo-passthrough pod (terminates TLS with cert-manager certificate)

# Ingress (HTTP + HTTPS)
curl http(s)://podinfo.172.18.50.2.nip.io
  → Docker bridge network
  → Cilium L2 ARP announcement (172.18.50.2)
  → Cilium Ingress Controller (shared mode)
  → Ingress — TLS terminated with cert-manager certificate
  → podinfo-ingress pod
```

## Stack

| Tool | Role |
|------|------|
| [KinD](https://kind.sigs.k8s.io/) | Local multi-node Kubernetes cluster inside Docker |
| [Cilium](https://cilium.io/) | eBPF-based CNI; replaces kube-proxy, provides L2 LoadBalancer, Gateway API and Ingress support |
| [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) | Modern replacement for Ingress (v1.4.1, including experimental CRDs) |
| [cert-manager](https://cert-manager.io/) | Kubernetes-native certificate management; issues and rotates TLS certificates |
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
5. `task cert-manager` — installs cert-manager via Helm
6. `task cert-manager-issuers` — applies the self-signed ClusterIssuer and TLS certificates
7. `task podinfo-gwapi` — deploys podinfo via Gateway API (HTTP + HTTPS)
8. `task podinfo-ingress` — deploys podinfo via Ingress (HTTP + HTTPS)
9. `task podinfo-passthrough` — deploys podinfo with pod-level TLS via TLSRoute (Passthrough)

### Individual tasks

```sh
task kind                   # Create the cluster
task delete                 # Tear down the cluster
task gwapi                  # Install Gateway API CRDs
task cilium                 # Install Cilium
task lb                     # Apply Cilium networking manifests
task cert-manager           # Install cert-manager
task cert-manager-issuers   # Apply ClusterIssuer and Certificate resources
task podinfo-gwapi          # Deploy test application via Gateway API
task podinfo-ingress        # Deploy test application via Ingress
task podinfo-passthrough    # Deploy test application with TLS Passthrough
task test-gwapi             # Curl loop — Gateway API HTTP
task test-gwapi-https       # Curl loop — Gateway API HTTPS (validated against self-signed CA)
task test-ingress           # Curl loop — Ingress HTTP
task test-ingress-https     # Curl loop — Ingress HTTPS (validated against self-signed CA)
task test-passthrough       # Curl loop — TLS Passthrough on port 8443 (validated against self-signed CA)
task update-vips            # Sync all VIP/hostname references across manifests
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
3. **Gateway** (`cilium/gateway.yaml`): A `Gateway` resource using `gatewayClassName: cilium` with three listeners, all pinned to `172.18.50.1` via the `lbipam.cilium.io/ips` annotation:
   - **Port 80** (`HTTP`): plain HTTP, handled via `HTTPRoute`
   - **Port 443** (`HTTPS`): TLS terminated at the Gateway using the `gateway-tls-secret` issued by cert-manager, then forwarded as plain HTTP to the pod via `HTTPRoute`
   - **Port 8443** (`TLS Passthrough`): encrypted traffic forwarded as-is to the pod via `TLSRoute`; the pod terminates TLS itself using `podinfo-passthrough-tls-secret`
4. **Ingress Controller**: Cilium runs in shared mode — one `cilium-ingress` Service in `kube-system` handles all Ingress resources, pinned to `172.18.50.2` via `ingressController.service.loadBalancerIP`.
5. **HTTPRoute** (defined in `app/podinfo-gwapi.yaml` Helm values): Routes traffic from `podinfo.172.18.50.1.nip.io` to the `podinfo-gwapi` ClusterIP service, attached to both the HTTP and HTTPS listeners.
6. **Ingress** (defined in `app/podinfo-ingress.yaml` Helm values): Routes traffic from `podinfo.172.18.50.2.nip.io` to the `podinfo-ingress` ClusterIP service, with a TLS block referencing `podinfo-ingress-tls-secret`.

## TLS / cert-manager

cert-manager is installed in the `cert-manager` namespace. The PKI setup (`cert-manager/cluster-issuer.yaml`) is a two-step chain:

1. **`selfsigned-bootstrap`** (`ClusterIssuer`): A root self-signed issuer used only to create the CA certificate.
2. **`selfsigned-ca`** (`Certificate`): A CA certificate stored as `selfsigned-ca-secret` in the `cert-manager` namespace.
3. **`selfsigned`** (`ClusterIssuer`): The main issuer, backed by the CA above. All application certificates are issued by this issuer.

| Certificate | Namespace | Secret | Covers |
|---|---|---|---|
| `gateway-tls` | `default` | `gateway-tls-secret` | `podinfo.172.18.50.1.nip.io` |
| `podinfo-ingress-tls` | `default` | `podinfo-ingress-tls-secret` | `podinfo.172.18.50.2.nip.io` |
| `podinfo-passthrough-tls` | `default` | `podinfo-passthrough-tls-secret` | `podinfo-passthrough.172.18.50.1.nip.io` |
| `hubble-tls` | `kube-system` | `hubble-tls-secret` | `hubble.172.18.50.2.nip.io` |

The HTTPS test tasks (`test-gwapi-https`, `test-ingress-https`, `test-passthrough`) fetch the CA certificate directly from the cluster secret and pass it to `curl --cacert` — the certificate is properly validated, not skipped with `-k`.

**TLS Passthrough vs. Gateway Termination:**

| | TLS Termination at Gateway | TLS Passthrough |
|---|---|---|
| Gateway sees plaintext | yes | no |
| Header-based routing | yes | no (SNI only) |
| HTTP→HTTPS redirect | yes | no |
| Cert lives in | Gateway Secret | Pod Secret |
| Gateway API resource | `HTTPRoute` | `TLSRoute` |
| Cilium CRD channel | standard | experimental |

## Applications

Three separate Helm releases of podinfo are deployed for comparison:

| Release | Path | VIP | Port | Hostname |
|---|---|---|---|---|
| `podinfo-gwapi` | Gateway API → HTTPRoute | `172.18.50.1` | 80 / 443 | `podinfo.172.18.50.1.nip.io` |
| `podinfo-passthrough` | Gateway API → TLSRoute (Passthrough) | `172.18.50.1` | 8443 | `podinfo-passthrough.172.18.50.1.nip.io` |
| `podinfo-ingress` | Ingress Controller | `172.18.50.2` | 80 / 443 | `podinfo.172.18.50.2.nip.io` |

All releases use 2 replicas, a `ClusterIP` service, and minimal resource requests (1m CPU, 16Mi memory). Each release uses a distinct UI color and message to make the three visually distinguishable.
