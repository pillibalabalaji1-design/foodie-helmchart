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


## Use your own domain (GoDaddy + NGINX Ingress)

1. Get the NGINX Ingress external address:

```bash
kubectl get svc -n ingress-nginx
```

2. In GoDaddy DNS:
- If you use root domain (`example.com`): create/update an **A** record to the ingress LoadBalancer IP.
- If you use subdomain (`app.example.com`): create/update a **CNAME** record to the ingress LoadBalancer hostname.

3. Set your chart host to that domain in values:

```yaml
ingress:
  host: app.example.com
```

4. Sync/redeploy the chart and verify:

```bash
kubectl get ingress -n foodie-app
```

> Frontend container listens on port `3000`; service/ingress still expose HTTP port `80` externally.
