# foodie-helmchart

Production-ready Helm chart for deploying the **Foodie** cloud kitchen stack on Kubernetes.

## Chart structure

```text
foodie/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── _helpers.tpl
    ├── backend-deployment.yaml
    ├── backend-service.yaml
    ├── configmap-env.yaml
    ├── database-service.yaml
    ├── database-statefulset.yaml
    ├── frontend-deployment.yaml
    ├── frontend-service.yaml
    ├── hpa-backend.yaml
    ├── hpa-frontend.yaml
    ├── ingress.yaml
    └── secret-dockerhub.yaml
```

## Prerequisites

- Kubernetes cluster
- Helm 3+
- NGINX Ingress controller

## Install NGINX Ingress controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install nginx-ingress ingress-nginx/ingress-nginx \
  --set controller.service.type=LoadBalancer
```

## Create DockerHub pull secret

Create in the namespace where you install Foodie:

```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-username=YOUR_USERNAME \
  --docker-password=YOUR_PASSWORD \
  --docker-email=YOUR_EMAIL
```

> If you prefer fully-managed secrets through Helm, set `dockerhubSecret.create=true` and provide `dockerhubSecret.dockerconfigjson` in values.

## Deploy Foodie

```bash
helm install foodie ./foodie --namespace foodie --create-namespace
```

## Routing architecture

- `https://foodie.kitchen/` -> frontend service (`ClusterIP`)
- `https://foodie.kitchen/api` -> backend service (`ClusterIP`)
- database runs as `StatefulSet` with PVC and is internal only (`ClusterIP` headless service)

Only the ingress controller is internet-facing (`LoadBalancer`).
