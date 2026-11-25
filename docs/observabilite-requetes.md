# Observabilité — bibliothèque de requêtes

Requêtes PromQL (Prometheus) et LogQL (Loki) utilisées sur le lab `astech-gitops-lab`,
avec ce qu'elles mesurent et les pièges rencontrés.

Stack : kube-prometheus-stack (Prometheus + Grafana), Loki (SingleBinary), Alloy (collecte des logs), sur un cluster kind.
Environnements observés : `astech-dev` (HPA 2 → 5 pods) et `astech-prod` (3 pods).

> Usage : Grafana → **Explore** → choisir la source (Prometheus ou Loki) → mode **Code**.
> Type **Instant** pour un chiffre, **Range** pour une courbe.

---

## 1. Les briques à connaître

### PromQL

| Brique | Rôle | Exemple |
|---|---|---|
| `{label="valeur"}`, `=~`, `!=` | Filtrer les séries | `{namespace=~"astech-.*"}` |
| `rate(x[5m])` | Vitesse par seconde d'un **compteur** (`_total`) | `rate(container_cpu_usage_seconds_total[5m])` |
| `sum by (label)` | Agréger par dimension | `sum by (pod) (...)` |
| `count`, `max`, `topk(N, ...)` | Compter, maximum, N premiers | `count(kube_pod_info)` |
| `max_over_time(x[5m])` | Maximum sur une fenêtre | utilisé dans les alertes |
| `/`, `* 100` | Ratios et pourcentages | consommation / limite |

