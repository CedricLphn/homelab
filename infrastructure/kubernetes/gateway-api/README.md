# Gateway API CRDs

Installs the upstream [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) CRDs (standard channel).

## What this provides

CRDs consumed by Traefik (and any future Gateway-API-aware controller):

- `GatewayClass`
- `Gateway`
- `HTTPRoute`
- `ReferenceGrant`
- `GRPCRoute`

## Version

Pinned to `v1.4.0` in `kustomization.yaml` — bumped via Renovate.

## Apply

Deployed by ArgoCD via `argocd-apps/gateway-api-crds.yaml`. Manual apply:

```bash
kubectl apply -k infrastructure/kubernetes/gateway-api/
```
