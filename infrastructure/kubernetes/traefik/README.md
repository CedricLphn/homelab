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
- Cilium with LB-IPAM pool ready (`infrastructure/kubernetes/cilium/`)

## Install Traefik via Helm

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update

helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --version 33.2.1 \
  --values infrastructure/kubernetes/traefik/values.yaml
```

Verify Traefik installed and got a LoadBalancer IP from Cilium:

```bash
kubectl get pods -n traefik
kubectl get svc -n traefik traefik
# EXTERNAL-IP should match a value from the CiliumLoadBalancerIPPool
```

The `io.cilium/lb-l2-announce: "true"` label on the Service makes Cilium L2-announce the IP on the LAN.

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

## DNS — router

On the LAN router, add a static DNS entry pointing the wildcard to the Traefik LoadBalancer IP, and allow remote DNS queries (needed for WireGuard clients):

```
/ip dns set allow-remote-requests=yes
/ip dns static add regexp="^.+\\.internal\\.example\\.lan\\\$" address=<TRAEFIK-LB-IP>
```

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
