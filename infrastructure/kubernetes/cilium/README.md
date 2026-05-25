# Cilium — CNI + LB-IPAM + L2 announce

Cilium replaces Flannel (CNI), kube-proxy (eBPF service routing), and MetalLB (LoadBalancer IPAM + L2 ARP). One mobile piece instead of three.

## What this provides

- LoadBalancer service IPs allocated from a LAN-side pool (`CiliumLoadBalancerIPPool`)
- L2 ARP announcements so the LAN routes the LoadBalancer IP to the cluster node (`CiliumL2AnnouncementPolicy`)
- eBPF kube-proxy replacement (faster, no iptables service rules)

Cilium itself is installed out-of-band via Helm (matches the External Secrets pattern in this repo). Only the homelab-specific pool/announce config is GitOps-managed here.

## Prerequisites — Talos machine config

Cilium expects Flannel and kube-proxy to be disabled. Patch `private/talos/controlplane.yaml`:

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

## Apply the homelab pool + L2 announce

Before applying, edit `lb-ip-pool.yaml` and `l2-announcement.yaml`:

- `<LAN-LB-CIDR>` → a free CIDR on your LAN reserved for LoadBalancer IPs (e.g. `192.168.1.240/29` → 8 IPs `.240`–`.247`). It MUST NOT overlap the DHCP range of your router.
- `<NODE-NETWORK-INTERFACE>` → the NIC name on the Talos node (e.g. `eth0`, `eno1`, `enp1s0`). Check with `talosctl get links --nodes <CONTROL-PLANE-IP>`.

Deployed by ArgoCD via `argocd-apps/cilium.yaml`. Manual apply:

```bash
kubectl apply -k infrastructure/kubernetes/cilium/
```

Verify:

```bash
kubectl get ciliumloadbalancerippool homelab-lan-pool
kubectl get ciliuml2announcementpolicy homelab-l2-announce
```

## Validate with a test LoadBalancer

```bash
kubectl create deploy nginx-test --image=nginx --port=80
kubectl expose deploy nginx-test --type=LoadBalancer --port=80
kubectl label svc nginx-test io.cilium/lb-l2-announce=true
kubectl get svc nginx-test                # EXTERNAL-IP from the pool
curl http://<EXTERNAL-IP>                 # nginx welcome page
kubectl delete deploy,svc nginx-test
```

## Notes

- Cilium ignores the `node.kubernetes.io/exclude-from-external-load-balancers` label that MetalLB respects, so a single-node control-plane works out of the box for L2 announce.
- `loadBalancer.mode: snat` is the default and works on any NIC. `dsr` is faster but requires NIC support — switch later if benchmarks show benefit.
- Hubble UI is disabled to keep the footprint small. Enable it later by setting `hubble.ui.enabled: true` and reinstalling.
