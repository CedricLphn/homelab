# RSSHub

Générateur de flux RSS open-source pour sites web sans flux natifs. RSSHub permet de suivre n'importe quel contenu web dans votre lecteur RSS préféré grâce à plus de 1000 routes prédéfinies.

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                   Tailscale Mesh                         │
│               rsshub.tail<id>.ts.net                     │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│                 rsshub namespace                         │
│  ┌────────────────────────────────────────────────────┐  │
│  │             RSSHub (port 1200)                     │  │
│  │            diygod/rsshub:latest                    │  │
│  │        Node.js + 1000+ routes RSS                  │  │
│  └──────────────────┬─────────────────────────────────┘  │
└─────────────────────┼────────────────────────────────────┘
                      │
┌─────────────────────▼────────────────────────────────────┐
│            shared-services namespace                     │
│  ┌────────────────────────────────────────────────────┐  │
│  │              Redis (DB 3)                          │  │
│  │         Cache des routes + contenu                 │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

## Fonctionnalités principales

- **1000+ routes prédéfinies** : Twitter, YouTube, GitHub, Reddit, Medium, et bien plus
- **Scraping intelligent** : Extraction automatique de contenu web
- **Cache Redis** : Performance optimisée avec mise en cache (1h TTL)
- **Support multilingue** : Routes pour sites du monde entier
- **Format RSS standard** : Compatible avec tous les lecteurs RSS
- **Pas d'authentification** : Accès simplifié protégé par Tailscale VPN

## Prérequis

- Kubernetes cluster avec Talos Linux
- Tailscale Operator installé
- Redis partagé dans namespace `shared-services`

## Déploiement

```bash
kubectl apply -k apps/rsshub/base/
```

## Vérification

```bash
# Vérifier le pod
kubectl get pods -n rsshub

# Consulter les logs
kubectl logs -f deployment/rsshub -n rsshub

# Vérifier l'ingress Tailscale
kubectl get ingress -n rsshub
```

Le démarrage prend environ 30-60 secondes. Attendez que le pod soit en état `Running` et `READY 1/1`.

## Accès

L'application est accessible via Tailscale :
- URL : `https://rsshub.tail<your-tailnet-id>.ts.net`

Pour obtenir l'URL exacte :
```bash
kubectl get ingress rsshub-ingress -n rsshub -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

## Configuration

### Variables d'environnement

| Variable | Valeur | Description |
|----------|--------|-------------|
| `NODE_ENV` | `production` | Mode production Node.js |
| `CACHE_TYPE` | `redis` | Utilise Redis pour le cache |
| `REDIS_URL` | `redis://redis.shared-services.svc.cluster.local:6379/3` | Connexion Redis DB 3 |
| `CACHE_EXPIRE` | `3600` | Expiration cache routes (1h) |
| `CACHE_CONTENT_EXPIRE` | `3600` | Expiration cache contenu (1h) |
| `TZ` | `Europe/Paris` | Fuseau horaire |

### Cache Redis

RSSHub utilise la base de données Redis n°3 (DB 3) du service partagé :
- DB 0 : Immich
- DB 1 : Paperless-ngx
- DB 2 : Wallabag
- **DB 3 : RSSHub** (nouvelle allocation)

Le cache stocke les routes et le contenu pendant 1 heure pour réduire la charge sur les sources externes.

## Utilisation

### Exemples de routes populaires

Remplacez `<rsshub-url>` par votre URL Tailscale complète.

**GitHub - Commits d'un repository :**
```
https://<rsshub-url>/github/commits/DIYgod/RSSHub
```

**YouTube - Chaîne :**
```
https://<rsshub-url>/youtube/user/@mkbhd
```

**Reddit - Subreddit :**
```
https://<rsshub-url>/reddit/r/selfhosted
```

**Medium - Publication :**
```
https://<rsshub-url>/medium/@username
```

**Twitter - Utilisateur (nécessite chromium-bundled) :**
```
https://<rsshub-url>/twitter/user/username
```

### Documentation des routes

Liste complète des routes disponibles : https://docs.rsshub.app/routes/

### Ajouter un flux à votre lecteur RSS

1. Trouvez la route RSSHub correspondante au site souhaité
2. Copiez l'URL complète (ex: `https://rsshub.tail<id>.ts.net/github/commits/user/repo`)
3. Ajoutez cette URL dans votre lecteur RSS (Miniflux, Feedly, etc.)

## Ressources

### Demandes vs Ressources allouées

- CPU : 100m (request) / 500m (limit)
- Mémoire : 256Mi (request) / 512Mi (limit)

Ces valeurs sont suffisantes pour un usage personnel avec l'image standard. Si vous utilisez chromium-bundled, augmentez à 512Mi-1Gi.

## Dépannage

### Le pod ne démarre pas

Vérifier les logs pour identifier l'erreur :
```bash
kubectl logs -f deployment/rsshub -n rsshub
```

Vérifier que Redis est accessible :
```bash
kubectl exec -n rsshub deployment/rsshub -- sh -c 'nc -zv redis.shared-services.svc.cluster.local 6379'
```

