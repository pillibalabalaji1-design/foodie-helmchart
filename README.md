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
- A domain managed in GoDaddy (or another DNS provider)

## Install NGINX Ingress controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install nginx-ingress ingress-nginx/ingress-nginx \
  --set controller.service.type=LoadBalancer \
  --namespace ingress-nginx --create-namespace
```

## If Ingress has no ADDRESS (fix ingress-nginx service first)

If `kubectl get ingress -n foodie-app` shows no `ADDRESS`, your NGINX controller service is likely not `LoadBalancer`.

1. Check the ingress controller service type:

```bash
kubectl get svc -n ingress-nginx
```

2. If needed, force service type to `LoadBalancer` on the running release:

```bash
helm upgrade --install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=LoadBalancer
```

3. If the service is still `ClusterIP`, patch it directly:

```bash
kubectl patch svc ingress-nginx-controller -n ingress-nginx \
  -p '{"spec":{"type":"LoadBalancer"}}'
```

4. Wait for an external endpoint to be assigned:

```bash
kubectl get svc ingress-nginx-controller -n ingress-nginx -w
```

When `EXTERNAL-IP` becomes an IP/hostname, point your GoDaddy DNS record to that value.

## Create DockerHub pull secret

Create in the namespace where you install Foodie:

```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-username=YOUR_USERNAME \
  --docker-password=YOUR_PASSWORD \
  --docker-email=YOUR_EMAIL \
  -n foodie-app
```

> If you prefer fully-managed secrets through Helm, set `dockerhubSecret.create=true` and provide `dockerhubSecret.dockerconfigjson` in values.

## Domain setup for GoDaddy

1. Open **GoDaddy -> DNS -> Manage Zones** for your purchased domain.
2. Choose your app host (recommended: subdomain such as `foodie.kitchen`).
3. Create DNS record based on ingress controller external endpoint:
   - Use **A** record when `EXTERNAL-IP` is an IPv4 address.
   - Use **CNAME** when `EXTERNAL-IP` is a cloud hostname (common with AWS load balancers).
4. Save and wait for propagation (often a few minutes, can take longer).
5. Validate DNS and ingress:

```bash
dig +short foodie.kitchen
kubectl get ingress -n foodie-app
```

## Point your GoDaddy domain to ingress (required)

1. Get the public endpoint of the NGINX ingress controller service:

```bash
kubectl get svc -n ingress-nginx
kubectl get svc -n ingress-nginx ingress-nginx-controller -o wide
```

2. In GoDaddy DNS, create/update records for your app host:
   - If `EXTERNAL-IP` is an IPv4 address, create an **A** record.
   - If `EXTERNAL-IP` is a hostname (common on AWS), create a **CNAME** record.

Example for subdomain `foodie.kitchen`:
- Type: `CNAME`
- Name: `foodie`
- Value: `<ingress-controller-external-hostname>`
- TTL: default

3. Wait for DNS propagation, then verify:

```bash
dig +short foodie.kitchen
```

## Deploy Foodie with your real domain

Set your host in both ingress and backend CORS values during install/upgrade:

```bash
helm upgrade --install foodie ./foodie \
  -n foodie-app --create-namespace \
  -f foodie/values-foodie-app.yaml \
  --set ingress.host=foodie.kitchen \
  --set backend.env.CORS_ORIGIN=https://foodie.kitchen
```

Verify ingress host and routing:

```bash
kubectl get ingress -n foodie-app
kubectl describe ingress foodie-ingress -n foodie-app
```

## Optional: Enable TLS (HTTPS)

If you have a TLS secret already:

```bash
helm upgrade --install foodie ./foodie \
  -n foodie-app \
  -f foodie/values-foodie-app.yaml \
  --set ingress.host=foodie.kitchen \
  --set backend.env.CORS_ORIGIN=https://foodie.kitchen \
  --set ingress.tls.enabled=true \
  --set ingress.tls.secretName=foodie-tls
```

## Routing architecture

- `https://<your-domain>/` -> frontend service (`ClusterIP`)
- `https://<your-domain>/api` -> backend service (`ClusterIP`)
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
