# Infrastructure K3s — jonathanlore.fr

Ce repo centralise toute l'infrastructure Kubernetes du VPS.
Le déploiement est automatisé via le **Jenkinsfile** à la racine.

## Structure

```
k8s/
├── Jenkinsfile                     ← pipeline CI/CD de l'infra
├── deploy.sh                       ← déploiement manuel Jenkins
├── reset.sh                        ← suppression/réinitialisation Jenkins
├── README.md
│
├── traefik/
│   ├── traefik-config.yaml         ← configuration Traefik (Let's Encrypt, entrypoints)
│   └── dashboard-ingress.yaml      ← exposition du dashboard Traefik
│
├── jenkins/
│   ├── Dockerfile                  ← image Jenkins custom (Docker + kubectl + Helm)
│   ├── configmap.yaml              ← options JVM Jenkins
│   ├── kustomization.yml
│   ├── namespace.yml
│   ├── pvc.yml
│   ├── deployment.yml
│   ├── service.yml
│   └── ingress.yml
│
├── infra/
│   └── namespace-infra.yaml        ← namespace `infra` (CronJobs, backups)
│
├── matomo/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── pvc-db.yaml                 ← stockage MariaDB (5Gi)
│   ├── pvc-matomo.yaml             ← stockage Matomo /var/www/html (5Gi)
│   ├── deployment-db.yaml          ← MariaDB 10.11
│   ├── service-db.yaml
│   ├── deployment-matomo.yaml      ← Matomo 5
│   ├── service-matomo.yaml
│   └── ingress.yaml                ← matomo.jonathanlore.fr
│
├── cronjobs/
│   ├── kustomization.yaml
│   ├── cronjob--backup-git.yaml    ← bare clone des dépôts GitHub → Restic
│   ├── cronjob--backup-infra.yaml  ← manifestes K3s → Restic
│   ├── cronjob--backup-jenkins.yaml← PV Jenkins → Restic
│   └── BACKUPS_README.md           ← documentation complète des backups et restaurations
│
└── monitoring/
    ├── kustomization.yaml
    ├── namespace.yaml
    ├── helm-values.yaml            ← config kube-prometheus-stack (sans secrets)
    ├── ingress-grafana.yaml
    ├── GRAFANA_DASHBOARDS.md       ← référence des dashboards et quand les utiliser
    └── README.md                   ← documentation détaillée de la stack monitoring
```

## Services exposés

| Service | URL | Namespace |
|---|---|---|
| Jenkins | https://jenkins.jonathanlore.fr | `jenkins` |
| Grafana | https://grafana.jonathanlore.fr | `monitoring` |
| Traefik | https://traefik.jonathanlore.fr | `kube-system` |
| Matomo | https://matomo.jonathanlore.fr | `matomo` |

## Déploiement automatisé

Tout push sur `main` déclenche le pipeline Jenkins qui :

1. **Deploy Traefik** — config Let's Encrypt + dashboard ingress
2. **Build Jenkins Image** — build `joanisky/jenkins-with-docker:latest` et push sur Docker Hub
3. **Deploy Jenkins** — via `kubectl apply -k jenkins/`
4. **Deploy Monitoring Stack** — via Helm (`kube-prometheus-stack`) + manifests
5. **Deploy Matomo** — Matomo + MariaDB via `kubectl apply -k matomo/`
6. **Deploy Backups** — namespace `infra` + CronJobs Restic
7. **Verify** — état des pods et des ingress
8. **Health Checks** — attend que Grafana et Jenkins répondent (timeout 3 min)
9. **Notification Discord** — succès ou échec envoyé sur Discord

### Credentials Jenkins requis

| ID | Type | Usage |
|---|---|---|
| `jenkins-dockerhub` | Username/Password | Push image Docker Hub |
| `kubeconfig` | File | Accès au cluster K3s |
| `GRAFANA_ADMIN_USER` | Secret text | Login Grafana |
| `GRAFANA_ADMIN_PASSWORD` | Secret text | Mot de passe Grafana |
| `discord-webhook-url` | Secret text | Notifications Discord |

