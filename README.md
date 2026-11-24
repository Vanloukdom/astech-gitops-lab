# astech-gitops-lab

Déploiement GitOps d'une application conteneurisée sur Kubernetes (kind) avec **Helm** et **Argo CD**.

```
VS Code ──git push──► GitHub ◄──surveille── Argo CD (cluster kind) ──► namespace astech
```

## Contenu

| Dossier | Rôle |
|---|---|
| `app/` | Image de l'application (nginx + `index.html`) |
| `helm/astech-app/` | Chart Helm : Deployment (probes), Service, Ingress Traefik |
| `argocd/astech-app.yaml` | Application Argo CD : lit le chart dans ce dépôt et le déploie |

## Prérequis

- Cluster kind `monapp`, contrôleur Ingress Traefik, Argo CD installés
- `kubectl config current-context` → `kind-monapp`

## Déploiement

```bash
# 1. Construire l'image et la charger dans kind (pas de registre en lab)
docker build -t astech-app:1.0 ./app
kind load docker-image astech-app:1.0 --name monapp

# 2. Déclarer l'application à Argo CD (une seule fois)
kubectl apply -f argocd/astech-app.yaml

# 3. Vérifier
kubectl get applications -n argocd
kubectl get pods -n astech
```

Ensuite, tout changement passe par Git : modifier `helm/astech-app/values.yaml`, commit, push → Argo CD synchronise.

## Corrections apportées au chart d'origine

- `deployment.yaml` : faute de frappe `.Values.iamge` → `.Values.image`
- `ingress.yaml` : API `extensions/v1beta1` (supprimée en 1.22) → `networking.k8s.io/v1`
- `values.yaml` : tag `latest` → `1.0`, `pullPolicy: Never` (image chargée localement)
- Ajout des readiness/liveness probes



## Challenge 3 — Ressources, autoscaling (HPA) et PodDisruptionBudget

### Objectif

Faire réagir l'application à la charge : en **dev**, le nombre de pods s'ajuste automatiquement
entre 2 et 5 selon le CPU consommé, tandis que la **prod** conserve 3 réplicas fixes.
Le tout reste piloté en GitOps par Argo CD, sans conflit entre l'HPA et la synchronisation.

```
hey (charge) ──► Service astech-dev ──► pods astech-dev
                                           │ CPU mesuré
                          metrics-server ◄─┘
                                │
                               HPA : CPU moyen > 50 % des requests ? ──► 2 → 5 pods
```

### Ce qui a été ajouté

| Élément | Fichier | Rôle |
|---|---|---|
| Requests / limits | `helm/astech-app/values.yaml` | `requests: 50m CPU / 32Mi`, `limits: 200m CPU / 64Mi` — base de calcul de l'HPA |
| Autoscaling | `values.yaml` (désactivé) + `values-dev.yaml` (activé) | HPA de 2 à 5 pods, cible 50 % de CPU |
| PodDisruptionBudget | `values.yaml` (désactivé) + `values-dev.yaml` (activé) | `minAvailable: 1` pendant les interruptions volontaires |
| `replicas` conditionnel | `templates/deployment.yaml` | Le champ disparaît quand l'HPA est actif |
| HPA | `templates/hpa.yaml` | `autoscaling/v2`, redescente après 60 s de calme |
| PDB | `templates/pdb.yaml` | `policy/v1` |
| metrics-server | `argocd/metrics-server.yaml` | Installé et géré par Argo CD (`--kubelet-insecure-tls` pour kind) |
| Configuration Traefik | `platform/traefik-values.yaml` | NodePort figé et adresse inscrite dans les Ingress |

**Point clé GitOps** : quand l'HPA est actif, `replicas` est retiré du Deployment.
Sinon, l'HPA et Argo CD (`selfHeal: true`) se disputeraient le champ en boucle :
l'HPA monterait à 4 pods, Argo remettrait la valeur de Git, et ainsi de suite.

```yaml
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
```

### Vérification du rendu avant push

```bash
C=helm/astech-app
helm template astech-dev  $C -f $C/values.yaml -f $C/values-dev.yaml  | grep -E '^kind:|^  replicas:|minReplicas|maxReplicas|minAvailable|cpu:'
helm template astech-prod $C -f $C/values.yaml -f $C/values-prod.yaml | grep -E '^kind:|^  replicas:'
```

