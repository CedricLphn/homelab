# Cilium — CNI + kube-proxy replacement (eBPF)

Cilium replaces Flannel (CNI) and kube-proxy (eBPF service routing).

Ingress / LoadBalancer is handled by Traefik in `hostNetwork` mode (binds directly to the node IP), so no LB-IPAM or L2 announcement is needed here.

## What this provides

- CNI: pod networking, NetworkPolicy, identity-based security
- Service routing: eBPF replacement for kube-proxy (`kubeProxyReplacement: true`)
- Optional observability: Hubble Relay (Hubble UI disabled)

Cilium itself is installed out-of-band via Helm (matches the External Secrets pattern in this repo). There is no homelab-specific Cilium config in this directory beyond `values.yaml` — the Kustomize resource list is intentionally empty.

## Prerequisites — Talos machine config

Cilium expects Flannel and kube-proxy to be disabled. Patch `private/talos/controlplane.yaml` (see `infrastructure/talos/patches/cilium-cni.yaml`):

```yaml
cluster:
  network:
    cni:
      name: none
  proxy:
    disabled: true
```

Apply with reboot:

```bash
talosctl apply-config --nodes <CONTROL-PLANE-IP> --file private/talos/controlplane.yaml --mode=reboot
```

The cluster will be unreachable until Cilium is installed (next step).

## Install Cilium via Helm

```bash
helm repo add cilium https://helm.cilium.io
helm repo update

helm install cilium cilium/cilium \
  --namespace kube-system \
  --version 1.18.2 \
  --values infrastructure/kubernetes/cilium/values.yaml
```

Wait for Cilium to become healthy:

```bash
cilium status --wait
kubectl get pods -n kube-system -l k8s-app=cilium
```

## Notes

- `kubeProxyReplacement: true` + `k8sServiceHost: 127.0.0.1` + `k8sServicePort: 7445` rely on Talos KubePrism (local API LB on the node).
- Hubble UI is disabled to keep the footprint small. Enable it later by setting `hubble.ui.enabled: true` and reinstalling.
- No `CiliumLoadBalancerIPPool` or `CiliumL2AnnouncementPolicy` because Traefik exposes itself via `hostNetwork` on the node IP — no LoadBalancer Services in use.
