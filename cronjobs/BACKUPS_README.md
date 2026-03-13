# Sauvegardes automatisées

## Stack technique

| Outil                  | Rôle                              |
|------------------------|-----------------------------------|
| **Restic**             | Création et gestion des snapshots |
| **rclone**             | Transport vers Google Drive       |
| **Kubernetes CronJob** | Planification automatique         |

---

## Architecture

Tous les CronJobs tournent dans le namespace `infra` et sauvegardent vers le même dépôt Restic : `rclone:gdrive:restic-thetiptop`.
```
Google Drive/
└── restic-thetiptop/       ← dépôt Restic unique
      ├── snapshots mariadb
      ├── snapshots infra
      ├── snapshots jenkins
      └── snapshots git
```

---

## CronJobs

| Nom                     | Heure | Ce qui est sauvegardé                                 | Namespace source  |
|-------------------------|-------|-------------------------------------------------------|-------------------|
| `restic-backup-mariadb` | 02h00 | Base de données MariaDB (dump SQL)                    | `the-tip-top-api` |
| `restic-backup-infra`   | 03h00 | Manifestes K3s (`~/k3s/` sur le VPS)                  | VPS               |
| `restic-backup-jenkins` | 04h00 | PersistentVolume Jenkins (jobs, credentials, plugins) | `jenkins`         |
| `restic-backup-git`     | 05h00 | Clone bare des dépôts GitHub                          | GitHub            |

---

## Politique de rétention

Appliquée automatiquement après chaque backup :

| Période      | Snapshots conservés |
|--------------|---------------------|
| Quotidien    | 7 jours             |
| Hebdomadaire | 4 semaines          |
| Mensuel      | 2 mois              |

---

## Secrets Kubernetes

Les secrets suivants doivent exister dans le namespace `infra` (créés manuellement, jamais versionnés) :

| Secret          | Clé               | Contenu                                   |
|-----------------|-------------------|-------------------------------------------|
| `restic-secret` | `RESTIC_PASSWORD` | Mot de passe du dépôt Restic              |
| `rclone-config` | `rclone.conf`     | Configuration rclone (token Google Drive) |
| `github-token`  | `GITHUB_TOKEN`    | Personal Access Token GitHub (read-only)  |
| `github-token`  | `GITHUB_USER`     | Nom d'utilisateur GitHub                  |

> ⚠️ Ces secrets ne sont pas dans le repo Git. En cas de recréation du cluster, les recréer manuellement avant de déployer les CronJobs.

---

## Dépôts GitHub sauvegardés

- `the-tip-top-front`
- `the-tip-top-api`
- `k8s`