## Matomo — premier déploiement

Le secret de base de données doit être créé **manuellement une seule fois** avant le premier déploiement (jamais versionné) :

```bash
kubectl create namespace matomo --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic matomo-db-secret \
  --from-literal=MARIADB_ROOT_PASSWORD=<mot_de_passe_root> \
  --from-literal=MARIADB_DATABASE=matomo \
  --from-literal=MARIADB_USER=matomo \
  --from-literal=MARIADB_PASSWORD=<mot_de_passe_user> \
  --namespace matomo
```

Après déploiement, ouvrir https://matomo.jonathanlore.fr pour lancer l'installeur web :
- **Hôte DB** : `matomo-db`
- **Port** : `3306`
- **Nom BDD** / **Utilisateur** / **Mot de passe** : les valeurs du secret ci-dessus

La configuration est persistée dans le PVC `matomo-data` — l'installeur ne se relance pas après un redémarrage de pod.

## Sauvegardes automatisées

Les backups tournent dans le namespace `infra` via des CronJobs Kubernetes.
Tous les snapshots sont envoyés sur Google Drive via **Restic + rclone**.

| CronJob | Heure | Ce qui est sauvegardé |
|---|---|---|
| `restic-backup-git` | 05h00 | Bare clones des dépôts GitHub |
| `restic-backup-infra` | 03h00 | Manifestes K3s (`~/k3s/` sur le VPS) |
| `restic-backup-jenkins` | 04h00 | PersistentVolume Jenkins |

Voir `cronjobs/BACKUPS_README.md` pour les procédures de restauration complètes.

### Secrets Kubernetes requis dans le namespace `infra`

Ces secrets doivent être créés manuellement (jamais versionnés) :

| Secret | Clé | Contenu |
|---|---|---|
| `restic-secret` | `RESTIC_PASSWORD` | Mot de passe du dépôt Restic |
| `rclone-config` | `rclone.conf` | Configuration rclone (token Google Drive) |
| `github-token` | `GITHUB_TOKEN` | Personal Access Token GitHub (read-only) |
| `github-token` | `GITHUB_USER` | Nom d'utilisateur GitHub |

## Déploiement manuel (urgence)

Si Jenkins est indisponible :

```bash
# Traefik
kubectl apply -f traefik/traefik-config.yaml
kubectl apply -f traefik/dashboard-ingress.yaml

# Jenkins
kubectl apply -k jenkins/

# Monitoring
kubectl apply -k monitoring/
helm upgrade --install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values monitoring/helm-values.yaml \
  --set grafana.adminPassword='<MOT_DE_PASSE>' \
  --wait --timeout 5m

# Matomo
kubectl apply -k matomo/

# Backups
kubectl apply -f infra/namespace-infra.yaml
kubectl apply -k cronjobs/
```

## Ajouter le monitoring d'un nouveau projet

Les `PodMonitor` sont centralisés dans ce repo (dans `monitoring/`).
Voir `monitoring/README.md` pour la procédure complète.

## Troubleshooting

**Pod Jenkins ne démarre pas :**
```bash
kubectl describe pod -n jenkins -l app=jenkins
kubectl logs -n jenkins -l app=jenkins
```

**Certificat TLS ne se génère pas :**
```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik | grep -i "acme\|certificate"
```

**Ingress ne fonctionne pas :**
```bash
kubectl describe ingress <nom> -n <namespace>
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik | grep <nom>
```

**Grafana inaccessible :**
```bash
kubectl get pods -n monitoring
kubectl rollout status deployment/kube-prometheus-stack-grafana -n monitoring
```

**CronJob backup en échec :**
```bash
kubectl get cronjobs -n infra
kubectl get jobs -n infra
kubectl logs -n infra -l job-name=<nom-du-job>
```

**Reset complet Jenkins :**
```bash
./reset.sh
kubectl apply -k jenkins/
```
