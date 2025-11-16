# Infrastructure Architecture / Architecture de l'infrastructure

[English](#english) | [Français](#français)

---

## English

### Overview

This homelab is built on a **bare-metal Kubernetes cluster** running **Talos Linux**, designed for security, simplicity, and self-hosted applications.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    Tailscale Network                        │
│                  (Private Mesh VPN)                         │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              Kubernetes Cluster (v1.34.0)                   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Control Plane Node(s)                        │  │
│  │  - API Server                                        │  │
│  │  - Scheduler                                         │  │
│  │  - Controller Manager                                │  │
│  │  - etcd                                              │  │
│  │  - KubePrism (local LB on :7445)                     │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Worker Node(s) - Optional                    │  │
│  │  - Kubelet                                           │  │
│  │  - Container Runtime                                 │  │
│  │  - Application Pods                                  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                 Talos Linux (v1.11.1)                       │
│  - Immutable OS                                             │
│  - API-driven configuration                                 │
│  - No SSH access                                            │
│  - Secure by default                                        │
└─────────────────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                  Physical Hardware                          │
│  - Local disk storage (/dev/sda)                            │
│  - Network: Private subnet (192.168.x.x or 10.x.x.x)        │
└─────────────────────────────────────────────────────────────┘
```

### Component Stack

#### 1. Operating System: Talos Linux

**Why Talos?**
- **Immutable**: Read-only rootfs, all changes via API
- **Secure**: No SSH, no shell, minimal attack surface
- **Kubernetes-native**: Designed exclusively for K8s
- **Simple**: API-driven configuration, no package manager

**Features Enabled:**
- KubePrism: Local load balancer for API server HA
- Host DNS: CoreDNS forwarding with host caching
- Control plane scheduling: Apps can run on control plane nodes
- Disk quotas: Project quota support for storage

**Version**: v1.11.1

#### 2. Kubernetes Cluster

**Version**: v1.34.0

**Networking:**
- **Pod Subnet**: `10.244.0.0/16`
- **Service Subnet**: `10.96.0.0/12`
- **DNS**: CoreDNS (cluster.local)
- **CNI**: None (default) / Flannel (optional)

**Control Plane:**
- Endpoint: `https://<CONTROL-PLANE-IP>:6443`
- HA-ready with KubePrism local load balancer
- Scheduling allowed on control plane nodes

**Pod Security:**
- Admission controller enabled
- Default: `baseline` enforcement
- Audit/Warn: `restricted` policy
- kube-system namespace exempted

#### 3. Storage: local-path-provisioner

**Storage Class**: `local-path`

**Benefits:**
- Simple, no external dependencies
- Good performance for homelab use
- Automatic PV provisioning

**Configuration:**
- Storage path: `/var/lib/local-path-provisioner`
- Mounted with `bind`, `rshared`, `rw` options
- Works on both control plane and worker nodes

**Volumes:**
- Immich: 500Gi (library) + 10Gi (cache) + 20Gi (postgres)
- Paperless: 10Gi (data) + 50Gi (media) + 10Gi (consume) + 10Gi (export) + postgres
- Wallabag: 10Gi (data) + 20Gi (images) + postgres
- Miniflux: 10Gi (postgres)
- Shared PostgreSQL: 20Gi
- Shared Redis: 2Gi

#### 4. Secret Management: External Secrets Operator + Bitwarden

**Architecture:**
```
Bitwarden Secrets Manager
         ↓
   (HTTPS API)
         ↓
Bitwarden SDK Server (in-cluster)
   Port 9998, self-signed cert
         ↓
External Secrets Operator
         ↓
Kubernetes Secrets (auto-synced every 1h)
         ↓
Application Pods
```

**Benefits:**
- Centralized secret storage outside cluster
- Automatic secret rotation
- No secrets in Git
- Easy secret updates (change in Bitwarden → auto-sync)

**Components:**
- **ClusterSecretStore**: Cluster-wide Bitwarden configuration
- **ExternalSecret**: Per-app secret definitions
- **Bitwarden SDK Server**: HTTPS proxy with self-signed certificate

#### 5. Ingress: Tailscale

**Why Tailscale?**
- Zero-trust network access
- No port forwarding needed
- Automatic HTTPS with Tailscale certificates
- Access from anywhere securely

**Configuration:**
- Tailscale Operator installed in cluster
- Ingress resources with `ingressClassName: tailscale`
- Each service gets `https://<service>.tail<id>.ts.net` URL
- Integrated with Kubernetes service discovery

**No traditional ingress controller** (Nginx, Traefik) needed!

#### 6. Authentication: Authelia (OIDC/OAuth2)

**Supported Apps:**
- Paperless-ngx: OAuth2
- Miniflux: OIDC

**Not Supported:**
- Wallabag: No OIDC support for user authentication
- Immich: Local authentication only

**Benefits:**
- Single sign-on (SSO) where possible
- Centralized user management
- 2FA support
- Session management

### Application Architecture

#### Shared Services Pattern

```
┌─────────────────────────────────────────────────────────┐
│              Shared Services Namespace                  │
│                                                         │
│  ┌──────────────────┐      ┌──────────────────┐        │
│  │  PostgreSQL 16   │      │     Redis 7      │        │
│  │                  │      │                  │        │
│  │  Databases:      │      │  DBs:            │        │
│  │  - paperless     │      │  0: immich       │        │
│  │  - wallabag      │      │  1: paperless    │        │
│  │  - miniflux      │      │  2: wallabag     │        │
│  └──────────────────┘      └──────────────────┘        │
│           ↑                         ↑                   │
│           │                         │                   │
└───────────┼─────────────────────────┼───────────────────┘
            │                         │
            │                         │
┌───────────┼─────────────────────────┼───────────────────┐
│           │                         │                   │
│  ┌────────┴────────┐   ┌───────────┴──────────┐        │
│  │  Paperless-ngx  │   │  Wallabag            │        │
│  │  Namespace      │   │  Namespace           │        │
│  └─────────────────┘   └──────────────────────┘        │
│                                                         │
│  ┌─────────────────┐   ┌──────────────────────┐        │
│  │  Miniflux       │   │  Immich              │        │
│  │  Namespace      │   │  Namespace           │        │
│  └─────────────────┘   └──────────────────────┘        │
│                           (has own PostgreSQL)         │
└─────────────────────────────────────────────────────────┘
```

**Rationale:**
- Reduce resource usage (single PostgreSQL/Redis instance)
- Centralized database management
- Easier backups
- Immich has dedicated PostgreSQL (requires VectorChord extension)

#### Per-Application Structure

Each app in `apps/` follows this pattern:

```
<app-name>/
├── README.md              # French documentation
└── base/
    ├── kustomization.yaml # Resource aggregation + namespace + labels
    ├── namespace.yaml     # Namespace with privileged PodSecurity
    ├── pvc.yaml          # Persistent Volume Claims
    ├── external-secrets.yaml # Bitwarden secret references
    ├── <component>-deployment.yaml
    ├── <component>-service.yaml
    └── tailscale-ingress.yaml
```

**Kustomize** (not Helm):
- Simple, declarative
- No templating complexity
- Easy to understand and modify
- Version control friendly

### Network Flow

```
User Device
    ↓
Tailscale Network (encrypted mesh)
    ↓
Tailscale Ingress (in-cluster)
    ↓
Kubernetes Service (ClusterIP)
    ↓
Application Pod
    ↓
Shared PostgreSQL/Redis (if applicable)
```

### Security Model

#### Defense in Depth

1. **Network Layer**:
   - Tailscale zero-trust network
   - No public exposure
   - Encrypted mesh VPN

2. **OS Layer**:
   - Talos immutable OS
   - No SSH access
   - API-driven only
   - Minimal attack surface

3. **Kubernetes Layer**:
   - Pod Security Admission (baseline/restricted)
   - RBAC enabled
   - Network policies (future enhancement)
   - Namespace isolation

4. **Application Layer**:
   - OAuth/OIDC where supported
   - Strong passwords (generated, stored in Bitwarden)
   - Regular updates

5. **Secret Management**:
   - External Secrets Operator
   - No secrets in Git
   - Centralized in Bitwarden
   - Automatic rotation capability

#### Threat Model

**Protected Against:**
- ✅ Public internet exposure
- ✅ SSH-based attacks (no SSH)
- ✅ Secret leakage in Git
- ✅ Unauthorized cluster access
- ✅ OS-level compromise (immutable OS)

**Not Protected Against:**
- ⚠️ Compromised Tailscale credentials
- ⚠️ Kubernetes RBAC bypass
- ⚠️ Application-level vulnerabilities
- ⚠️ Physical access to hardware

### Backup Strategy

**Current State**: Manual backups

**Recommended Additions:**
- Velero for Kubernetes resources
- Database dumps to external storage
- Regular etcd snapshots
- Version control for all manifests (this repo!)

### Monitoring & Observability

**Current State**: Kubernetes native (kubectl)

**Future Enhancements:**
- Prometheus + Grafana
- Loki for log aggregation
- Alertmanager for notifications
- Uptime monitoring

### Disaster Recovery

**Recovery Plan:**

1. **Talos Node Failure**:
   - Reinstall Talos on new hardware
   - Apply saved configuration from `private/`
   - Rejoin cluster

2. **Cluster Failure**:
   - Restore etcd from snapshot
   - Or rebuild from scratch (all config in Git)
   - Restore PV data from backups

3. **Data Loss**:
   - Restore from database dumps
   - Restore PV data from backups

### Scalability

**Current Capacity:**
- Single control plane node (can schedule workloads)
- Can add worker nodes as needed
- Storage limited by local disk size

**Scaling Options:**
- Add worker nodes for compute
- Add distributed storage (Rook/Ceph, Longhorn)
- Implement multi-control-plane HA
- External load balancer for API server

### Cost

**Total Cost**: ~€0/month 🎉

- No cloud fees
- No managed Kubernetes fees
- Electricity cost only
- Uses existing hardware

---

## Français

### Vue d'ensemble

Ce homelab est construit sur un **cluster Kubernetes bare-metal** exécutant **Talos Linux**, conçu pour la sécurité, la simplicité et les applications auto-hébergées.

### Diagramme d'architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Réseau Tailscale                         │
│                  (VPN Mesh Privé)                           │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              Cluster Kubernetes (v1.34.0)                   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Nœud(s) Control Plane                        │  │
│  │  - Serveur API                                       │  │
│  │  - Scheduler                                         │  │
│  │  - Controller Manager                                │  │
│  │  - etcd                                              │  │
│  │  - KubePrism (LB local sur :7445)                    │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Nœud(s) Worker - Optionnel                   │  │
│  │  - Kubelet                                           │  │
│  │  - Runtime de conteneurs                            │  │
│  │  - Pods d'applications                               │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                 Talos Linux (v1.11.1)                       │
│  - OS immuable                                              │
│  - Configuration pilotée par API                            │
│  - Pas d'accès SSH                                          │
│  - Sécurisé par défaut                                      │
└─────────────────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                  Matériel physique                          │
│  - Stockage sur disque local (/dev/sda)                     │
│  - Réseau: Sous-réseau privé (192.168.x.x ou 10.x.x.x)      │
└─────────────────────────────────────────────────────────────┘
```

### Pile de composants

#### 1. Système d'exploitation : Talos Linux

**Pourquoi Talos ?**
- **Immuable** : Rootfs en lecture seule, tous les changements via API
- **Sécurisé** : Pas de SSH, pas de shell, surface d'attaque minimale
- **Kubernetes-natif** : Conçu exclusivement pour K8s
- **Simple** : Configuration pilotée par API, pas de gestionnaire de paquets

**Fonctionnalités activées :**
- KubePrism : Load balancer local pour HA du serveur API
- Host DNS : Forwarding CoreDNS avec cache DNS hôte
- Ordonnancement control plane : Les apps peuvent tourner sur les nœuds control plane
- Quotas disque : Support des quotas projet pour le stockage

**Version** : v1.11.1

#### 2. Cluster Kubernetes

**Version** : v1.34.0

**Réseau :**
- **Sous-réseau pods** : `10.244.0.0/16`
- **Sous-réseau services** : `10.96.0.0/12`
- **DNS** : CoreDNS (cluster.local)
- **CNI** : Aucun (par défaut) / Flannel (optionnel)

**Control Plane :**
- Endpoint : `https://<IP-CONTROL-PLANE>:6443`
- Prêt pour HA avec KubePrism load balancer local
- Ordonnancement autorisé sur les nœuds control plane

**Sécurité des pods :**
- Contrôleur d'admission activé
- Par défaut : application `baseline`
- Audit/Warn : politique `restricted`
- Namespace kube-system exempté

#### 3. Stockage : local-path-provisioner

**StorageClass** : `local-path`

**Avantages :**
- Simple, pas de dépendances externes
- Bonnes performances pour un usage homelab
- Provisionnement automatique des PV

**Configuration :**
- Chemin de stockage : `/var/lib/local-path-provisioner`
- Monté avec options `bind`, `rshared`, `rw`
- Fonctionne sur les nœuds control plane et worker

**Volumes :**
- Immich : 500Gi (bibliothèque) + 10Gi (cache) + 20Gi (postgres)
- Paperless : 10Gi (data) + 50Gi (media) + 10Gi (consume) + 10Gi (export) + postgres
- Wallabag : 10Gi (data) + 20Gi (images) + postgres
- Miniflux : 10Gi (postgres)
- PostgreSQL partagé : 20Gi
- Redis partagé : 2Gi

#### 4. Gestion des secrets : External Secrets Operator + Bitwarden

**Architecture :**
```
Bitwarden Secrets Manager
         ↓
   (API HTTPS)
         ↓
Serveur SDK Bitwarden (dans le cluster)
   Port 9998, certificat auto-signé
         ↓
External Secrets Operator
         ↓
Secrets Kubernetes (synchro auto toutes les 1h)
         ↓
Pods d'application
```

**Avantages :**
- Stockage centralisé des secrets hors du cluster
- Rotation automatique des secrets
- Pas de secrets dans Git
- Mises à jour faciles (changement dans Bitwarden → synchro auto)

**Composants :**
- **ClusterSecretStore** : Configuration Bitwarden au niveau du cluster
- **ExternalSecret** : Définitions de secrets par application
- **Serveur SDK Bitwarden** : Proxy HTTPS avec certificat auto-signé

#### 5. Ingress : Tailscale

**Pourquoi Tailscale ?**
- Accès réseau zero-trust
- Pas besoin de port forwarding
- HTTPS automatique avec certificats Tailscale
- Accès depuis n'importe où de manière sécurisée

**Configuration :**
- Opérateur Tailscale installé dans le cluster
- Ressources Ingress avec `ingressClassName: tailscale`
- Chaque service obtient une URL `https://<service>.tail<id>.ts.net`
- Intégré avec la découverte de services Kubernetes

**Pas besoin de contrôleur ingress traditionnel** (Nginx, Traefik) !

#### 6. Authentification : Authelia (OIDC/OAuth2)

**Applications supportées :**
- Paperless-ngx : OAuth2
- Miniflux : OIDC

**Non supporté :**
- Wallabag : Pas de support OIDC pour l'authentification utilisateur
- Immich : Authentification locale uniquement

**Avantages :**
- Single sign-on (SSO) quand c'est possible
- Gestion centralisée des utilisateurs
- Support 2FA
- Gestion des sessions

### Architecture des applications

#### Pattern de services partagés

```
┌─────────────────────────────────────────────────────────┐
│         Namespace Services Partagés                     │
│                                                         │
│  ┌──────────────────┐      ┌──────────────────┐        │
│  │  PostgreSQL 16   │      │     Redis 7      │        │
│  │                  │      │                  │        │
│  │  Bases:          │      │  BDs:            │        │
│  │  - paperless     │      │  0: immich       │        │
│  │  - wallabag      │      │  1: paperless    │        │
│  │  - miniflux      │      │  2: wallabag     │        │
│  └──────────────────┘      └──────────────────┘        │
│           ↑                         ↑                   │
│           │                         │                   │
└───────────┼─────────────────────────┼───────────────────┘
            │                         │
            │                         │
┌───────────┼─────────────────────────┼───────────────────┐
│           │                         │                   │
│  ┌────────┴────────┐   ┌───────────┴──────────┐        │
│  │  Paperless-ngx  │   │  Wallabag            │        │
│  │  Namespace      │   │  Namespace           │        │
│  └─────────────────┘   └──────────────────────┘        │
│                                                         │
│  ┌─────────────────┐   ┌──────────────────────┐        │
│  │  Miniflux       │   │  Immich              │        │
│  │  Namespace      │   │  Namespace           │        │
│  └─────────────────┘   └──────────────────────┘        │
│                           (a son propre PostgreSQL)    │
└─────────────────────────────────────────────────────────┘
```

**Justification :**
- Réduire l'utilisation des ressources (instance PostgreSQL/Redis unique)
- Gestion centralisée des bases de données
- Sauvegardes plus faciles
- Immich a un PostgreSQL dédié (nécessite l'extension VectorChord)

#### Structure par application

Chaque app dans `apps/` suit ce pattern :

```
<nom-app>/
├── README.md              # Documentation en français
└── base/
    ├── kustomization.yaml # Agrégation ressources + namespace + labels
    ├── namespace.yaml     # Namespace avec PodSecurity privileged
    ├── pvc.yaml          # Persistent Volume Claims
    ├── external-secrets.yaml # Références secrets Bitwarden
    ├── <composant>-deployment.yaml
    ├── <composant>-service.yaml
    └── tailscale-ingress.yaml
```

**Kustomize** (pas Helm) :
- Simple, déclaratif
- Pas de complexité de templating
- Facile à comprendre et modifier
- Adapté au contrôle de version

### Flux réseau

```
Appareil utilisateur
    ↓
Réseau Tailscale (mesh chiffré)
    ↓
Ingress Tailscale (dans le cluster)
    ↓
Service Kubernetes (ClusterIP)
    ↓
Pod d'application
    ↓
PostgreSQL/Redis partagé (si applicable)
```

### Modèle de sécurité

#### Défense en profondeur

1. **Couche réseau** :
   - Réseau zero-trust Tailscale
   - Pas d'exposition publique
   - VPN mesh chiffré

2. **Couche OS** :
   - OS immuable Talos
   - Pas d'accès SSH
   - Piloté par API uniquement
   - Surface d'attaque minimale

3. **Couche Kubernetes** :
   - Pod Security Admission (baseline/restricted)
   - RBAC activé
   - Network policies (amélioration future)
   - Isolation par namespace

4. **Couche application** :
   - OAuth/OIDC quand supporté
   - Mots de passe forts (générés, stockés dans Bitwarden)
   - Mises à jour régulières

5. **Gestion des secrets** :
   - External Secrets Operator
   - Pas de secrets dans Git
   - Centralisés dans Bitwarden
   - Capacité de rotation automatique

#### Modèle de menaces

**Protégé contre :**
- ✅ Exposition sur internet public
- ✅ Attaques basées SSH (pas de SSH)
- ✅ Fuite de secrets dans Git
- ✅ Accès non autorisé au cluster
- ✅ Compromission au niveau OS (OS immuable)

**Non protégé contre :**
- ⚠️ Identifiants Tailscale compromis
- ⚠️ Contournement RBAC Kubernetes
- ⚠️ Vulnérabilités au niveau application
- ⚠️ Accès physique au matériel

### Stratégie de sauvegarde

**État actuel** : Sauvegardes manuelles

**Ajouts recommandés :**
- Velero pour les ressources Kubernetes
- Dumps de bases de données vers stockage externe
- Snapshots etcd réguliers
- Contrôle de version de tous les manifests (ce repo !)

### Monitoring & Observabilité

**État actuel** : Natif Kubernetes (kubectl)

**Améliorations futures :**
- Prometheus + Grafana
- Loki pour l'agrégation des logs
- Alertmanager pour les notifications
- Monitoring uptime

### Récupération après sinistre

**Plan de récupération :**

1. **Panne nœud Talos** :
   - Réinstaller Talos sur nouveau matériel
   - Appliquer la configuration sauvegardée depuis `private/`
   - Rejoindre le cluster

2. **Panne du cluster** :
   - Restaurer etcd depuis snapshot
   - Ou reconstruire depuis zéro (toute la config dans Git)
   - Restaurer les données PV depuis sauvegardes

3. **Perte de données** :
   - Restaurer depuis dumps de bases de données
   - Restaurer données PV depuis sauvegardes

### Scalabilité

**Capacité actuelle :**
- Nœud control plane unique (peut ordonnancer des workloads)
- Peut ajouter des nœuds worker au besoin
- Stockage limité par la taille du disque local

**Options de scaling :**
- Ajouter des nœuds worker pour le calcul
- Ajouter du stockage distribué (Rook/Ceph, Longhorn)
- Implémenter HA multi-control-plane
- Load balancer externe pour le serveur API

### Coût

**Coût total** : ~0€/mois 🎉

- Pas de frais cloud
- Pas de frais Kubernetes managé
- Coût électricité uniquement
- Utilise du matériel existant
