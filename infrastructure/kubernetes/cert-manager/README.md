# cert-manager + Homelab Root CA

Provides automatic TLS certificate issuance for internal services on `*.homelab.lastsector.lan` via a self-signed root CA.

## What this provides

- `cert-manager` namespace (PodSecurity privileged)
- Bootstrap `Issuer` (selfSigned) → generates the Homelab Root CA `Certificate` (ECDSA P-256, 10 years)
- `ClusterIssuer` `homelab-ca-issuer` referencing the root CA secret — used by any `Certificate` or Gateway annotation cluster-wide

The wildcard `*.homelab.lastsector.lan` certificate itself lives in `infrastructure/kubernetes/traefik/` next to the Gateway that consumes it.

## Prerequisites

cert-manager itself is installed out-of-band via Helm (same pattern as External Secrets Operator in this repo).

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.18.2 \
  --set crds.enabled=true \
  --set extraArgs={--feature-gates=ExperimentalGatewayAPISupport=true}
```

The `ExperimentalGatewayAPISupport` flag enables cert-manager to honor `cert-manager.io/cluster-issuer` annotations on Gateway resources directly. Optional here since the Certificate resource is declared explicitly.

## Apply the homelab CA

Deployed by ArgoCD via `argocd-apps/cert-manager.yaml`. Manual apply:

```bash
kubectl apply -k infrastructure/kubernetes/cert-manager/
```

Verify:

```bash
kubectl get certificate homelab-root-ca -n cert-manager        # Ready=True
kubectl get clusterissuer homelab-ca-issuer                    # Ready=True
```

## Distribute the Root CA to client devices

Export the CA cert and install it in the trust store of every device that will browse internal apps:

```bash
kubectl get secret homelab-root-ca -n cert-manager \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > homelab-ca.crt
```

- **macOS** : double-click `homelab-ca.crt` → Keychain Access → set to "Always Trust"
- **iOS** : AirDrop the file → install Profile → Settings → General → About → Certificate Trust Settings → enable full trust for "Homelab Root CA"
- **Android** : Settings → Security → Encryption & credentials → Install a certificate → CA certificate
- **Linux (system)** : `sudo cp homelab-ca.crt /usr/local/share/ca-certificates/ && sudo update-ca-certificates`
- **Firefox** : import via Settings → Privacy & Security → Certificates → View Certificates → Authorities → Import

## Rotation

The root CA renews automatically 30 days (720h) before expiry. Devices keep trusting the same key since cert-manager re-uses the existing private key on renewal.
