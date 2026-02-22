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

## Backend not responding or frontend cannot communicate with backend: troubleshooting guide

Use this runbook when the backend pod is `Running` but the backend service does not respond, or when the frontend cannot reach backend APIs.

### 1) Quick diagnosis flow

1. Confirm pod health and restarts.
2. Check backend logs for bind/runtime errors.
3. Verify service selectors and endpoints.
4. Verify backend container port vs service `targetPort`.
5. Validate readiness/liveness probe behavior.
6. Test connectivity from inside the cluster.
7. Inspect ingress and network policy behavior.

### 2) Core kubectl checks

Set variables first:

```bash
NS=foodie-app
APP=foodie-backend
SVC=foodie-backend-service
```

Pod status:

```bash
kubectl get pods -n $NS -l app=$APP -o wide
```

Pod logs (current + previous crash logs if any):

```bash
kubectl logs -n $NS -l app=$APP --tail=200
kubectl logs -n $NS -l app=$APP --previous --tail=200
```

Describe pod (events, probe failures, image/env details):

```bash
POD=$(kubectl get pods -n $NS -l app=$APP -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod -n $NS $POD
```

Describe service and service endpoints:

```bash
kubectl describe svc -n $NS $SVC
kubectl get svc -n $NS $SVC -o yaml
kubectl get endpoints -n $NS $SVC -o wide
kubectl get endpointslice -n $NS -l kubernetes.io/service-name=$SVC -o wide
```

Ingress configuration:

```bash
kubectl get ingress -n $NS
kubectl describe ingress -n $NS
kubectl get ingress -n $NS -o yaml
```

### 3) Verify backend is listening on the expected port

If backend service exposes port `5000`, ensure process inside container is listening there and on `0.0.0.0` (not `127.0.0.1`):

```bash
kubectl exec -n $NS $POD -- sh -c "ss -lntp || netstat -lntp"
kubectl exec -n $NS $POD -- sh -c "printenv | sort"
```

Expected pattern:
- Listening socket includes `:5000`.
- Bind address should be `0.0.0.0:5000` or `[::]:5000`.

### 4) Verify Service `targetPort` matches container `containerPort`

Check deployment container port:

```bash
kubectl get deploy -n $NS -l app=$APP -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{.spec.template.spec.containers[*].name}{": "}{.spec.template.spec.containers[*].ports[*].containerPort}{"\n\n"}{end}'
```

Check service port mapping:

```bash
kubectl get svc -n $NS $SVC -o jsonpath='{.spec.ports[*].port}{" -> targetPort="}{.spec.ports[*].targetPort}{"\n"}'
```

If service `port` is `5000`, `targetPort` must resolve to the same backend listening port (numeric 5000 or correctly named port).

### 5) Check readiness/liveness probes

Probe failures can keep pod `Running` but unavailable behind service endpoints.

```bash
kubectl get deploy -n $NS -l app=$APP -o yaml | sed -n '/readinessProbe:/,/livenessProbe:/p'
kubectl describe pod -n $NS $POD | sed -n '/Readiness/,/Conditions/p'
```

Verify:
- Probe path/port is correct.
- Startup time is sufficient (`initialDelaySeconds`, `timeoutSeconds`).
- HTTP probe path returns success from inside container.

### 6) Test connectivity from inside the cluster

From temporary debug pod:

```bash
kubectl run -n $NS net-debug --rm -it --restart=Never \
  --image=curlimages/curl:8.5.0 -- sh

# inside pod
curl -sv http://$SVC.$NS.svc.cluster.local:5000/health
curl -sv http://$SVC:5000/health
```

Direct-to-pod test (bypass service):

```bash
POD_IP=$(kubectl get pod -n $NS $POD -o jsonpath='{.status.podIP}')
kubectl run -n $NS net-debug --rm -it --restart=Never \
  --image=curlimages/curl:8.5.0 -- sh -c "curl -sv http://$POD_IP:5000/health"
```

Interpretation:
- Pod IP works, service fails -> service selector/port mapping issue.
- Both fail -> app/probe/runtime issue.


### 7) Frontend -> backend communication specific checks

If UI shows states like **"Checking Backend"** forever, backend may be healthy but unreachable from browser route/config. Validate these items:

1. **Frontend calls relative `/api` path (recommended)**

```bash
kubectl get ingress -n $NS -o yaml
```

- Confirm ingress has both routes on same host:
  - `/` -> frontend service
  - `/api` -> backend service
- Prefer frontend API base URL as relative path (`/api`) to avoid cross-origin issues.

2. **If frontend uses absolute backend URL, verify CORS origin exactly matches**

```bash
kubectl describe deploy -n $NS -l app=$APP
kubectl get configmap -n $NS
```

- Backend `CORS_ORIGIN` must match browser origin exactly, including protocol and host:
  - `https://foodie.kitchen` != `http://foodie.kitchen`
  - `https://www.foodie.kitchen` != `https://foodie.kitchen`

3. **Check browser-visible API path through ingress**

```bash
curl -Ik https://<your-domain>/api/health
```

- Expected: HTTP response from backend health endpoint (200/204 depending on app).
- If 404/503, inspect ingress paths, backend service name, and endpoints.

4. **Verify frontend env/config points to correct API URL**

```bash
kubectl describe pod -n $NS $(kubectl get pods -n $NS -l app.kubernetes.io/component=frontend -o jsonpath='{.items[0].metadata.name}')
```

- Confirm frontend runtime/build env for API base URL is correct for production domain.
- Ensure no stale image/config is serving an old backend URL.

### 8) Common-fix checklist

- [ ] **Wrong service selector**
  - `kubectl get svc -n $NS $SVC -o yaml` selector labels must match pod labels.
  - Validate with `kubectl get pods -n $NS --show-labels`.

- [ ] **Port mismatch**
  - `containerPort`, app listen port, and service `targetPort` must align.
  - Named ports in service must exactly match container port name.

- [ ] **App binds to localhost only**
  - App should bind `0.0.0.0`, not `127.0.0.1`.
  - Typical fix: set host env/config to `0.0.0.0`.

- [ ] **NetworkPolicy blocks traffic**
  - Check policies in namespace:
    ```bash
    kubectl get networkpolicy -n $NS
    kubectl describe networkpolicy -n $NS
    ```
  - Ensure ingress to backend from frontend/ingress namespace is explicitly allowed.

- [ ] **Environment variable misconfiguration**
  - Validate required env vars:
    ```bash
    kubectl describe pod -n $NS $POD | sed -n '/Environment:/,/Mounts:/p'
    ```
  - Confirm DB host/port, service URLs, CORS origin, and required secrets/configmaps are populated.

### 9) Fast recovery commands after fixes

```bash
kubectl rollout restart deploy -n $NS -l app=$APP
kubectl rollout status deploy -n $NS -l app=$APP
kubectl get endpoints -n $NS $SVC -w
```

When endpoints populate and readiness is `True`, re-test from frontend and through ingress.
