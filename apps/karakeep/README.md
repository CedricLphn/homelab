# Karakeep

Application de bookmarking self-hosted (anciennement Hoarder) avec tagging automatique et recherche full-text.

## Composants

| Composant | Image | Port | Role |
|-----------|-------|------|------|
| Karakeep | `ghcr.io/karakeep-app/karakeep:0.31.0` | 3000 | Application principale (Next.js + SQLite) |
| Meilisearch | `getmeili/meilisearch:v1.12.8` | 7700 | Moteur de recherche full-text |
| Chrome | `gcr.io/zenika-hub/alpine-chrome:124` | 9222 | Navigateur headless pour scraping/screenshots |

## Architecture

- **Base de donnees** : SQLite interne (WAL mode) — pas de PostgreSQL externe
- **Recherche** : Meilisearch dedie dans le namespace
- **Scraping** : Chrome headless pour extraction de metadonnees et screenshots
- **Authentification** : Authelia via OIDC (inscriptions desactivees)
- **Ingress** : Tailscale (`https://karakeep.tail<ID>.ts.net`)

## Secrets Bitwarden

| Secret | Usage |
|--------|-------|
| `karakeep-nextauth-secret` | Cle JWT pour NextAuth |
| `karakeep-meili-master-key` | Cle master Meilisearch |
| `karakeep-oidc-client-id` | Client ID Authelia |
| `karakeep-oidc-client-secret` | Client secret Authelia |
| `karakeep-oidc-wellknown-url` | URL discovery OIDC Authelia |

## Deploiement

```bash
kubectl apply -k apps/karakeep/base/
```

## Verification

```bash
kubectl get pods -n karakeep
kubectl get externalsecrets -n karakeep
kubectl get ingress -n karakeep
kubectl logs -f deployment/karakeep -n karakeep
```

## Stockage

- `karakeep-data` (10Gi) : Donnees application + base SQLite
- `meilisearch-data` (5Gi) : Index de recherche Meilisearch