- **dev** : `HorizontalPodAutoscaler`, `PodDisruptionBudget`, ressources, **aucune** ligne `replicas:`
- **prod** : `replicas: 3`, ni HPA ni PDB

### Test de charge

```bash
kubectl get hpa -n astech-dev -w                 # observation
kubectl run load -n astech-dev --rm -it --restart=Never --image=williamyeh/hey -- \
  -z 3m -c 50 http://astech-dev-astech-app.astech-dev.svc.cluster.local
```

La charge est envoyée **depuis le cluster**, directement au nom DNS interne du Service
(`<service>.<namespace>.svc.cluster.local`), sans passer par Traefik.

### Résultats observés

```
Events (kubectl describe hpa astech-dev-astech-app -n astech-dev) :
  SuccessfulRescale  New size: 5; reason: cpu resource utilization (percentage of request) above target
  SuccessfulRescale  New size: 3; reason: All metrics below target
  SuccessfulRescale  New size: 2; reason: All metrics below target
```

- Au repos : `cpu: 2%/50%`, **2 pods** (plancher `minReplicas`).
- Sous charge : montée directe à **5 pods** — la politique par défaut autorise un doublement toutes les 15 s.
- Après la charge : retour à **2 pods** après la fenêtre de stabilisation de 60 s.
- `ScalingLimited: TooFewReplicas` au repos : l'HPA voudrait 1 pod, `minReplicas: 2` l'en empêche (comportement attendu).
- Argo CD est resté **`Synced`** pendant toute la montée et la descente : aucun conflit avec l'HPA.

### Test du PodDisruptionBudget

```bash
kubectl drain monapp-control-plane --pod-selector=app=astech-app --ignore-daemonsets --dry-run=server
kubectl uncordon monapp-control-plane
```

Simulation sans effet réel (`--dry-run=server`) : avec 2 pods et `minAvailable: 1`, la première éviction
est acceptée, la seconde refusée (`Cannot evict pod as it would violate the pod's disruption budget`).

### Incident résolu : applications bloquées en `Progressing`

**Symptôme** — `astech-dev` et `astech-prod` restaient `Synced` / `Progressing` depuis plusieurs jours,
alors que les pods étaient `Running` et que les pages répondaient.

**Diagnostic** — santé objet par objet dans Argo CD :

```bash
kubectl get application astech-dev -n argocd \
  -o jsonpath='{range .status.resources[*]}{.kind}{"  "}{.name}{"  "}{.health.status}{"\n"}{end}'
kubectl get ingress -A      # colonne ADDRESS vide
```

**Cause** — Argo CD ne considère un Ingress comme `Healthy` que si son statut contient une adresse.
Traefik, exposé en **NodePort** sur kind, ne la renseignait pas (et `publishedService` ne suffit pas :
il recopie une adresse LoadBalancer qui n'existe pas en NodePort).

**Correction** — Traefik inscrit l'IP publique du VPS dans chaque Ingress, configuration versionnée
dans `platform/traefik-values.yaml` :

```yaml
providers:
  kubernetesIngress:
    publishedService:
      enabled: false
additionalArguments:
  - --providers.kubernetesingress.ingressendpoint.ip=93.127.162.226
```

```bash
helm upgrade traefik traefik/traefik -n traefik -f platform/traefik-values.yaml
kubectl -n traefik rollout status deploy/traefik
```

**Résultat** — `ADDRESS` renseignée, applications `Synced` / `Healthy`.

**Leçon** — un `Progressing` permanent dans Argo CD ne signifie pas que l'application est en panne :
Argo applique des règles de santé par type d'objet. On liste la santé de chaque ressource
pour isoler celle qui bloque, plutôt que de chercher dans les pods.

### Ce que ce challenge démontre

- Dimensionner un conteneur avec des **requests et limits** et comprendre leur rôle (scheduler, HPA, OOMKill).
- Mettre en place un **HPA `autoscaling/v2`** avec fenêtre de stabilisation et le rendre compatible avec **GitOps**.
- Protéger la disponibilité pendant les maintenances avec un **PodDisruptionBudget**.
- Activer une fonctionnalité **par environnement** grâce aux fichiers de valeurs Helm.
- Diagnostiquer un statut de santé Argo CD et corriger la configuration du contrôleur Ingress.
