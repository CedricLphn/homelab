# Radicale

Radicale est un serveur CalDAV et CardDAV léger et open-source pour la synchronisation de calendriers et contacts.

## Version

- **Radicale**: 3.5.8.2
- **Image Docker**: `tomsquest/docker-radicale:3.5.8.2`

## Fonctionnalités

- Synchronisation CalDAV (calendriers)
- Synchronisation CardDAV (contacts)
- Interface web intégrée pour la gestion
- Authentification par htpasswd (bcrypt)
- Stockage sur filesystem
- Compatible avec tous les clients CalDAV/CardDAV (Thunderbird, DAVx5, Apple Calendar, etc.)

## Architecture

### Composants

- **Radicale**: Serveur CalDAV/CardDAV
- **Stockage**: PVC de 5Gi pour les collections
- **Authentification**: Fichier htpasswd avec encryption bcrypt
- **Accès**: Ingress Tailscale avec TLS automatique

### Prérequis

- StorageClass `local-path` configuré
- Tailscale Operator pour l'ingress

## Configuration des utilisateurs

Les utilisateurs sont définis dans un secret Bitwarden synchronisé via External Secrets Operator.

### Prérequis

Le secret `radicale-htpasswd` doit exister dans Bitwarden Secrets Manager avec le contenu du fichier htpasswd.

### Générer un hash bcrypt

```bash
# Avec Docker
docker run --rm python:3.11-alpine sh -c "pip install bcrypt -q && python -c \"import bcrypt; print('USERNAME:' + bcrypt.hashpw(b'PASSWORD', bcrypt.gensalt()).decode())\""

# Ou avec htpasswd (si disponible)
htpasswd -B -n username
```

### Format du secret Bitwarden

Le secret `radicale-htpasswd` dans Bitwarden doit contenir le fichier htpasswd complet avec un utilisateur par ligne :

```
admin:$2b$12$2aYYJxEDxfh6Be6CQjVuzeK1UIk4Ead2GW0tJCRUIXH1lw2bLi2j.
alice:$2b$12$JtBfo7X6v/mEzp/H6Wwfp.xDHcn7dT8E9Xe/e7F1aUE0WojPys.Jm
bob:$2b$12$Xx8pdrgZwqGpRMJoeLmRZekYOnkCI8/sud7OD9FG6CsNWWHeqknre
```

**Credentials par défaut** :
- Username: `admin`
- Password: `admin123`

**⚠️ Important** : Changez ces credentials avant de passer en production !

## Déploiement

### Prérequis Bitwarden

Avant de déployer, créer le secret `radicale-htpasswd` dans Bitwarden Secrets Manager avec le contenu htpasswd :

```
admin:$2b$12$2aYYJxEDxfh6Be6CQjVuzeK1UIk4Ead2GW0tJCRUIXH1lw2bLi2j.
```

### Déployer l'application

```bash
# Déployer l'application
kubectl apply -k apps/radicale/base/

# Vérifier le déploiement
kubectl get pods -n radicale
kubectl get ingress -n radicale
kubectl get externalsecret -n radicale

# Voir les logs
kubectl logs -f deployment/radicale -n radicale
```

## Accès

L'application est accessible via Tailscale à l'adresse :
```
https://radicale.tail<tailnet-id>.ts.net
```

## Configuration des clients

### DAVx5 (Android)

1. Installer DAVx5 depuis F-Droid ou Google Play
2. Ajouter un compte
3. Choisir "Se connecter avec une URL et un nom d'utilisateur"
4. URL de base : `https://radicale.tail<tailnet-id>.ts.net/`
5. Nom d'utilisateur : `admin` (ou votre utilisateur htpasswd)
6. Mot de passe : `admin123` (ou votre mot de passe)
7. Sélectionner les calendriers et carnets d'adresses à synchroniser

### Thunderbird

1. Installer Thunderbird
2. Pour les calendriers : Fichier → Nouveau → Calendrier → Sur le réseau
3. Format : CalDAV
4. URL : `https://radicale.tail<tailnet-id>.ts.net/username/calendar.ics/`
5. Pour les contacts : Carnet d'adresses → Nouveau → Carnet d'adresses CardDAV
6. URL : `https://radicale.tail<tailnet-id>.ts.net/username/contacts.vcf/`

### Apple Calendar / Contacts (macOS/iOS)

**Calendrier:**
1. Réglages → Calendrier → Comptes → Ajouter un compte
2. Choisir "Autre" → "Ajouter un compte CalDAV"
3. Serveur : `radicale.tail<tailnet-id>.ts.net`
4. Nom d'utilisateur et mot de passe
5. Activer SSL

**Contacts:**
1. Réglages → Contacts → Comptes → Ajouter un compte
2. Choisir "Autre" → "Ajouter un compte CardDAV"
3. Serveur : `radicale.tail<tailnet-id>.ts.net`
4. Nom d'utilisateur et mot de passe
5. Activer SSL

## Structure des collections

Les collections sont stockées dans `/data/collections` suivant cette structure :
```
/data/collections/
└── username/
    ├── calendar.ics/      # Calendrier par défaut
    ├── contacts.vcf/      # Contacts par défaut
    └── ...                # Autres collections
```

Chaque utilisateur a son propre dossier avec ses collections CalDAV et CardDAV.

## Gestion des utilisateurs

Les utilisateurs sont définis dans le secret Bitwarden `radicale-htpasswd`.

