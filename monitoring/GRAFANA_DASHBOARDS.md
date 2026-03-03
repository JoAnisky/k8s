# 📊 Dashboards Grafana — Référence

## 🔧 Kubernetes — Composants internes

**Controller Manager**
Surveille les contrôleurs K8s (deployments, replicasets).
Utile si un deployment ne se met pas à jour ou si des pods ne se créent pas.

**Kubelet**
Agent qui tourne sur chaque nœud et démarre les pods.
Utile si des pods restent en `Pending` ou crashent au démarrage.

**Scheduler**
Décide sur quel nœud placer chaque pod.
Peu utile sur un cluster mono-nœud, mais aide à diagnostiquer des pods non schedulés.

**Proxy**
Gère les règles réseau internes (iptables).
Utile si des services ne se joignent pas entre eux.

---

## 🌐 Kubernetes — Réseau

**Networking / Cluster**
Vue globale du trafic réseau sur l'ensemble du cluster.

**Networking / Namespace (Pods)**
Trafic réseau par pod dans un namespace donné.
Utile pour voir quel pod consomme de la bande passante.

**Networking / Namespace (Workload)**
Même chose mais agrégé par deployment plutôt que par pod individuel.

**Networking / Pod**
Zoom sur un pod spécifique.

**Networking / Workload**
Zoom sur un deployment spécifique.

---

## 💾 Kubernetes — Stockage & Ressources

**Compute Resources / Workload**
CPU et RAM par deployment.
⭐ **Le plus utile du quotidien** : voir si `symfony-api` ou `mariadb` saturent leurs limites.

**Persistent Volumes**
Utilisation des PVC.
⚠️ **À surveiller pour MariaDB** : si le PVC de 5Gi se remplit, la base plante.

---

## 🖥️ Node Exporter — VPS

**Node Exporter / Nodes**
CPU, RAM, disque, charge du VPS.
⭐ **Premier dashboard à consulter en cas de lenteur générale.**

**USE Method / Node**
Vue synthétique Utilization/Saturation/Errors pour le nœud.
Bon pour détecter rapidement un goulot d'étranglement.

**USE Method / Cluster**
Même chose à l'échelle cluster (redondant sur mono-nœud).

**Node Exporter / AIX** et **Node Exporter / MacOS**
❌ Inutiles dans ce contexte (VPS Linux).

---

## 📊 Application & Base de données

**Symfony Application Monitoring** / **Symfony Application Overview**
Métriques HTTP de l'API TheTipTop : requêtes totales, codes de réponse, temps de réponse p50/p95/p99.
⭐ **Premier dashboard à consulter en cas d'erreurs utilisateur.**

**MySQL Exporter Quickstart and Dashboard**
Métriques MariaDB : connexions, requêtes/sec, requêtes lentes, InnoDB buffer pool, threads.
⭐ **À consulter si l'API est lente et que Symfony semble normal.**

---

## 🔍 Prometheus

**Prometheus / Overview**
Prometheus qui se surveille lui-même.
Utile uniquement si tu suspectes un problème dans la collecte des métriques.

---

## 🚨 Résumé — quel dashboard ouvrir selon la situation

| Situation | Dashboard |
|---|---|
| L'API répond lentement | Symfony Application Overview → MySQL Exporter |
| Le VPS est lent ou surchargé | Node Exporter / Nodes |
| Un pod crashe ou ne démarre pas | Kubernetes / Compute Resources / Workload |
| La BDD risque de manquer de place | Kubernetes / Persistent Volumes |
| Erreurs réseau entre services | Kubernetes / Networking / Namespace |
| Les métriques semblent absentes | Prometheus / Overview |