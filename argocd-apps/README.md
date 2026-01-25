# ArgoCD Applications - Homelab Project

Ce répertoire contient les définitions des Applications ArgoCD pour gérer toutes les applications du homelab en mode GitOps, organisées dans le projet **homelab**.

## Structure

- `project-homelab.yaml` - Définition du projet ArgoCD "homelab"
- `apps.yaml` - App of Apps qui déploie automatiquement toutes les autres applications
- `*.yaml` - Définitions individuelles pour chaque application

## Projet ArgoCD "homelab"

Toutes les applications sont organisées dans un projet ArgoCD dédié avec:
- **Repository autorisé**: `https://github.com/CedricLphn/homelab.git`
- **Destinations**: Tous les namespaces du cluster local
- **Permissions**: Groupe `argocd-admins` a un accès admin complet

## Déploiement

### Étape 1: Commiter et pousser vers GitHub

```bash
git add argocd-apps/ apps/argocd/
git commit -m "feat: add ArgoCD applications with homelab project"
git push
```

### Étape 2: Créer le projet homelab

```bash
kubectl apply -f argocd-apps/project-homelab.yaml
```

### Étape 3: Déployer toutes les applications (App of Apps)

```bash
kubectl apply -f argocd-apps/apps.yaml
```

Cette commande déploie l'Application "apps" qui va automatiquement créer et synchroniser toutes les autres applications dans le projet homelab.

### Alternative: Déployer une application spécifique

```bash
kubectl apply -f argocd-apps/shared-services.yaml
kubectl apply -f argocd-apps/immich.yaml
# etc.
```

## Applications disponibles

| Application | Namespace | Description |
|------------|-----------|-------------|
| shared-services | shared-services | PostgreSQL 16 + Redis 7 partagés |
| immich | immich | Gestion photos/vidéos |
| paperless-ngx | paperless-ngx | Gestion documentaire |
| wallabag | wallabag | Read-it-later |
| miniflux | miniflux | Lecteur RSS |
| stirling-pdf | stirling-pdf | Manipulation PDF |
| radicale | radicale | CalDAV/CardDAV |
| rsshub | rsshub | Générateur de flux RSS |

## Configuration

Chaque Application ArgoCD est configurée avec:
- **project**: `homelab` - Toutes les apps sont dans le projet homelab
- **syncPolicy.automated.prune**: Supprime les ressources qui ne sont plus dans Git
- **syncPolicy.automated.selfHeal**: Resynchronise automatiquement si des modifications manuelles sont détectées
- **syncOptions.CreateNamespace**: Crée automatiquement le namespace si nécessaire

## Ordre de déploiement recommandé

1. **Projet homelab** (`project-homelab.yaml`)
2. **App of Apps** (`apps.yaml`)
3. ArgoCD se charge du reste automatiquement!

## Workflow GitOps

Une fois déployé, pour modifier tes applications:

1. Modifie le code dans `apps/*/base/`
2. Commit et push vers GitHub
3. ArgoCD détecte automatiquement le changement (polling toutes les 3 minutes)
4. ArgoCD synchronise automatiquement les ressources

## Surveillance

Après déploiement, surveille l'état dans l'interface ArgoCD:
- **URL**: `http://argocd.tail<your-id>.ts.net`
- **Username**: `admin`
- **Password**: `password`

Dans l'interface, tu verras:
- 📁 **Projet "homelab"** contenant toutes tes applications
- ✅ Applications synchronisées (vert) ou désynchronisées (jaune)
- ❤️ État de santé (Healthy/Progressing/Degraded)
- 🔄 Historique des synchronisations

## Commandes utiles

```bash
# Lister toutes les applications du projet homelab
kubectl get applications -n argocd -l argocd.argoproj.io/project=homelab

# Forcer une synchronisation manuelle
kubectl patch application <app-name> -n argocd -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"normal"}}}' --type=merge

# Voir les détails d'une application
kubectl describe application <app-name> -n argocd
```
