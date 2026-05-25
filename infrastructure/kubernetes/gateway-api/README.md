# Gateway API CRDs

Installs the upstream [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) CRDs (experimental channel).

## What this provides

CRDs consumed by Traefik (and any future Gateway-API-aware controller):

- `GatewayClass`, `Gateway`, `HTTPRoute`, `GRPCRoute`, `ReferenceGrant` (standard)
- `TCPRoute`, `TLSRoute`, `UDPRoute`, `BackendTLSPolicy` (experimental)
- `XListenerSets`, `XBackendTrafficPolicies` (experimental, alpha)

The Traefik provider watches all of these on startup; missing CRDs cause the gateway provider to never become ready. The experimental channel is therefore mandatory for Traefik v3.6.x.

## Version

Pinned to `v1.3.0` in `kustomization.yaml`. Traefik v3.6.15 reads TLSRoute at `v1alpha2` which is the version shipped by Gateway API v1.3. Upgrading to v1.4 moves TLSRoute to `v1alpha3` and breaks Traefik startup — bump in lock-step with the Traefik version.

## Apply

Deployed by ArgoCD via `argocd-apps/gateway-api-crds.yaml`. Manual apply (use server-side, the CRDs exceed the 256 KiB last-applied annotation limit):

```bash
kubectl apply -k infrastructure/kubernetes/gateway-api/ --server-side --force-conflicts
```
