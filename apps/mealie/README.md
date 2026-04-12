# Mealie

Gestionnaire de recettes self-hosted avec import depuis le web, recherche, planification de repas et liste de courses.

## Composants

| Composant | Image | Port | Role |
|-----------|-------|------|------|
| Mealie | `ghcr.io/mealie-recipes/mealie:v3.14.0` | 9000 | Application FastAPI + frontend Nuxt |

## Architecture

- **Base de donnees** : PostgreSQL partagee (`shared-services/postgres`), DB `mealie`
- **Stockage** : PVC `mealie-data` 10Gi en `local-path` pour `/app/data` (recettes, images, backups)
- **Authentification** : Authelia via OIDC — groupe `mealie-users` obligatoire, groupe global `admins` pour les droits admin
- **Ingress** : Tailscale (`https://mealie.tail<ID>.ts.net`)
- **Langue** : francais (fr-FR) configurable par utilisateur en UI, et par defaut au niveau site dans Settings

## Secrets Bitwarden

| Secret | Usage |
|--------|-------|
| `mealie-db-password` | Mot de passe Postgres (user `mealie`) |
| `mealie-oidc-client-id` | Client ID Authelia |
| `mealie-oidc-client-secret` | Client secret Authelia |
| `mealie-oidc-wellknown-url` | URL discovery OIDC Authelia (contient l'URL Tailscale privee) |

## ConfigMap privee

Le `BASE_URL` est dans un ConfigMap `mealie-urls` non committe :

```
private/configmaps/mealie-urls-configmap.yaml
```

Ce ConfigMap doit etre applique manuellement apres creation du namespace (premier deploiement seulement) :

```bash
kubectl apply -f private/configmaps/mealie-urls-configmap.yaml
```

## Deploiement

Gere automatiquement par ArgoCD via `argocd-apps/mealie.yaml`. Prerequis avant le premier sync :

1. Creer les secrets dans Bitwarden (voir tableau ci-dessus)
2. Configurer le client OIDC `mealie` dans Authelia et creer le groupe `mealie-users`
3. Creer la DB et l'utilisateur Postgres (voir section suivante)
4. Appliquer le ConfigMap `mealie-urls` prive
5. Push sur `main` — ArgoCD synchronise automatiquement

## Creation DB (une fois)

Le script postgres-init ne tourne qu'au premier boot de Postgres. Pour ajouter la DB apres coup :

```bash
kubectl exec -n shared-services deployment/postgres -- \
  psql -U postgres -c "CREATE USER mealie WITH PASSWORD '<password>';"
kubectl exec -n shared-services deployment/postgres -- \
  psql -U postgres -c "CREATE DATABASE mealie OWNER mealie;"
kubectl exec -n shared-services deployment/postgres -- \
  psql -U postgres -c "GRANT ALL PRIVILEGES ON DATABASE mealie TO mealie;"
```

## Validation

```bash
kubectl get pods -n mealie
kubectl get externalsecret -n mealie
kubectl get ingress -n mealie
curl -sk https://mealie.tail<ID>.ts.net/api/app/about
```

`externalsecret` doit etre `SecretSynced`, le pod `Running` et `Ready`, et le endpoint `/api/app/about` retourne du JSON.

## OIDC Authelia

Le groupe `admins` existant est reutilise pour les droits admin (`OIDC_ADMIN_GROUP=admins`). Les utilisateurs membres de `mealie-users` peuvent se connecter ; ceux membres de `admins` obtiennent automatiquement les droits admin au premier login — pas de promotion manuelle necessaire, contrairement a Paperless.
