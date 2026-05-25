# Traefik + Homelab Gateway

Traefik v3 implements the Kubernetes Gateway API in this cluster. A single cluster-wide `Gateway` accepts HTTPS traffic on the wildcard host `*.homelab.lastsector.lan`; each application declares its own `HTTPRoute` and attaches to this Gateway.

## What this provides

- `traefik` namespace
- `Certificate` `wildcard-internal-tls` issued by the `homelab-ca-issuer` ClusterIssuer (cf. `infrastructure/kubernetes/cert-manager/`)
- `Gateway` `homelab-gateway` with:
  - HTTPS listener on port 443, TLS-terminated with the wildcard cert
  - HTTP listener on port 80 (apps can declare HTTPRoutes to redirect 80→443)
  - `allowedRoutes.namespaces.from: All` — HTTPRoutes from any namespace can attach

## Prerequisites

- Gateway API CRDs installed (`infrastructure/kubernetes/gateway-api/`)
- cert-manager + Homelab Root CA + ClusterIssuer (`infrastructure/kubernetes/cert-manager/`)
- Cilium installed (`infrastructure/kubernetes/cilium/`)

## Network exposure model: `hostNetwork`

The Traefik pod runs in `hostNetwork: true` and binds directly to the node's ports `:80` and `:443`. This means the node's primary IP (`10.0.1.3` in this cluster) is the Gateway entry point. No `LoadBalancer` IP, no L2 announcement, no extra subnet required — the wildcard DNS just points at the node IP.

Trade-offs:
- ✅ Works with a single node IP, no LB-IPAM / no router reconfiguration
- ✅ Lower latency (no extra service indirection)
- ⚠️ Only one Traefik replica can run on a given node (ports 80/443 conflict)
- ⚠️ Ports 80/443 must be free on the node — no other host process can bind them

## Install Traefik via Helm

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update

helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --version 39.0.9 \
  --values infrastructure/kubernetes/traefik/values.yaml
```

> Pin chart 39.x (Traefik v3.6.x). Chart 40.x ships Traefik v3.7.1 which currently fails to start its Gateway provider against upstream Gateway API CRDs (it requests `v1.TLSRoute` which exists only at `v1alpha2`/`v1alpha3`).

Verify the pod is bound to the host network:

```bash
kubectl get pods -n traefik -o wide        # IP column should equal node IP
kubectl exec -n traefik deploy/traefik -- ss -tlnp | grep -E ':80|:443'
```

## Apply the Gateway + wildcard cert

Deployed by ArgoCD via `argocd-apps/traefik.yaml`. Manual apply:

```bash
kubectl apply -k infrastructure/kubernetes/traefik/
```

Verify:

```bash
kubectl get certificate wildcard-internal-tls -n traefik       # Ready=True
kubectl get gatewayclass traefik                               # Accepted=True
kubectl get gateway homelab-gateway -n traefik                 # Programmed=True, address listed
```

## DNS — LAN router

On the LAN router, add a static DNS entry mapping the wildcard to the node's IP, and allow remote DNS queries (needed for WireGuard clients).

For RouterOS-flavored CLIs:

```
/ip dns set allow-remote-requests=yes
/ip dns static add regexp="^.+\\.homelab\\.lastsector\\.lan\\\$" address=<NODE-IP>
```

For other routers (Pi-hole, AdGuard, OPNsense/Unbound) use their equivalent "local DNS / static A" feature with the same wildcard → node IP.

## Smoke test

```bash
curl -k https://anything.homelab.lastsector.lan/
# HTTP/2 404  — Gateway responds, no HTTPRoute matches yet. Expected.
```

After installing the CA on the client (cf. `cert-manager/README.md`), drop the `-k` flag and you should still get 404 with a green padlock.

## HTTPRoute pattern (used by every app)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: <app>
  namespace: <namespace>
spec:
  parentRefs:
    - name: homelab-gateway
      namespace: traefik
      sectionName: https
  hostnames:
    - <app>.homelab.lastsector.lan
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: <service>
          port: <port>
```

No `ReferenceGrant` is required because the Gateway already opens its listeners to routes from all namespaces, and the HTTPRoute's backend lives in the same namespace as the route.

## HTTP→HTTPS redirect (optional)

Each app can declare a second HTTPRoute attached to the `http` listener with a `RequestRedirect` filter:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: <app>-http-redirect
  namespace: <namespace>
spec:
  parentRefs:
    - name: homelab-gateway
      namespace: traefik
      sectionName: http
  hostnames:
    - <app>.homelab.lastsector.lan
  rules:
    - filters:
        - type: RequestRedirect
          requestRedirect:
            scheme: https
            statusCode: 301
```