**Compteur ou jauge ?**
- Compteur (`_total`, ne fait qu'augmenter) → toujours avec `rate()`.
- Jauge (valeur instantanée : mémoire, nombre de pods) → s'utilise directement.

### LogQL

| Brique | Rôle | Exemple |
|---|---|---|
| `{namespace="..."}` | Sélectionner les flux (labels indexés) | `{namespace="astech-prod", app="astech-app"}` |
| `\|= "texte"` / `!= "texte"` | Garder / exclure les lignes contenant un texte | `!= "kube-probe"` |
| `\|~ "regex"` | Filtrer par expression régulière | `\|~ "\" 5[0-9]{2} "` |
| `count_over_time(...[5m])` | Transformer des logs en nombre de lignes | volume, taux d'erreur |
| `rate(...[5m])` | Lignes par seconde | débit de logs |
| `\| pattern` / `\| json` | Extraire des champs du texte au moment de la requête | code HTTP, chemin |

---

## 2. Logs de l'application (Loki)

**Logs de la prod**
```logql
{namespace="astech-prod"}
```

**Trafic réel uniquement (sans les probes Kubernetes)**
```logql
{namespace=~"astech-.*", app="astech-app"} != "kube-probe"
```

**Un seul pod**
```logql
{namespace="astech-prod", pod=~".*nr45p"}
```

**Vraies erreurs nginx (hors bruit du favicon)**
```logql
{namespace=~"astech-.*"} |= "[error]" != "favicon.ico"
```

**Nombre de requêtes HTTP en prod sur 15 min**
```logql
sum(count_over_time({namespace="astech-prod", app="astech-app"} != "kube-probe" != "[error]" [15m]))
```
> Piège : un 404 produit deux lignes (accès + erreur nginx). Exclure `[error]` évite de compter deux fois.

**Répartition du trafic par pod**
```logql
sum by (pod) (count_over_time({namespace="astech-prod", app="astech-app"} != "kube-probe" != "[error]" [15m]))
```

**Taux d'erreur HTTP (4xx + 5xx) en %**
```logql
100 *
sum(count_over_time({namespace="astech-prod", app="astech-app"} != "kube-probe"
  | pattern `<_> - - [<_>] "<_> <_> <_>" <status> <_>`
  | status >= 400 [15m]))
/
sum(count_over_time({namespace="astech-prod", app="astech-app"} != "kube-probe"
  | pattern `<_> - - [<_>] "<_> <_> <_>" <status> <_>`
  | status != "" [15m]))
```
> Constat : taux gonflé par les 404 de `favicon.ico` demandés automatiquement par le navigateur.
> Un indicateur doit mesurer ce qui compte pour l'utilisateur : exclure le bruit ou corriger la cause.

**Top 5 des URL demandées**
```logql
topk(5, sum by (path) (count_over_time({namespace="astech-dev", app="astech-app"} != "kube-probe"
  | pattern `<_> - - [<_>] "<method> <path> <_>" <_>` [1h])))
```
> `path` est extrait à la requête, jamais stocké comme label indexé (voir cardinalité).

**Volume de logs par environnement (à afficher en Time series)**
```logql
sum by (namespace) (count_over_time({namespace=~"astech-.*"}[5m]))
```

**Débit de logs généré par les probes (lignes/s)**
```logql
sum(rate({namespace=~"astech-.*"} |= "kube-probe" [5m]))
```
> ≈ 0,3 ligne/s par pod (readiness 5 s + liveness 10 s). À 50 pods : ~1,3 M lignes/jour sans valeur.
> Remèdes : route `/healthz` avec `access_log off;` dans nginx, ou `stage.drop` dans Alloy.

**Vérifier qu'une correction a pris effet (ex. favicon)**
```logql
sum(count_over_time({namespace="astech-dev"} |= "favicon.ico" [5m]))
```
> La courbe doit tomber à zéro au moment du rolling update.

---

## 3. Ressources des pods (Prometheus)

**Mémoire par pod en Mi**
```promql
container_memory_working_set_bytes{namespace="astech-prod", container="astech-app"} / 1024 / 1024
```
> `working_set` est la valeur comparée à la limite par le kubelet pour l'OOMKill.

**Mémoire en % de la limite**
```promql
100 *
  sum by (pod) (container_memory_working_set_bytes{namespace="astech-prod", container="astech-app"})
/
  sum by (pod) (kube_pod_container_resource_limits{namespace="astech-prod", container="astech-app", resource="memory"})
```
> Piège : labels différents entre cAdvisor et kube-state-metrics → `sum by (pod)` des deux côtés,
> sinon résultat vide. Constat : ~5-15 % → limite surdimensionnée (right-sizing).

**CPU par pod en millicores**
```promql
1000 * sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="astech-dev", container="astech-app"}[5m]))
```
> Cohérence avec l'HPA : 2 % de 50m (requests) ≈ 1m.

**CPU bridé par la limite (throttling)**
```promql
rate(container_cpu_cfs_throttled_periods_total{namespace="astech-dev", container="astech-app"}[1m])
```
> Un CPU moyen bas peut cacher un CPU bridé pendant les pics : cause classique de lenteur « invisible ».

---

## 4. État des objets Kubernetes (Prometheus / kube-state-metrics)

**Pods disponibles par environnement**
```promql
kube_deployment_status_replicas_available{namespace=~"astech-.*"}
```

**Nombre de pods (total et par namespace)**
```promql
count(kube_pod_info)
count by (namespace) (kube_pod_info)
sum by (namespace) (kube_pod_status_phase{phase="Running"})
```

**Redémarrages des conteneurs**
```promql
sum by (pod) (kube_pod_container_status_restarts_total{namespace=~"astech-.*"})
```

**Écart entre réplicas voulus et disponibles**
```promql
kube_deployment_spec_replicas{namespace="astech-prod"} - kube_deployment_status_replicas_available{namespace="astech-prod"}
```

**Date de création des pods**
```promql
kube_pod_created{namespace="astech-prod"}
```

**Dater un rolling update (ancien et nouveau ReplicaSet)**
```promql
count by (replicaset) (kube_pod_info{namespace="astech-dev", created_by_kind="ReplicaSet"})
kube_deployment_metadata_generation{namespace="astech-dev"}
```

---

## 5. Autoscaling (HPA)

**Réplicas voulus vs actuels**
```promql
kube_horizontalpodautoscaler_status_desired_replicas{namespace="astech-dev"}
kube_horizontalpodautoscaler_status_current_replicas{namespace="astech-dev"}
```

Observation du test de charge (`hey -z 3m -c 50`) :
- délai avant le premier nouveau pod : ~30-60 s (metrics-server + boucle HPA + démarrage) ;
- CPU plafonné à 200m (`limits.cpu`) → throttling, pas de kill ;
- retour à 2 pods ~1-2 min après la charge (`stabilizationWindowSeconds: 60`).

> Méthode : Explore → **Split** (Prometheus à gauche, Loki à droite) → actualisation 5 s.

---

## 6. Alertes

**Règle fournie : `KubePodCrashLooping`**
```promql
max_over_time(kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff", job="kube-state-metrics"}[5m]) >= 1
```
`for: 15m` → n'alerte que sur un problème persistant (évite le flapping pendant un déploiement).

**Règle perso : moins de 3 pods en prod pendant 1 min**
```promql
sum(kube_deployment_status_replicas_available{namespace="astech-prod"})
```
Condition `IS BELOW 3`, pending period `1m`.
> Supprimer un pod ne déclenche pas l'alerte (remplacé en quelques secondes) : c'est voulu.
> Test : `kubectl scale deploy astech-prod-astech-app -n astech-prod --replicas=2` (prod sans selfHeal), puis Sync dans Argo.

**Règle perso basée sur les logs : > 10 erreurs nginx en 5 min**
```logql
sum(count_over_time({namespace="astech-prod"} |= "[error]" != "favicon.ico" [5m]))
```
Condition `IS ABOVE 10`. Déclenchement :
```bash
for i in $(seq 15); do
  curl -s -o /dev/null -H "Host: astech.93.127.162.226.nip.io" http://localhost:8081/page-inexistante-$i
done
```
> Alerte sur logs : détecte des problèmes applicatifs, mais coûteuse et fragile si le format change.
> Préférer une métrique exposée par l'application quand c'est possible.

---

## 7. À retenir

1. **Exclure le bruit** (probes, favicon) avant de calculer un indicateur.
2. **Résolution** : Prometheus collecte toutes les 30 s ; un événement plus bref peut lui échapper → croiser avec les logs et les events Kubernetes.
3. **Corréler** métriques et logs sur la même période (Split).
4. **Alerter sur la durée** : `for` / pending period pour ne signaler que les problèmes persistants.
5. **Cardinalité** : labels stables et peu nombreux (`namespace`, `app`, `pod`). Jamais d'IP client, d'ID utilisateur, d'ID de requête ou d'URL complète en label ; extraire ces champs au moment de la requête.
6. **Méthode RED** pour une application lente : Rate (débit), Errors (erreurs), Duration (latence).

## 8. Manques identifiés dans le lab

| Manque | Impact | Piste |
|---|---|---|
| Pas de latence dans les logs nginx | Impossible de mesurer la durée de réponse | Ajouter `$request_time` au `log_format` |
| Métriques Traefik non collectées | Pas d'histogramme de latence ni de taux d'erreur côté Ingress | `ServiceMonitor` pour Traefik |
| Pas de traces | Impossible de suivre une requête entre services | OpenTelemetry + Grafana Tempo |
| IP client perdue (socat + NodePort) | Pas de blocage, de rate limiting ni d'audit par IP | `externalTrafficPolicy: Local`, protocole PROXY |
| Prometheus en `emptyDir` | Historique perdu si le pod est recréé | `storageSpec` avec un PVC |
