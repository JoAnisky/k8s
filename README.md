# 🏗️ Infrastructure K3s — jonathanlore.fr

Ce repo centralise toute l'infrastructure Kubernetes du VPS.
Le déploiement est automatisé via le **Jenkinsfile** à la racine.

## Structure

```
k8s-infra/
├── Jenkinsfile                 ← pipeline CI/CD de l'infra
├── Dockerfile                  ← image Jenkins custom (Docker + kubectl + Helm)
├── README.md
│
├── traefik/
│   ├── traefik-config.yaml     ← configuration Traefik (Let's Encrypt, entrypoints)
│   └── dashboard-ingress.yaml  ← exposition du dashboard Traefik
│
├── jenkins/
│   ├── kustomization.yml
│   ├── namespace.yml
│   ├── pvc.yml
│   ├── deployment.yml
│   ├── service.yml
│   └── ingress.yml
│
└── monitoring/
    ├── kustomization.yaml
    ├── namespace.yaml
    ├── helm-values.yaml        ← config kube-prometheus-stack (sans secrets)
    ├── ingress-grafana.yaml
    └── README.md               ← documentation détaillée de la stack monitoring
```

## Services exposés

| Service | URL | Namespace |
|---|---|---|
| Jenkins | https://jenkins.jonathanlore.fr | `jenkins` |
| Grafana | https://grafana.jonathanlore.fr | `monitoring` |
| Traefik | https://traefik.jonathanlore.fr | `kube-system` |

## Déploiement automatisé

Tout push sur `main` déclenche le pipeline Jenkins qui :

1. **Build** l'image Jenkins custom et la push sur Docker Hub
2. **Déploie Traefik** — config Let's Encrypt + dashboard ingress
3. **Déploie Jenkins** — via `kubectl apply -k jenkins/`
4. **Déploie la stack Monitoring** — via Helm (`kube-prometheus-stack`) + manifests
5. **Vérifie** l'état des pods et des ingress
6. **Health checks** — attend que Grafana et Jenkins répondent

### Credentials Jenkins requis

| ID | Type | Usage |
|---|---|---|
| `jenkins-dockerhub` | Username/Password | Push image Docker |
| `kubeconfig` | File | Accès au cluster K3s |
| `GRAFANA_ADMIN_USER` | Secret text | Login Grafana |
| `GRAFANA_ADMIN_PASSWORD` | Secret text | Mot de passe Grafana |

## Déploiement manuel (urgence)

Si Jenkins est indisponible, les commandes manuelles sont :

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

**Reset complet Jenkins :**
```bash
kubectl delete namespace jenkins --ignore-not-found=true
sleep 15
kubectl apply -k jenkins/
```