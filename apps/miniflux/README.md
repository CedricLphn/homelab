# Miniflux Kustomize Chart

Déploiement Kubernetes de Miniflux avec Kustomize, intégrant Bitwarden External Secrets Operator, Tailscale Operator et authentification OIDC via Authelia.

## Architecture

- **Miniflux**: Lecteur de flux RSS/Atom minimaliste et open source
- **PostgreSQL**: Base de données
- **Authelia OIDC**: Authentification OpenID Connect

## Prérequis

- Cluster Kubernetes (Talos Linux)
- Kustomize
- Bitwarden External Secrets Operator configuré avec un `ClusterSecretStore` nommé `bitwarden-cluster-secretstore`
- Tailscale Operator installé
- StorageClass `local-path` configurée
- Authelia configuré avec un client OIDC pour Miniflux

## Configuration Authelia

Ajouter le client suivant dans la configuration Authelia (`identity_providers.oidc.clients`):

```yaml
- client_id: 'miniflux'
  client_name: 'Miniflux'
  client_secret: '<hashed-secret>'
  public: false
  authorization_policy: 'two_factor'
  redirect_uris:
    - 'https://miniflux.tail3161aa.ts.net/oauth2/oidc/callback'
  scopes:
    - 'openid'
    - 'profile'
    - 'email'
  response_types:
    - 'code'
  grant_types:
    - 'authorization_code'
```

## Secrets Bitwarden

Créer les secrets suivants dans Bitwarden Secret Manager (key: value):

### miniflux-db-username
```
key: miniflux-db-username
value: <db-username>
```

### miniflux-db-password
```
key: miniflux-db-password
value: <db-password>
```

### miniflux-admin-username
```
key: miniflux-admin-username
value: <admin-username>
```

### miniflux-admin-password
```
key: miniflux-admin-password
value: <admin-password>
```

### miniflux-oidc-client-id
```
key: miniflux-oidc-client-id
value: miniflux
```

### miniflux-oidc-client-secret
```
key: miniflux-oidc-client-secret
value: <oidc-client-secret>
```

### miniflux-oidc-discovery-endpoint
```
key: miniflux-oidc-discovery-endpoint
value: https://auth.example.com
```

**Note**: L'URL de découverte OIDC ne doit PAS inclure `.well-known/openid-configuration` car Miniflux l'ajoute automatiquement.

## Configuration Miniflux

### Variables d'environnement principales

Modifier `base/miniflux-deployment.yaml`:

- `BASE_URL`: URL Tailscale de votre instance
- `OAUTH2_REDIRECT_URL`: URL de callback OIDC (doit correspondre à Authelia)
- `OAUTH2_USER_CREATION`: 1 pour créer automatiquement les utilisateurs OIDC

### Stockage

Modifier les tailles dans `base/pvc.yaml`:

- `postgres-data`: 10Gi

## Déploiement

```bash
kubectl apply -k miniflux/base/
```

## Accès

L'application est exposée via Tailscale Operator.

Pour obtenir l'URL Tailscale:
```bash
kubectl get ingress -n miniflux
```

URL d'accès : `https://miniflux.tail3161aa.ts.net`

## Configuration initiale

### Authentification

Miniflux supporte deux méthodes d'authentification :

1. **OIDC (recommandé)** : Cliquer sur le bouton de connexion OIDC et s'authentifier via Authelia
2. **Local** : Utiliser le compte admin créé avec les credentials Bitwarden

Les utilisateurs OIDC sont automatiquement créés lors de la première connexion si `OAUTH2_USER_CREATION=1`.

### Premier accès

1. Accéder à l'URL Tailscale
2. Se connecter via OIDC ou avec le compte admin
3. Configurer les flux RSS/Atom dans les réglages

## Fonctionnalités

- Interface minimaliste et rapide
- Support RSS, Atom et JSON Feed
- Mode lecture avec extraction du contenu complet
- Raccourcis clavier
- Thème clair/sombre
- API complète pour intégrations
- Applications mobiles tierces disponibles
- Filtres et règles de transformation de flux
- Import/export OPML

## Maintenance

### Sauvegardes

Sauvegarder le PVC:
- `postgres-data`

### Mise à jour

```bash
kubectl rollout restart deployment/miniflux -n miniflux
```

### Logs

```bash
kubectl logs -f deployment/miniflux -n miniflux
```

### Vérification de santé

```bash
kubectl exec -n miniflux deployment/miniflux -- curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/healthcheck --max-time 5
```

### Migrations de base de données

Les migrations sont exécutées automatiquement au démarrage si `RUN_MIGRATIONS=1`.

Pour forcer manuellement :
```bash
kubectl exec -n miniflux deployment/miniflux -- miniflux -migrate
```

### Gestion des utilisateurs

Créer un utilisateur admin supplémentaire :
```bash
kubectl exec -n miniflux deployment/miniflux -- miniflux -create-admin
```

## Support

Documentation officielle: https://miniflux.app/docs/
