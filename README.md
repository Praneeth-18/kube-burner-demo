# kube-burner Orchestration

This guide walks through the demo end to end: building the sample application, deploying it with kube-burner, exercising it with an exponential load generator, and observing metrics via direct port-forwards. 

---
## 1. Prerequisites

- Docker Desktop (or Docker Engine) with permission to share this repository directory.
- `kubectl` and either **kind** (recommended) or another Kubernetes cluster.
- `curl`, `tar`, and `bash` for pulling binaries.
- Clone the repo:
  ```bash
  git clone https://github.com/Praneeth-18/kube-burner-demo.git
  cd kube-burner-demo
  ```

---
## 2. Build the demo containers

```bash
docker build -t kind.local/demo-app-backend:latest examples/demos/app-load-demo/images/backend

docker build -t kind.local/demo-app-frontend:latest examples/demos/app-load-demo/images/frontend

docker build -t kind.local/demo-app-load:latest examples/demos/app-load-demo/images/load-generator
```

Change the tags if you are using a different cluster/registry.

---
## 3. Create and prepare the cluster

For kind:

```bash
kind create cluster --name kube-burner-demo
kind load docker-image kind.local/demo-app-backend:latest --name kube-burner-demo
kind load docker-image kind.local/demo-app-frontend:latest --name kube-burner-demo
kind load docker-image kind.local/demo-app-load:latest --name kube-burner-demo
kubectl get nodes
```

Skip the `kind load` step if your cluster already has access to these images.

---
## 4. Install kube-burner

Download the binary matching your platform. Example (macOS arm64):

```bash
mkdir -p bin
curl -L https://github.com/kube-burner/kube-burner/releases/latest/download/kube-burner-darwin-arm64 -o bin/kube-burner
chmod +x bin/kube-burner
```

Linux amd64:

```bash
curl -L https://github.com/kube-burner/kube-burner/releases/latest/download/kube-burner-linux-amd64 -o bin/kube-burner
chmod +x bin/kube-burner
```

You can also build from source:

```bash
go install github.com/kube-burner/kube-burner/cmd/kube-burner@latest
# copy or symlink $GOPATH/bin/kube-burner into ./bin/
```

`./bin/` is ignored by git so you can drop the binary there safely.

---
## 5. Configure the run

Edit `examples/demos/app-load-demo/tmp/demo-user-data.yaml`—this file controls names, image tags, replica counts, load profile, and the UUID. Example:

```yaml
uuid: demo-run-001
namespacePrefix: app-demo
appName: movie-night
backendImage: kind.local/demo-app-backend:latest
frontendImage: kind.local/demo-app-frontend:latest
loadGeneratorImage: kind.local/demo-app-load:latest
baselinePause: 60s
loadPause: 180s
loadGeneratorBaseRps: "4"
loadGeneratorRampFactor: "2"
loadGeneratorRampIntervalSeconds: "30"
loadGeneratorRunDurationSeconds: "240"
```

Remember to change the `uuid` before each run; the namespace will be `${namespacePrefix}-${uuid}` (e.g. `app-demo-demo-run-001`).

---
## 6. Run the demo

```bash
./bin/kube-burner init \
  -c examples/demos/app-load-demo/config/demo-run.yml \
  --user-data examples/demos/app-load-demo/tmp/demo-user-data.yaml \
  --uuid demo-run-001
```

kube-burner workflow:
1. Pre-pull images via a temporary DaemonSet.
2. Deploy backend and frontend.
3. Pause for the baseline window (`baselinePause`).
4. Deploy the load generator; it drives traffic until `loadPause` elapses (or `RUN_DURATION_SECONDS` is hit).
5. Delete the load generator Deployment/Service (backend/front-end remain).

---
## 7. Observe via port-forward

While the namespace exists:

- **Frontend UI**
  ```bash
  kubectl port-forward svc/demo-frontend -n app-demo-demo-run-001 8080:80
  ```
  Visit <http://localhost:8080/> to watch counters update.

- **Backend metrics**
  ```bash
  kubectl port-forward svc/demo-backend -n app-demo-demo-run-001 8081:8080
  curl http://localhost:8081/metrics | grep app_interactions_total
  ```

- **Load generator metrics (during load window)**
  ```bash
  kubectl port-forward svc/demo-load -n app-demo-demo-run-001 2112:2112
  curl http://localhost:2112/metrics | grep lg_
  ```

These endpoints expose everything you need: HTTP server metrics, interaction counts, and load-generator gauges/counters.

---
## 8. Cleanup

```bash
./bin/kube-burner destroy --uuid demo-run-001
kubectl delete namespace app-demo-demo-run-001 --ignore-not-found
```

Optional:

```bash
kind delete cluster --name kube-burner-demo
```

Before the next run, update the `uuid` in `demo-user-data.yaml`.

---
## 9. Troubleshooting highlights

- **Pods cannot pull images**: ensure you loaded the images into kind or pushed them to a reachable registry.
- **Load ends too soon**: increase `loadPause` and/or `loadGeneratorRunDurationSeconds`.
- **UUID conflict**: destroy the old run or pick a fresh UUID.