### Une route retourne une erreur 500

Certaines routes nécessitent l'image `chromium-bundled` pour le rendu JavaScript (réseaux sociaux, sites dynamiques). Voir section "Upgrade vers chromium-bundled" ci-dessous.

### Le cache ne fonctionne pas

Vérifier que Redis DB 3 contient des clés RSSHub :
```bash
kubectl exec -n shared-services deployment/redis -- redis-cli -n 3 KEYS '*'
```

Après avoir accédé à une route, des clés doivent apparaître.

### Routes trop lentes

Si les routes prennent trop de temps à charger :
1. Vérifier les logs RSSHub pour identifier les timeouts
2. Ajuster `CACHE_EXPIRE` à une valeur plus élevée (ex: 7200 pour 2h)
3. Réduire `REQUEST_TIMEOUT` si les sources sont souvent inaccessibles

### Tailscale ingress ne fonctionne pas

Si l'URL Tailscale ne répond pas :
```bash
# Vérifier le statut de l'ingress
kubectl describe ingress rsshub-ingress -n rsshub

# Vérifier les logs Tailscale operator
kubectl logs -n tailscale deployment/operator --tail=50

# Vérifier que le device apparaît dans la console Tailscale
# https://login.tailscale.com/admin/machines
```

## Maintenance

### Redémarrer l'application

```bash
kubectl rollout restart deployment/rsshub -n rsshub
```

Cela force le téléchargement de la dernière image `latest` et redémarre le pod.

### Vider le cache Redis

Si vous souhaitez forcer le rafraîchissement de toutes les routes :
```bash
kubectl exec -n shared-services deployment/redis -- redis-cli -n 3 FLUSHDB
```

### Surveiller l'utilisation des ressources

```bash
# Resources du pod RSSHub
kubectl top pod -n rsshub

# Resources globales du node
kubectl top node
```

### Mettre à jour RSSHub

RSSHub utilise l'image `latest` avec `imagePullPolicy: Always`. Pour forcer la mise à jour :
```bash
kubectl rollout restart deployment/rsshub -n rsshub
```

Si vous préférez épingler une version spécifique :
```yaml
# Dans rsshub-deployment.yaml
image: diygod/rsshub:2024-12-01
imagePullPolicy: IfNotPresent
```

## Upgrade vers chromium-bundled

Certaines routes nécessitent le rendu JavaScript (Twitter, Instagram, sites SPA). Si vous rencontrez des erreurs sur ces routes, passez à l'image chromium-bundled.

### Étapes

1. **Modifier `rsshub-deployment.yaml` :**

```yaml
# Ligne ~27
image: diygod/rsshub:chromium-bundled
imagePullPolicy: Always

# Lignes resources (~67-72)
resources:
  requests:
    cpu: 200m
    memory: 512Mi
  limits:
    cpu: 500m
    memory: 1Gi
```

2. **Appliquer les changements :**

```bash
kubectl apply -k apps/rsshub/base/
```

3. **Surveiller le redémarrage :**

```bash
kubectl logs -f deployment/rsshub -n rsshub
```

Le démarrage sera plus lent (~30-60s) car l'image est plus volumineuse (~1.5GB).

### Différences

| Aspect | Standard | Chromium-bundled |
|--------|----------|------------------|
| Taille image | ~500MB | ~1.5GB |
| Mémoire | 256-512Mi | 512Mi-1Gi |
| Routes supportées | ~80% (HTTP uniquement) | 100% (avec JS) |
| Démarrage | ~15-30s | ~30-60s |

## Améliorations futures (optionnelles)

### Authentification ACCESS_KEY

Pour ajouter une protection par clé d'accès aux routes :

1. Créer un secret dans Bitwarden nommé `rsshub-access-key`
2. Créer `apps/rsshub/base/external-secrets.yaml`
3. Ajouter la variable d'environnement `ACCESS_KEY` dans le deployment
4. Mettre à jour `kustomization.yaml` pour inclure `external-secrets.yaml`

Les URLs deviennent : `https://rsshub.tail<id>.ts.net/route?key=<ACCESS_KEY>`

### Cache persistant local

Pour cacher du contenu média localement :

1. Créer un PVC de 1Gi dans `apps/rsshub/base/pvc.yaml`
2. Monter le volume dans le deployment
3. Augmenter `CACHE_CONTENT_EXPIRE` pour garder le contenu plus longtemps

### Routes personnalisées

RSSHub permet d'ajouter des routes custom :

1. Créer un ConfigMap avec vos modules JavaScript
2. Monter le ConfigMap dans `/app/lib/routes-custom/`
3. Redémarrer le pod

Documentation : https://docs.rsshub.app/joinus/advanced/script-standard

## Ressources externes

- **Documentation officielle** : https://docs.rsshub.app/
- **Routes disponibles** : https://docs.rsshub.app/routes/
- **Repository GitHub** : https://github.com/DIYgod/RSSHub
- **Docker Hub** : https://hub.docker.com/r/diygod/rsshub
