# 📊 Stack Monitoring

Ce dossier contient tous les manifests Kubernetes et la configuration Helm pour la stack de monitoring.
La stack est **transverse** — elle surveille l'ensemble du cluster, pas uniquement TheTipTop.

## Architecture

```
[Symfony pod — repo: the-tip-top]
  └── artprima/prometheus-metrics-bundle
        └── expose GET /metrics/prometheus
              └── retourne les compteurs HTTP en format texte Prometheus

[PodMonitor — repo: k8s-infra/monitoring/]
  └── dit à Prometheus : "scrape les pods symfony-api dans the-tip-top-api"
  └── label release: kube-prometheus-stack → Prometheus Operator le détecte

[Prometheus]
  └── toutes les 15s, appelle /metrics/prometheus sur le pod Symfony
  └── stocke les valeurs dans sa base de données time-series (TSDB)

[Grafana]
  └── interroge Prometheus via des requêtes PromQL
  └── affiche les résultats en graphiques sur le dashboard
```

## Composants

| Composant | Rôle |
|---|---|
| **Prometheus** | Scrape et stocke les métriques |
| **Grafana** | Visualisation des métriques (dashboards) |
| **Alertmanager** | Gestion des alertes |
| **kube-state-metrics** | Métriques Kubernetes (pods, deployments...) |
| **node-exporter** | Métriques infra du nœud (CPU, RAM, disque) |
| **Prometheus Operator** | Gère Prometheus dynamiquement via les CRDs Kubernetes |

Tous ces composants sont installés d'un seul coup via le Helm chart **`kube-prometheus-stack`**.

## Fichiers

```
monitoring/
├── kustomization.yaml          ← liste des manifests appliqués par kubectl apply -k
├── namespace.yaml              ← namespace "monitoring"
├── helm-values.yaml            ← configuration du chart kube-prometheus-stack (sans secrets)
├── ingress-grafana.yaml        ← exposition de Grafana via Traefik + TLS
└── podmonitor-symfo.yaml       ← cible de scraping pour le pod Symfony (the-tip-top-api)
```

> ℹ️ **Note architecture** : les `PodMonitor` des autres projets doivent être ajoutés ici au fur
> et à mesure. Chaque projet expose ses métriques, ce repo centralise la découverte.

## Concepts clés

### Prometheus Operator et PodMonitor

`kube-prometheus-stack` ne déploie pas Prometheus seul — il déploie aussi un **Prometheus Operator**,
un contrôleur Kubernetes qui surveille des ressources personnalisées (CRDs) comme `PodMonitor` et
`ServiceMonitor`.

Quand on crée un `PodMonitor`, l'Operator le détecte et reconfigure Prometheus automatiquement pour
scraper les pods ciblés. **Sans l'Operator, il faudrait modifier manuellement la config de Prometheus
à chaque nouveau service à surveiller.**

> ⚠️ L'Operator ne détecte que les PodMonitors qui ont le label `release: kube-prometheus-stack`.
> Sans ce label, le PodMonitor existe dans Kubernetes mais Prometheus l'ignore complètement.

### Annotations `prometheus.io/scrape` vs PodMonitor

Les annotations `prometheus.io/scrape: "true"` sur les pods sont une convention de **l'ancien
Prometheus standalone**. Elles ne fonctionnent pas avec `kube-prometheus-stack` qui utilise
l'Operator — elles sont ignorées. Il faut obligatoirement passer par un `PodMonitor` ou un
`ServiceMonitor`.

### Namespace du bundle Symfony (`thetiptop`)

Le namespace configuré dans `artprima_prometheus_metrics.yaml` est le **préfixe des noms de métriques**
exposées. Avec `namespace: thetiptop`, les métriques s'appellent :

- `thetiptop_http_requests_total`
- `thetiptop_http_2xx_responses_total`
- `thetiptop_request_durations_histogram_seconds`

Ce namespace doit correspondre à la variable **Namespace** dans le dashboard Grafana.

### Stockage APCu

Le bundle Symfony stocke les compteurs en mémoire via **APCu** (`PROM_METRICS_DSN=apcu://localhost`).
Les métriques sont perdues au redémarrage du pod — c'est acceptable car Prometheus conserve
l'historique dans sa propre TSDB.

## Déploiement

Le déploiement est entièrement géré par le **Jenkinsfile de ce repo** (`k8s/Jenkinsfile`),
dans le stage `Deploy Monitoring Stack`.

```bash
# Namespace + Ingress + PodMonitors
kubectl apply -k monitoring/

# Secret Grafana — écrit directement dans le secret géré par Helm
kubectl create secret generic kube-prometheus-stack-grafana \
  --from-literal=admin-user=<GRAFANA_ADMIN_USER> \
  --from-literal=admin-password=<GRAFANA_ADMIN_PASSWORD> \
  --from-literal=ldap-toml='' \
  --namespace monitoring \
  --dry-run=client -o yaml | kubectl apply -f -

# Install / upgrade via Helm
helm upgrade --install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values monitoring/helm-values.yaml \
  --set grafana.adminPassword=<GRAFANA_ADMIN_PASSWORD> \
  --wait --timeout 5m
```

> 🔐 Les secrets Grafana ne sont **jamais** dans `helm-values.yaml` ni dans le repo Git.
> Ils sont injectés au moment du déploiement via les credentials Jenkins
> `GRAFANA_ADMIN_USER` et `GRAFANA_ADMIN_PASSWORD`.

## Accès

| Service | URL |
|---|---|
| Grafana | https://grafana.the-tip-top.jonathanlore.fr |

Identifiants : voir les credentials Jenkins `GRAFANA_ADMIN_USER` / `GRAFANA_ADMIN_PASSWORD`.

## Ajouter le monitoring d'un nouveau projet

1. Créer un `PodMonitor` dans ce dossier :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: <nom-du-projet>
  namespace: monitoring
  labels:
    release: kube-prometheus-stack   # ← obligatoire
spec:
  namespaceSelector:
    matchNames:
      - <namespace-du-projet>
  selector:
    matchLabels:
      app: <label-du-pod>
  podMetricsEndpoints:
    - path: /metrics/prometheus
      port: http
```

2. L'ajouter dans `kustomization.yaml`
3. Commiter — le pipeline Jenkins déploiera automatiquement

## Dashboard Symfony (TheTipTop)

Le dashboard est importé depuis le fichier JSON fourni par le bundle dans le repo `the-tip-top` :

```
vendor/artprima/prometheus-metrics-bundle/grafana/symfony-app-overview.json
```

Dans Grafana : **Dashboards → New → Import → Upload JSON file**

Après import, sélectionner la datasource Prometheus et changer la variable **Namespace**
de `symfony` à `thetiptop`.

### Métriques disponibles

| Métrique | Type | Description |
|---|---|---|
| `thetiptop_http_requests_total` | Counter | Nombre total de requêtes HTTP |
| `thetiptop_http_2xx_responses_total` | Counter | Requêtes avec réponse 2xx |
| `thetiptop_http_4xx_responses_total` | Counter | Erreurs client (4xx) |
| `thetiptop_request_durations_histogram_seconds` | Histogram | Temps de réponse (p50 / p95 / p99) |

## Dashboards recommandés (à importer via ID Grafana.com)

| Dashboard | ID | Description |
|---|---|---|
| Node Exporter Full | `1860` | CPU, RAM, disque, réseau du nœud |
| Kubernetes cluster | `315` | Vue globale des pods et deployments |
| MySQL Overview | `7362` | Métriques MariaDB (à venir avec mysqld-exporter) |