# Wallabag Kustomize Chart

Déploiement Kubernetes de Wallabag avec Kustomize, intégrant Bitwarden External Secrets Operator et Tailscale Operator.

## Architecture

- **Wallabag**: Application de sauvegarde et lecture d'articles web (read-it-later)
- **PostgreSQL**: Base de données
- **Redis**: Cache et file d'attente de tâches

## Prérequis

- Cluster Kubernetes (Talos Linux)
- Kustomize
- Bitwarden External Secrets Operator configuré avec un `ClusterSecretStore` nommé `bitwarden-cluster-secretstore`
- Tailscale Operator installé
- StorageClass `local-path` configurée

## Secrets Bitwarden

Créer les secrets suivants dans Bitwarden Secret Manager (key: value):

### wallabag-db-username
```
key: wallabag-db-username
value: <db-username>
```

### wallabag-db-password
```
key: wallabag-db-password
value: <db-password>
```

### wallabag-secret-key
```
key: wallabag-secret-key
value: <symfony-secret-key-aleatoire>
```

**IMPORTANT**: Les secrets (clé secrète et mots de passe) ne doivent PAS contenir les caractères suivants qui posent problème avec Symfony : `&`, `%`, `=`. De plus, ne commencez PAS un secret par `#` car il serait interprété comme un commentaire. Utilisez uniquement des caractères alphanumériques, `-`, `_`, `~`, `@`, `!`, `*`, `/`, `+` et `#` (mais pas en début de chaîne).

## Configuration Wallabag

### Variables d'environnement principales

Modifier `base/wallabag-deployment.yaml`:

- `SYMFONY__ENV__DOMAIN_NAME`: URL Tailscale de votre instance
- `SYMFONY__ENV__LOCALE`: Langue (défaut: fr)
- `SYMFONY__ENV__FOSUSER_REGISTRATION`: true pour permettre l'inscription publique
- `SYMFONY__ENV__FOSUSER_CONFIRMATION`: true pour envoyer un email de confirmation
- `SYMFONY__ENV__FROM_EMAIL`: Adresse email pour les envois (activation, etc.)

### Stockage

Modifier les tailles dans `base/pvc.yaml`:

- `wallabag-data`: 10Gi (base de données SQLite si utilisée, cache)
- `wallabag-images`: 20Gi (images téléchargées localement)
- `postgres-data`: 10Gi
- `redis-data`: 1Gi

## Déploiement

```bash
kubectl apply -k base/
```

## Accès

L'application est exposée via Tailscale Operator.

Pour obtenir l'URL Tailscale:
```bash
kubectl get ingress -n wallabag
```

URL d'accès : `https://wallabag.tail3161aa.ts.net`

## Configuration initiale

Lors de la première connexion:
1. Créer un compte utilisateur (si registration activée) ou utiliser les credentials par défaut: `wallabag:wallabag`
2. Modifier le mot de passe par défaut
3. Configurer les paramètres de l'application dans les réglages

## Fonctionnalités

- Sauvegarde d'articles web pour lecture hors ligne
- Support des flux RSS
- Tags et catégories
- Export en ePub, PDF, MOBI, etc.
- Applications mobiles disponibles (iOS/Android)
- Extensions navigateur (Chrome, Firefox)
- Import depuis Pocket, Instapaper, etc.

## Maintenance

### Sauvegardes

Sauvegarder les PVCs:
- `wallabag-data`
- `wallabag-images`
- `postgres-data`

### Mise à jour

```bash
kubectl rollout restart deployment/wallabag -n wallabag
```

### Logs

```bash
kubectl logs -f deployment/wallabag -n wallabag
```

### Migration de base de données

Si une mise à jour nécessite une migration de base de données:
```bash
kubectl exec -n wallabag deployment/wallabag -- /var/www/wallabag/bin/console doctrine:migrations:migrate --env=prod --no-interaction
```

## Support

Documentation officielle: https://doc.wallabag.org/
