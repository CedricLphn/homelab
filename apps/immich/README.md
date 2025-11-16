# Immich Kustomize Chart

Déploiement Kubernetes d'Immich avec Kustomize, intégrant Bitwarden External Secrets Operator et Tailscale Operator.

## Architecture

- **Immich Server**: Application principale de gestion de photos/vidéos
- **Immich Machine Learning**: Service ML pour reconnaissance faciale et recherche sémantique
- **PostgreSQL**: Base de données avec VectorChord pour la recherche vectorielle
- **Redis**: Cache et file d'attente

## Prérequis

- Cluster Kubernetes
- Kustomize
- Bitwarden External Secrets Operator configuré avec un `ClusterSecretStore` nommé `bitwarden-cluster-secretstore`
- Tailscale Operator installé
- StorageClass `local-path` configurée

## Secrets Bitwarden

Créer les secrets suivants dans Bitwarden Secret Manager (key: value):

### immich-db-username
```
key: immich-db-username
value: <db-username>
```

### immich-db-password
```
key: immich-db-password
value: <db-password>
```

## Configuration

### Variables d'environnement principales

Modifier `base/immich-server-deployment.yaml`:

- `TZ`: Fuseau horaire (défaut: Europe/Paris)
- `UPLOAD_LOCATION`: Chemin de stockage interne (ne pas modifier)

### Stockage

Modifier les tailles dans `base/pvc.yaml`:

- `immich-library`: 500Gi (photos, vidéos, thumbs)
- `immich-model-cache`: 10Gi (modèles ML)
- `postgres-data`: 20Gi

## Déploiement

```bash
kubectl apply -k base/
```

## Accès

L'application est exposée via Tailscale Operator. Le service sera automatiquement accessible sur votre réseau Tailscale.

Pour obtenir l'URL:
```bash
kubectl get ingress -n immich
```

L'URL sera de la forme `https://immich.<tailnet>.ts.net`

## Configuration post-installation

Lors de la première connexion:
1. Créer un compte administrateur
2. Configurer les paramètres de l'application
3. Installer l'app mobile et connecter-vous

## Maintenance

### Sauvegardes

Sauvegarder les PVCs:
- `immich-library` (CRITIQUE - contient toutes vos photos/vidéos)
- `postgres-data`

### Mise à jour

```bash
kubectl rollout restart deployment/immich-server -n immich
kubectl rollout restart deployment/immich-machine-learning -n immich
```

### Logs

```bash
kubectl logs -f deployment/immich-server -n immich
kubectl logs -f deployment/immich-machine-learning -n immich
```

## Support

Documentation officielle: https://docs.immich.app/
