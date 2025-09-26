# Stage 1 Demo: Manual App Run (no kube-burner)

This walkthrough spins up the frontend + backend on a Kind cluster and lets you click the UI and inspect metrics without kube-burner.

## 1. Build images locally
```bash
export DEMO_REGISTRY=kind.local/demo-app

docker build -t ${DEMO_REGISTRY}-backend:latest  examples/demos/app-load-demo/images/backend
docker build -t ${DEMO_REGISTRY}-frontend:latest examples/demos/app-load-demo/images/frontend
```

## 2. Load images into Kind
```bash
kind create cluster                    # only if the cluster isn’t already up
kind load docker-image ${DEMO_REGISTRY}-backend:latest
kind load docker-image ${DEMO_REGISTRY}-frontend:latest
```

## 3. Create namespace
```bash
kubectl create namespace app-demo-standalone
```

## 4. Apply manifests (files created under your scratch directory)
```bash
kubectl apply -f backend.yaml          # backend deployment + service
kubectl apply -f frontend.yaml         # configmap + frontend deployment + service
```

## 5. Port-forward to access the app and backend
```bash
kubectl port-forward svc/demo-backend  -n app-demo-standalone 8081:8080
kubectl port-forward svc/demo-frontend -n app-demo-standalone 8080:80
```

Open the UI: <http://localhost:8080> and click the buttons.

## 6. Inspect metrics (optional)
```bash
curl http://localhost:8081/metrics | head
```
Look for `app_interactions_total`, `app_active_sessions`, and the HTTP histogram.

## 7. Cleanup after the demo
```bash
kubectl delete namespace app-demo-standalone
```

# Stage 2 Demo: Add Load Generator (still no kube-burner)

This builds on Stage 1. The frontend and backend remain running in the
`app-demo-standalone` namespace. We’ll add the Python load generator to send
exponentially increasing traffic so you can watch the metrics spike.

## Prerequisites
- Stage 1 already deployed and port-forwards in place (UI at http://localhost:8080, backend metrics at http://localhost:8081/metrics)
- Same `DEMO_REGISTRY=kind.local/demo-app` setting as before

## 1. Build and load the load-generator image
```bash
export DEMO_REGISTRY=kind.local/demo-app

docker build -t ${DEMO_REGISTRY}-load:latest examples/demos/app-load-demo/images/load-generator
kind load docker-image ${DEMO_REGISTRY}-load:latest
```

## 2. Apply the load generator manifests
`load-generator.yaml` deploys the client and a service exposing its metrics.
Edit the image name if you use a different registry.
```bash
kubectl apply -f examples/demos/app-load-demo/load-generator.yaml
```

## 3. Watch metrics
- Backend: the `/metrics` endpoint on port 8081 now shows rapidly growing `app_interactions_total`, higher latency buckets, and more active sessions.
- Load generator: port-forward if you want to inspect its metrics.
  ```bash
  kubectl port-forward svc/demo-load -n app-demo-standalone 2112:2112
  curl http://localhost:2112/metrics | head
  ```
  Look for `lg_current_rps`, `lg_sent_requests_total`, `lg_errors_total`.

## 4. Stop the traffic
When finished, delete the load generator (leave the app running if you like).
```bash
kubectl delete -f examples/demos/app-load-demo/load-generator.yaml
```