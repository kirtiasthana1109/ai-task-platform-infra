# AI Task Platform — Infrastructure

Kubernetes manifests and ArgoCD configuration for the [AI Task Processing Platform](../ai-task-platform). This repo is the GitOps source of truth — ArgoCD watches `k8s/` here and applies whatever's committed.

## Layout

```
k8s/
  namespace.yaml         ai-task-platform namespace
  configmap.yaml         non-secret config shared by backend/worker
  secret.yaml            template only — see below, never commit real values
  mongo/                 StatefulSet + headless Service (persistent volume)
  redis/                 Deployment + Service (queue)
  backend/                Deployment + Service + HorizontalPodAutoscaler
  frontend/               Deployment + Service
  worker/                 Deployment + HorizontalPodAutoscaler (no Service — pure consumer)
  ingress.yaml            routes /api -> backend, / -> frontend
  kustomization.yaml      ties all of the above together, and is what CI bumps image tags in
argocd/
  application.yaml        ArgoCD Application, auto-sync enabled
```

## Before you deploy

1. **Secrets**: `k8s/secret.yaml` is a placeholder with `REPLACE_ME` values — do not apply it as-is. Generate a real one and apply it directly (don't commit it):
   ```bash
   kubectl create secret generic app-secrets \
     --namespace ai-task-platform \
     --from-literal=JWT_SECRET="$(openssl rand -hex 32)" \
     --from-literal=MONGO_ROOT_USERNAME=admin \
     --from-literal=MONGO_ROOT_PASSWORD="$(openssl rand -hex 16)" \
     --from-literal=MONGO_URI="mongodb://admin:<same-password>@mongo:27017/ai_task_platform?authSource=admin"
   ```
2. **Image names**: `kustomization.yaml` and the three Deployments reference `yourdockerhubusername/ai-task-platform-*`. Replace that placeholder with your actual Docker Hub username everywhere it appears before the first deploy.
3. **Ingress controller**: the Ingress assumes an `nginx` IngressClass is already installed on the cluster (k3s ships Traefik by default — either install the ingress-nginx controller, or swap `ingressClassName` to `traefik` and adjust annotations).

## Deploying with ArgoCD

```bash
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml -n argocd
kubectl apply -f argocd/application.yaml
```

`application.yaml` has `syncPolicy.automated` with `prune` and `selfHeal` on, so once it's created, any commit to this repo's `k8s/` folder is automatically applied to the cluster — no manual `kubectl apply` needed after the first bootstrap. See `../LOCAL_DEMO_GUIDE.md` for a full walkthrough of standing this up on a local k3d cluster and getting a dashboard screenshot.

## Applying manifests directly (without ArgoCD)

```bash
kubectl apply -k k8s/
```

Useful for a quick sanity check before wiring up GitOps.

## Scaling the worker manually

```bash
kubectl scale deployment worker -n ai-task-platform --replicas=5
```

The HPA in `k8s/worker/hpa.yaml` will otherwise adjust this automatically based on CPU (2-10 replicas) — a manual scale is overridden by the HPA on its next evaluation.
