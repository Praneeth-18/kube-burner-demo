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
