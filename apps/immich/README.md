# Immich Kustomize Chart

[English](#english) | [Français](#français)

---

## English

Kubernetes deployment of Immich with Kustomize, integrating Bitwarden External Secrets Operator and Tailscale Operator.

### Architecture

- **Immich Server**: Main photo/video management application
- **Immich Machine Learning**: ML service for facial recognition and semantic search
- **PostgreSQL**: Database with VectorChord for vector search
- **Redis**: Cache and job queue

### Prerequisites

- Kubernetes cluster
- Kustomize
- Bitwarden External Secrets Operator configured with a `ClusterSecretStore` named `bitwarden-cluster-secretstore`
- Tailscale Operator installed
- StorageClass `local-path` configured

### Bitwarden Secrets

Create the following secrets in Bitwarden Secrets Manager:

#### immich-db-username
```
key: immich-db-username
value: <db-username>
```

#### immich-db-password
```
key: immich-db-password
value: <db-password>
```

#### immich-redis-password
```
key: immich-redis-password
value: <redis-password>
```

### Configuration

#### Main Environment Variables

Edit `base/immich-server-deployment.yaml`:

- `TZ`: Timezone (default: Europe/Paris)
- `UPLOAD_LOCATION`: Internal storage path (do not modify)

#### Storage

Adjust sizes in `base/pvc.yaml`:

- `immich-library`: 500Gi (photos, videos, thumbnails)
- `immich-model-cache`: 10Gi (ML models)
- `postgres-data`: 20Gi

### Deployment

```bash
kubectl apply -k base/
```

### Access

The application is exposed via Tailscale Operator. The service will be automatically accessible on your Tailscale network.

To get the URL:
```bash
kubectl get ingress -n immich
```

The URL will be in the format `https://immich.<tailnet>.ts.net`

### Post-Installation Configuration

On first connection:
1. Create an administrator account
2. Configure application settings
3. Install the mobile app and connect

### Maintenance

#### Backups

Backup the following PVCs:
- `immich-library` (CRITICAL - contains all your photos/videos)
- `postgres-data`

#### Updates

```bash
kubectl rollout restart deployment/immich-server -n immich
kubectl rollout restart deployment/immich-machine-learning -n immich
```

#### Logs

```bash
kubectl logs -f deployment/immich-server -n immich
kubectl logs -f deployment/immich-machine-learning -n immich
```

### Support

Official documentation: https://docs.immich.app/

---

## Français

Déploiement Kubernetes d'Immich avec Kustomize, intégrant Bitwarden External Secrets Operator et Tailscale Operator.

### Architecture

- **Immich Server**: Application principale de gestion de photos/vidéos
- **Immich Machine Learning**: Service ML pour reconnaissance faciale et recherche sémantique
- **PostgreSQL**: Base de données avec VectorChord pour la recherche vectorielle
- **Redis**: Cache et file d'attente

### Prérequis

- Cluster Kubernetes
- Kustomize
- Bitwarden External Secrets Operator configuré avec un `ClusterSecretStore` nommé `bitwarden-cluster-secretstore`
- Tailscale Operator installé
- StorageClass `local-path` configurée

### Secrets Bitwarden

Créer les secrets suivants dans Bitwarden Secrets Manager :

#### immich-db-username
```
key: immich-db-username
value: <db-username>
```

#### immich-db-password
```
key: immich-db-password
value: <db-password>
```

#### immich-redis-password
```
key: immich-redis-password
value: <redis-password>
```

### Configuration

#### Variables d'environnement principales

Modifier `base/immich-server-deployment.yaml`:

- `TZ`: Fuseau horaire (défaut: Europe/Paris)
- `UPLOAD_LOCATION`: Chemin de stockage interne (ne pas modifier)

#### Stockage

Modifier les tailles dans `base/pvc.yaml`:

- `immich-library`: 500Gi (photos, vidéos, thumbs)
- `immich-model-cache`: 10Gi (modèles ML)
- `postgres-data`: 20Gi

### Déploiement

```bash
kubectl apply -k base/
```

### Accès

L'application est exposée via Tailscale Operator. Le service sera automatiquement accessible sur votre réseau Tailscale.

Pour obtenir l'URL:
```bash
kubectl get ingress -n immich
```

L'URL sera de la forme `https://immich.<tailnet>.ts.net`

### Configuration post-installation

Lors de la première connexion:
1. Créer un compte administrateur
2. Configurer les paramètres de l'application
3. Installer l'app mobile et se connecter

### Maintenance

#### Sauvegardes

Sauvegarder les PVCs:
- `immich-library` (CRITIQUE - contient toutes vos photos/vidéos)
- `postgres-data`

#### Mise à jour

```bash
kubectl rollout restart deployment/immich-server -n immich
kubectl rollout restart deployment/immich-machine-learning -n immich
```

#### Logs

```bash
kubectl logs -f deployment/immich-server -n immich
kubectl logs -f deployment/immich-machine-learning -n immich
```

### Support

Documentation officielle: https://docs.immich.app/
