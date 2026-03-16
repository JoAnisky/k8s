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


## Restauration Jenkins

### 1. Lister les snapshots disponibles

Déclencher un job one-shot pour afficher l'historique des snapshots :
```bash
kubectl create job --from=cronjob/restic-backup-jenkins list-snapshots -n infra
kubectl logs -n infra -l job-name=list-snapshots -f
```

Noter l'ID du snapshot à restaurer (ex: `1ab7877c`).

Nettoyer le job une fois terminé :
```bash
kubectl delete job list-snapshots -n infra
```

### 2. Arrêter Jenkins
```bash
kubectl scale deployment jenkins -n jenkins --replicas=0
kubectl get pods -n jenkins -w  # attendre que le pod disparaisse
```

### 3. Lancer la restauration

Remplacer `<SNAPSHOT_ID>` par l'ID noté à l'étape 1 :
```bash
kubectl run restic-restore \
  --image=alpine:latest \
  --restart=Never \
  -n infra \
  --overrides='{
    "spec": {
      "containers": [{
        "name": "restic-restore",
        "image": "alpine:latest",
        "command": ["/bin/sh", "-c", "apk add --no-cache restic rclone && mkdir -p /root/.config/rclone && cp /rclone/rclone.conf /root/.config/rclone/rclone.conf && restic -r rclone:gdrive:restic-thetiptop restore  --target / --include /jenkins && echo RESTAURATION OK"],
        "env": [
          {"name": "RESTIC_PASSWORD", "valueFrom": {"secretKeyRef": {"name": "restic-secret", "key": "RESTIC_PASSWORD"}}}
        ],
        "volumeMounts": [
          {"name": "rclone-config", "mountPath": "/rclone", "readOnly": true},
          {"name": "jenkins-data", "mountPath": "/jenkins"}
        ]
      }],
      "volumes": [
        {"name": "rclone-config", "secret": {"secretName": "rclone-config"}},
        {"name": "jenkins-data", "hostPath": {"path": "/var/lib/rancher/k3s/storage/pvc-3f3990e9-5163-4f42-b763-9f8d3590c505_jenkins_jenkins-home", "type": "Directory"}}
      ]
    }
  }'
```

Suivre les logs :
```bash
kubectl logs -n infra restic-restore -f
```

Le message `RESTAURATION OK` confirme le succès.

### 4. Relancer Jenkins
```bash
kubectl scale deployment jenkins -n jenkins --replicas=1
kubectl get pods -n jenkins -w
```

### 5. Nettoyer
```bash
kubectl delete pod restic-restore -n infra
```

## Restauration infra K3s

### 1. Lister les snapshots disponibles

```bash
kubectl create job --from=cronjob/restic-backup-infra list-snapshots-infra -n infra
kubectl logs -n infra -l job-name=list-snapshots-infra -f
```

Noter l'ID du snapshot à restaurer - choisir le plus récent pris **avant** l'incident.

Nettoyer le job une fois terminé :

```bash
kubectl delete job list-snapshots-infra -n infra
```

### 2. Lancer la restauration

Remplacer `<SNAPSHOT_ID>` par l'ID noté à l'étape 1 :

```bash
kubectl run restic-restore-infra \
  --image=alpine:latest \
  --restart=Never \
  -n infra \
  --overrides='{
    "spec": {
      "containers": [{
        "name": "restic-restore-infra",
        "image": "alpine:latest",
        "command": ["/bin/sh", "-c", "apk add --no-cache restic rclone && mkdir -p /root/.config/rclone && cp /rclone/rclone.conf /root/.config/rclone/rclone.conf && restic -r rclone:gdrive:restic-thetiptop restore <SNAPSHOT_ID> --target / --include /infra && echo RESTAURATION OK"],
        "env": [
          {"name": "RESTIC_PASSWORD", "valueFrom": {"secretKeyRef": {"name": "restic-secret", "key": "RESTIC_PASSWORD"}}}
        ],
        "volumeMounts": [
          {"name": "rclone-config", "mountPath": "/rclone", "readOnly": true},
          {"name": "infra-data", "mountPath": "/infra"}
        ]
      }],
      "volumes": [
        {"name": "rclone-config", "secret": {"secretName": "rclone-config"}},
        {"name": "infra-data", "hostPath": {"path": "/home/anisky/k3s", "type": "Directory"}}
      ]
    }
  }'
```

