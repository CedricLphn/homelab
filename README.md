# 🏠 Homelab Infrastructure

> Self-hosted applications running on Talos Linux and Kubernetes

[English](#english) | [Français](#français)

---

## English

### 📋 Overview

This repository contains the complete infrastructure-as-code for my personal homelab, running on a bare-metal Kubernetes cluster powered by Talos Linux.

**Key Technologies:**
- **OS**: [Talos Linux](https://www.talos.dev/) v1.11.1 (immutable, secure, minimal)
- **Kubernetes**: v1.34.0
- **Deployment**: Kustomize (GitOps-ready)
- **Storage**: local-path-provisioner
- **Secrets**: Bitwarden Secrets Manager via External Secrets Operator
- **Ingress**: Tailscale (secure, private networking)
- **Authentication**: Authelia (OIDC/OAuth2)

### 🚀 Deployed Applications

| Application | Description | Authentication | Storage |
|-------------|-------------|----------------|---------|
| [Immich](https://immich.app/) | Photo and video management (Google Photos alternative) | Local + Tailscale | 500Gi library + PostgreSQL |
| [Paperless-ngx](https://docs.paperless-ngx.com/) | Document management system | OAuth (Authelia) | 50Gi media + PostgreSQL |
| [Wallabag](https://wallabag.org/) | Read-it-later service (Pocket alternative) | Local | 20Gi images + PostgreSQL |
| [Miniflux](https://miniflux.app/) | Minimalist RSS reader | OIDC (Authelia) + Local admin | PostgreSQL |
| **Shared Services** | PostgreSQL 16 + Redis 7 | - | 20Gi + 2Gi |

### 📁 Repository Structure

```
homelab/
├── README.md                    # This file (bilingual)
├── .gitignore                   # Protects sensitive data
│
├── docs/                        # Documentation
│   ├── architecture.md          # Infrastructure architecture
│   └── setup-guide.md           # Step-by-step setup guide
│
├── infrastructure/              # Base infrastructure
│   ├── talos/                   # Talos Linux configurations
│   │   ├── README.md
│   │   ├── controlplane.yaml.example
│   │   ├── worker.yaml.example
│   │   └── patches/             # Configuration patches
│   └── kubernetes/              # Kubernetes base configs
│       └── external-secrets/    # Secret management setup
│
├── apps/                        # Application deployments
│   ├── immich/
│   ├── paperless-ngx/
│   ├── wallabag/
│   ├── miniflux/
│   └── shared-services/
│
└── private/                     # ⚠️ GITIGNORED - Sensitive configs
    ├── talos/                   # Real Talos configs (secrets, keys)
    ├── kubernetes/              # Real Bitwarden configs (org/project IDs)
    └── archive/                 # Old/backup configurations
```

### 🔒 Security & Secrets Management

**External Secrets Operator** automatically syncs secrets from Bitwarden Secrets Manager:
- All sensitive data stored in Bitwarden (not in Git)
- 1-hour refresh interval
- Centralized secret management
- See [`infrastructure/kubernetes/external-secrets/`](infrastructure/kubernetes/external-secrets/) for setup

**What's NOT in Git:**
- Talos machine tokens, CA keys, cluster secrets
- Bitwarden organization/project IDs
- Kubernetes service account keys
- Application passwords and API keys

### 🛠️ Quick Start

1. **Hardware Requirements**:
   - At least 1 control plane node (can be combined with worker)
   - Network: Private subnet (e.g., 192.168.1.0/24 or 10.0.0.0/8)
   - Storage: Local disk for persistent volumes

2. **Setup Talos**:
   ```bash
   # Generate configurations
   talosctl gen config my-cluster https://<control-plane-ip>:6443

   # Apply to nodes
   talosctl apply-config --insecure --nodes <node-ip> --file controlplane.yaml
   ```

3. **Bootstrap Kubernetes**:
   ```bash
   talosctl bootstrap --nodes <control-plane-ip>
   talosctl kubeconfig --nodes <control-plane-ip>
   ```

4. **Deploy Applications**:
   ```bash
   # Install External Secrets Operator first
   # Then deploy apps
   kubectl apply -k apps/immich/base
   ```

Refer to Talos and Kubernetes official documentation for detailed setup instructions.

### 📚 Documentation

- [Architecture Overview](docs/architecture.md) - Infrastructure design and components
- [Talos Configuration](infrastructure/talos/README.md) - Talos Linux setup
- [Secret Management](infrastructure/kubernetes/external-secrets/README.md) - ESO + Bitwarden

### 🤝 Contributing

This is a personal homelab repository, but feel free to:
- Open issues for questions
- Fork and adapt for your own homelab
- Submit PRs for documentation improvements

### 📝 License

MIT License - Feel free to use this as inspiration for your own homelab!

### 🔗 References

- [Talos Linux](https://www.talos.dev/)
- [Kubernetes](https://kubernetes.io/)
- [External Secrets Operator](https://external-secrets.io/)
- [Bitwarden Secrets Manager](https://bitwarden.com/products/secrets-manager/)
- [Authelia](https://www.authelia.com/)
- [Tailscale](https://tailscale.com/)

---

## Français

### 📋 Aperçu

Ce dépôt contient l'infrastructure complète de mon homelab personnel, fonctionnant sur un cluster Kubernetes bare-metal propulsé par Talos Linux.

**Technologies clés :**
- **OS** : [Talos Linux](https://www.talos.dev/) v1.11.1 (immuable, sécurisé, minimal)
- **Kubernetes** : v1.34.0
- **Déploiement** : Kustomize (prêt pour GitOps)
- **Stockage** : local-path-provisioner
- **Secrets** : Bitwarden Secrets Manager via External Secrets Operator
- **Ingress** : Tailscale (réseau privé et sécurisé)
- **Authentification** : Authelia (OIDC/OAuth2)

### 🚀 Applications déployées

| Application | Description | Authentification | Stockage |
|-------------|-------------|------------------|----------|
| [Immich](https://immich.app/) | Gestion de photos et vidéos (alternative à Google Photos) | Local + Tailscale | 500Gi bibliothèque + PostgreSQL |
| [Paperless-ngx](https://docs.paperless-ngx.com/) | Système de gestion documentaire | OAuth (Authelia) | 50Gi média + PostgreSQL |
| [Wallabag](https://wallabag.org/) | Service "lire plus tard" (alternative à Pocket) | Local | 20Gi images + PostgreSQL |
| [Miniflux](https://miniflux.app/) | Lecteur RSS minimaliste | OIDC (Authelia) + Admin local | PostgreSQL |
| **Services partagés** | PostgreSQL 16 + Redis 7 | - | 20Gi + 2Gi |

### 📁 Structure du dépôt

```
homelab/
├── README.md                    # Ce fichier (bilingue)
├── .gitignore                   # Protection des données sensibles
│
├── docs/                        # Documentation
│   ├── architecture.md          # Architecture de l'infrastructure
│   └── setup-guide.md           # Guide d'installation étape par étape
│
├── infrastructure/              # Infrastructure de base
│   ├── talos/                   # Configurations Talos Linux
│   │   ├── README.md
│   │   ├── controlplane.yaml.example
│   │   ├── worker.yaml.example
│   │   └── patches/             # Patches de configuration
│   └── kubernetes/              # Configurations Kubernetes de base
│       └── external-secrets/    # Configuration de la gestion des secrets
│
├── apps/                        # Déploiements d'applications
│   ├── immich/
│   ├── paperless-ngx/
│   ├── wallabag/
│   ├── miniflux/
│   └── shared-services/
│
└── private/                     # ⚠️ GITIGNORE - Configurations sensibles
    ├── talos/                   # Vraies configs Talos (secrets, clés)
    ├── kubernetes/              # Vraies configs Bitwarden (IDs org/projet)
    └── archive/                 # Anciennes configurations/sauvegardes
```

### 🔒 Sécurité & Gestion des secrets

**External Secrets Operator** synchronise automatiquement les secrets depuis Bitwarden Secrets Manager :
- Toutes les données sensibles stockées dans Bitwarden (pas dans Git)
- Intervalle de rafraîchissement : 1 heure
- Gestion centralisée des secrets
- Voir [`infrastructure/kubernetes/external-secrets/`](infrastructure/kubernetes/external-secrets/) pour la configuration

**Ce qui N'EST PAS dans Git :**
- Tokens machine Talos, clés CA, secrets de cluster
- IDs organisation/projet Bitwarden
- Clés de compte de service Kubernetes
- Mots de passe et clés API des applications

### 🛠️ Démarrage rapide

1. **Prérequis matériels** :
   - Au moins 1 nœud control plane (peut être combiné avec worker)
   - Réseau : Sous-réseau privé (ex: 192.168.1.0/24 ou 10.0.0.0/8)
   - Stockage : Disque local pour volumes persistants

2. **Configuration Talos** :
   ```bash
   # Générer les configurations
   talosctl gen config my-cluster https://<ip-control-plane>:6443

   # Appliquer aux nœuds
   talosctl apply-config --insecure --nodes <ip-noeud> --file controlplane.yaml
   ```

3. **Bootstrap Kubernetes** :
   ```bash
   talosctl bootstrap --nodes <ip-control-plane>
   talosctl kubeconfig --nodes <ip-control-plane>
   ```

4. **Déployer les applications** :
   ```bash
   # Installer External Secrets Operator d'abord
   # Puis déployer les apps
   kubectl apply -k apps/immich/base
   ```

Consultez la documentation officielle de Talos et Kubernetes pour les instructions d'installation détaillées.

### 📚 Documentation

- [Vue d'ensemble de l'architecture](docs/architecture.md) - Design et composants de l'infrastructure
- [Configuration Talos](infrastructure/talos/README.md) - Configuration de Talos Linux
- [Gestion des secrets](infrastructure/kubernetes/external-secrets/README.md) - ESO + Bitwarden

### 🤝 Contribution

Ceci est un dépôt de homelab personnel, mais n'hésitez pas à :
- Ouvrir des issues pour poser des questions
- Forker et adapter pour votre propre homelab
- Soumettre des PRs pour améliorer la documentation

### 📝 Licence

Licence MIT - N'hésitez pas à utiliser ce projet comme inspiration pour votre propre homelab !

### 🔗 Références

- [Talos Linux](https://www.talos.dev/)
- [Kubernetes](https://kubernetes.io/)
- [External Secrets Operator](https://external-secrets.io/)
- [Bitwarden Secrets Manager](https://bitwarden.com/products/secrets-manager/)
- [Authelia](https://www.authelia.com/)
- [Tailscale](https://tailscale.com/)
