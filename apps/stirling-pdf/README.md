# Stirling-PDF

Application web auto-hébergée permettant d'effectuer diverses opérations sur des fichiers PDF : fusion, division, conversion, OCR, compression, signature, et bien plus encore.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Tailscale Mesh                       │
│              stirling-pdf.tail<id>.ts.net               │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│                 stirling-pdf namespace                  │
│  ┌───────────────────────────────────────────────────┐  │
│  │              Stirling-PDF (port 8080)             │  │
│  │         docker.stirlingpdf.com/stirling-pdf       │  │
│  └───────────────────────────────────────────────────┘  │
│                          │                              │
│  ┌───────────────────────▼───────────────────────────┐  │
│  │                    PVC (2Gi)                      │  │
│  │    - tessdata (OCR)                               │  │
│  │    - configs                                      │  │
│  │    - customFiles                                  │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## Fonctionnalités principales

- **Fusion/Division** : Combiner ou séparer des PDFs
- **Conversion** : PDF vers/depuis Word, Excel, images, HTML
- **OCR** : Reconnaissance de texte dans les images/scans
- **Compression** : Réduire la taille des fichiers
- **Sécurité** : Ajouter/supprimer mots de passe, permissions
- **Signature** : Signer numériquement des documents
- **Métadonnées** : Éditer les informations du document
- **Filigrane** : Ajouter texte ou images en filigrane

## Prérequis

- Kubernetes cluster avec Talos Linux
- StorageClass `local-path` configurée
- Tailscale Operator installé

## Déploiement

```bash
kubectl apply -k apps/stirling-pdf/base/
```

## Vérification

```bash
kubectl get pods -n stirling-pdf
kubectl get svc -n stirling-pdf
kubectl logs -f deployment/stirling-pdf -n stirling-pdf
```

## Accès

L'application est accessible via Tailscale :
- URL : `https://stirling-pdf.tail<your-tailnet-id>.ts.net`

## Configuration

### Variables d'environnement

| Variable | Valeur | Description |
|----------|--------|-------------|
| `DOCKER_ENABLE_SECURITY` | `false` | Désactive l'authentification intégrée |
| `INSTALL_BOOK_AND_ADVANCED_HTML_OPS` | `false` | N'installe pas Calibre (économise des ressources) |
| `LANGS` | `fr_FR` | Langue de l'interface |

### Stockage

Le PVC de 2Gi stocke :
- `tessdata/` : Données OCR Tesseract
- `configs/` : Configuration de l'application
- `customFiles/` : Fichiers personnalisés (logos, etc.)

## Ressources

- CPU : 100m (request) / 1000m (limit)
- Mémoire : 256Mi (request) / 1Gi (limit)

## Dépannage

### L'OCR ne fonctionne pas

Vérifier que les données de langue sont téléchargées :
```bash
kubectl exec -n stirling-pdf deployment/stirling-pdf -- ls -la /usr/share/tessdata/
```

### Application lente

Stirling-PDF peut être gourmand en ressources lors de conversions complexes. Augmenter les limits si nécessaire.

### Erreur de permission sur le stockage

Vérifier que le PVC est bien monté et que les permissions sont correctes (fsGroup: 1000).