Suivre les logs :

```bash
kubectl logs -n infra restic-restore-infra -f
```

Le message `RESTAURATION OK` confirme le succès.

### 3. Vérifier

```bash
ls ~/k3s/
```

> ⚠️ Restic ne supprime pas les fichiers existants lors d'une restauration - il rajoute uniquement ce qui manque. Supprimer manuellement les fichiers indésirables si nécessaire.

### 4. Nettoyer

```bash
kubectl delete pod restic-restore-infra -n infra
```

## Restauration Git

Les dépôts GitHub sont sauvegardés sous forme de **bare clones** dans le repo Restic.
En cas de perte d'accès à GitHub, ils peuvent être restaurés et clonés localement.

### 1. Lister les snapshots disponibles

```bash
kubectl create job --from=cronjob/restic-backup-git list-snapshots-git -n infra
kubectl logs -n infra -l job-name=list-snapshots-git -f
```

Noter l'ID du snapshot à restaurer — choisir le plus récent pris **avant** l'incident.

Nettoyer le job une fois terminé :

```bash
kubectl delete job list-snapshots-git -n infra
```

### 2. Lancer la restauration

Remplacer `<SNAPSHOT_ID>` par l'ID noté à l'étape 1 :

```bash
kubectl run restic-restore-git \
  --image=alpine:latest \
  --restart=Never \
  -n infra \
  --overrides='{
    "spec": {
      "containers": [{
        "name": "restic-restore-git",
        "image": "alpine:latest",
        "command": ["/bin/sh", "-c", "apk add --no-cache restic rclone git && mkdir -p /root/.config/rclone && cp /rclone/rclone.conf /root/.config/rclone/rclone.conf && restic -r rclone:gdrive:restic-thetiptop restore <SNAPSHOT_ID> --target / --include /git-backup && echo RESTAURATION OK && ls /git-backup/"],
        "env": [
          {"name": "RESTIC_PASSWORD", "valueFrom": {"secretKeyRef": {"name": "restic-secret", "key": "RESTIC_PASSWORD"}}}
        ],
        "volumeMounts": [
          {"name": "rclone-config", "mountPath": "/rclone", "readOnly": true},
          {"name": "restore-target", "mountPath": "/git-backup"}
        ]
      }],
      "volumes": [
        {"name": "rclone-config", "secret": {"secretName": "rclone-config"}},
        {"name": "restore-target", "hostPath": {"path": "/home/anisky/git-restore", "type": "DirectoryOrCreate"}}
      ]
    }
  }'
```

Suivre les logs :

```bash
kubectl logs -n infra restic-restore-git -f
```

Le message `RESTAURATION OK` suivi de la liste des repos confirme le succès.

### 3. Extraire le code depuis un bare clone

Les bare clones sont restaurés dans `/home/anisky/git-restore/`. Pour récupérer le code :

```bash
# Autoriser git à accéder au répertoire (créé par root via le pod)
git config --global --add safe.directory /home/anisky/git-restore/<repo>.git

# Cloner depuis le bare clone local
git clone /home/anisky/git-restore/<repo>.git <destination>

# Exemple
git clone /home/anisky/git-restore/the-tip-top-front.git the-tip-top-front-restored
```

### 4. Nettoyer

```bash
kubectl delete pod restic-restore-git -n infra

# sudo requis car les fichiers ont été créés par root (pod Alpine)
sudo rm -rf ~/git-restore
sudo rm -rf ~/<destination>
```