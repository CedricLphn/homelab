# ArgoCD Applications

Ce répertoire contient les définitions des Applications ArgoCD pour gérer toutes les applications du homelab en mode GitOps.

## Structure

- `apps.yaml` - App of Apps qui déploie automatiquement toutes les autres applications
- `*.yaml` - Définitions individuelles pour chaque application

## Déploiement

### Option 1: Déployer tout d'un coup (App of Apps)

```bash
kubectl apply -f argocd-apps/apps.yaml
```

Cette commande déploie l'Application "apps" qui va automatiquement créer et synchroniser toutes les autres applications.

### Option 2: Déployer une application spécifique

```bash
kubectl apply -f argocd-apps/immich.yaml
```

## Configuration

Chaque Application ArgoCD est configurée avec:
- **syncPolicy.automated.prune**: Supprime les ressources qui ne sont plus dans Git
- **syncPolicy.automated.selfHeal**: Resynchronise automatiquement si des modifications manuelles sont détectées
- **syncOptions.CreateNamespace**: Crée automatiquement le namespace si nécessaire

## Ordre de déploiement recommandé

1. `shared-services` - Base de données et Redis partagés
2. Toutes les autres apps (peuvent être déployées en parallèle)

## Surveillance

Après déploiement, surveille l'état dans l'interface ArgoCD:
- URL: http://argocd.tail<your-id>.ts.net
- Username: admin
- Password: password
