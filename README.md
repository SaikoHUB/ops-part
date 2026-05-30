# Ops Part — Prometheus & Grafana sur K3s

## 👤 Auteur
Étudiant : JB (Saiko)  
Rôle : Ops — Déploiement Prometheus + Grafana + Service Discovery + Alerting + AWX

## 📝 Contexte du projet
Ce repo fait partie du projet de groupe DevOps-010 "Corréler métriques et logs".  
La stack est déployée sur un VPS OVH avec K3s (Kubernetes léger).

Répartition du groupe :
- **Coco (Dev)** : App Flask instrumentée avec Prometheus + logs JSON → [repo](https://github.com/Coco-Bd/ynov-DevOps-010)
- **JB / Saiko (Ops)** : Déploiement Prometheus + Grafana + Service Discovery ← vous êtes ici
- **Enzo / Rodove (Analyste)** : Stack Loki/Promtail + corrélation logs → [repo](https://github.com/rodove-tv/DevOps_analyst)

## 🏗️ Architecture

    Flask App (port 5000)
          ↓ scrape /metrics toutes les 15s
    Prometheus (port 31090) ← Kubernetes SD détecte le pod via label app=flask-app
          ↓ datasource
    Grafana (port 31300) ← Dashboard 4 panels + seuils visuels

## 📂 Structure du repo

    ops-part/
    ├── ansible/
    │   └── prometheus.yml         # Playbook Ansible déployé via AWX
    ├── prometheus-rbac.yml        # ServiceAccount + ClusterRole + ClusterRoleBinding
    ├── prometheus-config.yml      # ConfigMap Prometheus + Kubernetes SD + règles d'alerte
    ├── prometheus-deployment.yml  # Deployment + Service NodePort Prometheus
    ├── grafana-deployment.yml     # Deployment + Service NodePort Grafana
    ├── dashboard.json             # Export du dashboard Grafana
    └── README.md

## 🚀 Déploiement manuel

    # 1. Permissions RBAC (nécessaires pour le Kubernetes Service Discovery)
    kubectl apply -f prometheus-rbac.yml

    # 2. Configuration Prometheus + règles d'alerte
    kubectl apply -f prometheus-config.yml

    # 3. Déploiement Prometheus
    kubectl apply -f prometheus-deployment.yml

    # 4. Déploiement Grafana
    kubectl apply -f grafana-deployment.yml

## 🤖 Déploiement via AWX
Le playbook `ansible/prometheus.yml` est synchronisé depuis ce repo Git dans AWX.  
Il déploie automatiquement toute la stack et attend que Prometheus et Grafana soient prêts.

Résultat du dernier job AWX :

    TASK [Gathering Facts]          ok
    TASK [Apply RBAC]               ok
    TASK [Apply Prometheus config]  changed
    TASK [Apply Prometheus deployment] ok
    TASK [Apply Grafana deployment] ok
    TASK [Wait for Prometheus to be ready] ok
    TASK [Wait for Grafana to be ready]    ok
    localhost : ok=7  changed=1  unreachable=0  failed=0

## ✅ Vérifications

    # Vérifier que les pods tournent
    kubectl get pods | grep -E "prometheus|grafana"

    # Vérifier les services exposés
    kubectl get services | grep -E "prometheus|grafana"

    # Vérifier que Flask expose bien ses métriques
    curl http://10.42.0.54:5000/metrics | head -20

    # Vérifier les targets Prometheus (doit afficher flask-app UP)
    curl http://localhost:31090/api/v1/targets | python3 -m json.tool

## 🧪 Tests de charge

    # Générer du trafic sur l'app Flask (200 requêtes OK + 200 erreurs)
    for i in $(seq 1 200); do
      curl -s http://10.42.0.54:5000/ok > /dev/null
      curl -s http://10.42.0.54:5000/error > /dev/null
    done

    # Trafic continu avec délai (observer les courbes en temps réel)
    for i in $(seq 1 50); do
      curl -s http://10.42.0.54:5000/ok > /dev/null
      curl -s http://10.42.0.54:5000/error > /dev/null
      sleep 0.5
    done

## 🔔 Alertes Prometheus
3 règles d'alerte configurées dans la ConfigMap :

| Alerte | Condition | Sévérité | Délai |
|--------|-----------|----------|-------|
| HighErrorRate | `rate(http_requests_total{status="500"}[1m]) > 0.1` | warning | 30s |
| FlaskDown | `up{job="flask-app"} == 0` | critical | 10s |
| HighRequestRate | `rate(http_requests_total[1m]) > 10` | info | 30s |

## 📊 Dashboard Grafana
4 panels disponibles sur `http://57.128.51.49:31300` (admin / admin123) :

| Panel | Requête PromQL | Description |
|-------|---------------|-------------|
| Requêtes HTTP totales | `http_requests_total` | Compteur cumulatif des requêtes 200 et 500 |
| Taux d'erreurs 500 | `rate(http_requests_total{status="500"}[1m])` | Taux d'erreurs par seconde — seuil rouge à 0.1 |
| Requêtes par seconde | `rate(http_requests_total[1m])` | Charge globale — seuil orange à 10 |
| Uptime Flask | `process_start_time_seconds{job="flask-app"}` | Date de démarrage du process Flask |

## 🔗 Accès
| Service | URL |
|---------|-----|
| Prometheus | http://57.128.51.49:31090 |
| Grafana | http://57.128.51.49:31300 |
| Flask App | http://57.128.51.49:31000 |
| AWX | http://57.128.51.49:31358 |