### Ajouter un utilisateur

1. Générer un nouveau hash bcrypt :
   ```bash
   docker run --rm python:3.11-alpine sh -c "pip install bcrypt -q && python -c \"import bcrypt; print('nouvel-utilisateur:' + bcrypt.hashpw(b'motdepasse123', bcrypt.gensalt()).decode())\""
   ```

2. Modifier le secret `radicale-htpasswd` dans Bitwarden Secrets Manager :
   - Ajouter la nouvelle ligne générée (username:hash)

3. Le secret sera automatiquement synchronisé dans Kubernetes (refresh: 1h max)

4. Optionnel - forcer la synchronisation immédiate :
   ```bash
   kubectl annotate externalsecret radicale-users -n radicale \
     force-sync=$(date +%s) --overwrite
   ```

5. Redémarrer Radicale :
   ```bash
   kubectl rollout restart deployment/radicale -n radicale
   ```

### Supprimer un utilisateur

1. Modifier le secret `radicale-htpasswd` dans Bitwarden
2. Retirer la ligne de l'utilisateur
3. Attendre la synchronisation (1h max) ou forcer avec l'annotation ci-dessus
4. Redémarrer Radicale
5. Optionnel : supprimer les données :
   ```bash
   kubectl exec -n radicale deployment/radicale -- rm -rf /data/collections/ancien-utilisateur
   ```

## Maintenance

### Voir les logs

```bash
kubectl logs -f deployment/radicale -n radicale
```

### Redémarrer le serveur

```bash
kubectl rollout restart deployment/radicale -n radicale
```

### Vérifier l'état de santé

```bash
kubectl exec -n radicale deployment/radicale -- \
  curl -s -o /dev/null -w "%{http_code}" http://localhost:5232/ --max-time 5
```

### Sauvegarder les données

```bash
# Créer une archive des collections
kubectl exec -n radicale deployment/radicale -- tar czf /tmp/backup.tar.gz /data/collections

# Copier l'archive localement
kubectl cp radicale/radicale-<pod-id>:/tmp/backup.tar.gz ./radicale-backup-$(date +%Y%m%d).tar.gz
```

### Restaurer les données

```bash
# Copier l'archive vers le pod
kubectl cp ./radicale-backup.tar.gz radicale/radicale-<pod-id>:/tmp/backup.tar.gz

# Extraire l'archive
kubectl exec -n radicale deployment/radicale -- tar xzf /tmp/backup.tar.gz -C /
```

## Dépannage

### Le pod ne démarre pas

Vérifier les logs :
```bash
kubectl logs -n radicale deployment/radicale
```

### Erreur d'authentification (401 Unauthorized)

1. **Vérifier les credentials htpasswd** :
   ```bash
   kubectl get secret radicale-users -n radicale -o jsonpath='{.data.users}' | base64 -d
   ```

2. **Tester l'authentification** :
   ```bash
   curl -v https://radicale.tail<tailnet-id>.ts.net/ -u admin:admin123
   ```

3. **Vérifier les logs Radicale** :
   ```bash
   kubectl logs -n radicale deployment/radicale --tail=50
   ```

### Les clients ne peuvent pas se connecter

1. **Vérifier l'ingress Tailscale** :
   ```bash
   kubectl get ingress -n radicale
   ```

2. **Tester l'accès depuis un navigateur** :
   - Ouvrir `https://radicale.tail<tailnet-id>.ts.net/`
   - Vous devriez voir une popup d'authentification HTTP Basic
   - Entrer les credentials htpasswd (admin/admin123 par défaut)
   - L'interface web Radicale devrait s'afficher

### Le secret Bitwarden ne se synchronise pas

Vérifier le statut de l'ExternalSecret :
```bash
kubectl get externalsecret radicale-users -n radicale
kubectl describe externalsecret radicale-users -n radicale
```

Si le statut indique une erreur :
1. Vérifier que le secret `radicale-htpasswd` existe dans Bitwarden
2. Vérifier les logs d'External Secrets Operator :
   ```bash
   kubectl logs -n external-secrets-system deployment/external-secrets
   ```
3. Forcer une resynchronisation :
   ```bash
   kubectl annotate externalsecret radicale-users -n radicale \
     force-sync=$(date +%s) --overwrite
   ```

### Problèmes de permissions

Si des erreurs de permissions apparaissent dans les logs Radicale :
```bash
kubectl exec -n radicale deployment/radicale -- chown -R 2999:2999 /data
```

## Ressources

- Site officiel : https://radicale.org
- Documentation : https://radicale.org/v3.html
- GitHub : https://github.com/Kozea/Radicale
- Image Docker : https://hub.docker.com/r/tomsquest/docker-radicale

## Notes

- **Authentification** : Gérée par htpasswd avec encryption bcrypt
- **Création automatique des espaces** : Chaque utilisateur authentifié obtient automatiquement son propre espace
- **Isolation** : Radicale (via `radicale.rights.owner_only`) garantit que chaque utilisateur accède uniquement à ses propres collections
- **Interface web** : Accessible à la racine `/` pour consulter et gérer les collections
- **URLs des collections** : Format `/<username>/<collection>/` (ex: `/admin/calendar.ics/`)
- **HTTPS** : Géré automatiquement par Tailscale Ingress
- **Sécurité** : Authentification HTTP Basic avec hashes bcrypt stockés dans un Secret Kubernetes
