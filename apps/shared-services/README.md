# Shared Services

[English](#english) | [Français](#français)

---

## English

Shared services for PostgreSQL and Redis used by multiple homelab applications.

### Architecture

- **PostgreSQL 16**: Shared database with separate databases
  - `paperless`: Paperless-ngx database
  - `wallabag`: Wallabag database
  - `miniflux`: Miniflux database
- **Redis 7**: Shared cache and queue with numbered databases
  - Database 0: Immich
  - Database 1: Paperless-ngx
  - Database 2: Wallabag

### Prerequisites

- Kubernetes cluster (Talos Linux)
- Kustomize
- Bitwarden External Secrets Operator configured with a `ClusterSecretStore` named `bitwarden-cluster-secretstore`
- StorageClass `local-path` configured

### Bitwarden Secrets

Create the following secrets in Bitwarden Secret Manager (key: value):

#### shared-postgres-username
```
key: shared-postgres-username
value: <postgres-admin-username>
```

#### shared-postgres-password
```
key: shared-postgres-password
value: <postgres-admin-password>
```

**IMPORTANT**: Avoid problematic special characters: `&`, `%`, `=`, and `#` at the beginning of the string.

### Deployment

```bash
kubectl apply -k shared-services/base/
```

### Service Access

#### PostgreSQL

Connect from a pod in the cluster:
```bash
kubectl run -it --rm postgres-client --image=postgres:16-alpine --restart=Never -- \
  psql -h postgres.shared-services.svc.cluster.local -U <username> -d <database>
```

#### Redis

Connect from a pod in the cluster:
```bash
kubectl run -it --rm redis-client --image=redis:7-alpine --restart=Never -- \
  redis-cli -h redis.shared-services.svc.cluster.local
```

### Application Usage

#### PostgreSQL Configuration

In application deployments, use:
```yaml
- name: POSTGRES_HOST
  value: postgres.shared-services.svc.cluster.local
- name: POSTGRES_PORT
  value: "5432"
- name: POSTGRES_DB
  value: <database-name>  # paperless, wallabag, or miniflux
```

#### Redis Configuration

In application deployments, use:
```yaml
- name: REDIS_HOST
  value: redis.shared-services.svc.cluster.local
- name: REDIS_PORT
  value: "6379"
- name: REDIS_DB
  value: "<number>"  # 0 for Immich, 1 for Paperless, 2 for Wallabag
```

### Maintenance

#### PostgreSQL Backups

Backup a specific database:
```bash
kubectl exec -n shared-services deployment/postgres -- \
  pg_dump -U <username> <database> > backup-<database>.sql
```

Backup all databases:
```bash
kubectl exec -n shared-services deployment/postgres -- \
  pg_dumpall -U <username> > backup-all.sql
```

#### PostgreSQL Restore

Restore a database:
```bash
kubectl exec -i -n shared-services deployment/postgres -- \
  psql -U <username> -d <database> < backup-<database>.sql
```

#### Redis Backups

Manual backup:
```bash
kubectl exec -n shared-services deployment/redis -- redis-cli SAVE
kubectl cp shared-services/<redis-pod>:/data/dump.rdb ./redis-backup.rdb
```

#### Logs

PostgreSQL:
```bash
kubectl logs -f deployment/postgres -n shared-services
```

Redis:
```bash
kubectl logs -f deployment/redis -n shared-services
```

#### Health Checks

PostgreSQL:
```bash
kubectl exec -n shared-services deployment/postgres -- pg_isready
```

Redis:
```bash
kubectl exec -n shared-services deployment/redis -- redis-cli ping
```

#### Interactive Connection

PostgreSQL:
```bash
kubectl exec -it -n shared-services deployment/postgres -- psql -U <username>
```

Redis:
```bash
kubectl exec -it -n shared-services deployment/redis -- redis-cli
```

### Storage

- `postgres-data`: 20Gi (PostgreSQL data)
- `redis-data`: 2Gi (Redis data with AOF)

### Resources

#### PostgreSQL
- Requests: 512Mi RAM, 250m CPU
- Limits: 1Gi RAM, 500m CPU

#### Redis
- Requests: 256Mi RAM, 100m CPU
- Limits: 512Mi RAM, 250m CPU

### Applications Using These Services

- **Paperless-ngx**: PostgreSQL (database: paperless), Redis (database: 1)
- **Wallabag**: PostgreSQL (database: wallabag), Redis (database: 2)
- **Miniflux**: PostgreSQL (database: miniflux)
- **Immich**: Redis (database: 0)

---

## Français

Services partagés pour PostgreSQL et Redis utilisés par plusieurs applications du homelab.

### Architecture

- **PostgreSQL 16**: Base de données partagée avec databases séparées
  - `paperless`: Base Paperless-ngx
  - `wallabag`: Base Wallabag
  - `miniflux`: Base Miniflux
- **Redis 7**: Cache et file d'attente partagés avec databases numérotées
  - Database 0: Immich
  - Database 1: Paperless-ngx
  - Database 2: Wallabag

### Prérequis

- Cluster Kubernetes (Talos Linux)
- Kustomize
- Bitwarden External Secrets Operator configuré avec un `ClusterSecretStore` nommé `bitwarden-cluster-secretstore`
- StorageClass `local-path` configurée

### Secrets Bitwarden

Créer les secrets suivants dans Bitwarden Secret Manager (key: value):

#### shared-postgres-username
```
key: shared-postgres-username
value: <postgres-admin-username>
```

#### shared-postgres-password
```
key: shared-postgres-password
value: <postgres-admin-password>
```

**IMPORTANT**: Éviter les caractères spéciaux problématiques: `&`, `%`, `=`, et `#` en début de chaîne.

### Déploiement

```bash
kubectl apply -k shared-services/base/
```

### Accès aux services

#### PostgreSQL

Connexion depuis un pod dans le cluster:
```bash
kubectl run -it --rm postgres-client --image=postgres:16-alpine --restart=Never -- \
  psql -h postgres.shared-services.svc.cluster.local -U <username> -d <database>
```

#### Redis

Connexion depuis un pod dans le cluster:
```bash
kubectl run -it --rm redis-client --image=redis:7-alpine --restart=Never -- \
  redis-cli -h redis.shared-services.svc.cluster.local
```

### Utilisation par les applications

#### Configuration PostgreSQL

Dans les deployments des applications, utiliser:
```yaml
- name: POSTGRES_HOST
  value: postgres.shared-services.svc.cluster.local
- name: POSTGRES_PORT
  value: "5432"
- name: POSTGRES_DB
  value: <nom-database>  # paperless, wallabag, ou miniflux
```

#### Configuration Redis

Dans les deployments des applications, utiliser:
```yaml
- name: REDIS_HOST
  value: redis.shared-services.svc.cluster.local
- name: REDIS_PORT
  value: "6379"
- name: REDIS_DB
  value: "<numero>"  # 0 pour Immich, 1 pour Paperless, 2 pour Wallabag
```

### Maintenance

#### Backups PostgreSQL

Backup d'une database spécifique:
```bash
kubectl exec -n shared-services deployment/postgres -- \
  pg_dump -U <username> <database> > backup-<database>.sql
```

Backup de toutes les databases:
```bash
kubectl exec -n shared-services deployment/postgres -- \
  pg_dumpall -U <username> > backup-all.sql
```

#### Restore PostgreSQL

Restore d'une database:
```bash
kubectl exec -i -n shared-services deployment/postgres -- \
  psql -U <username> -d <database> < backup-<database>.sql
```

#### Backups Redis

Backup manuel:
```bash
kubectl exec -n shared-services deployment/redis -- redis-cli SAVE
kubectl cp shared-services/<redis-pod>:/data/dump.rdb ./redis-backup.rdb
```

#### Logs

PostgreSQL:
```bash
kubectl logs -f deployment/postgres -n shared-services
```

Redis:
```bash
kubectl logs -f deployment/redis -n shared-services
```

#### Vérification de santé

PostgreSQL:
```bash
kubectl exec -n shared-services deployment/postgres -- pg_isready
```

Redis:
```bash
kubectl exec -n shared-services deployment/redis -- redis-cli ping
```

#### Connexion interactive

PostgreSQL:
```bash
kubectl exec -it -n shared-services deployment/postgres -- psql -U <username>
```

Redis:
```bash
kubectl exec -it -n shared-services deployment/redis -- redis-cli
```

### Stockage

- `postgres-data`: 20Gi (données PostgreSQL)
- `redis-data`: 2Gi (données Redis avec AOF)

### Ressources

#### PostgreSQL
- Requests: 512Mi RAM, 250m CPU
- Limits: 1Gi RAM, 500m CPU

#### Redis
- Requests: 256Mi RAM, 100m CPU
- Limits: 512Mi RAM, 250m CPU

### Applications utilisant ces services

- **Paperless-ngx**: PostgreSQL (database: paperless), Redis (database: 1)
- **Wallabag**: PostgreSQL (database: wallabag), Redis (database: 2)
- **Miniflux**: PostgreSQL (database: miniflux)
- **Immich**: Redis (database: 0)
