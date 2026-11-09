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
