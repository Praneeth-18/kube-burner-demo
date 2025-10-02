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
docker build -t kind.local/demo-app-backend:latest images/backend

docker build -t kind.local/demo-app-frontend:latest images/frontend

docker build -t kind.local/demo-app-load:latest images/load-generator
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

Download the binary matching your platform. 

(OR) 

Simply run "make build" in the root of the cloned [kube-burner](https://github.com/kube-burner/kube-burner) and copy the "bin" folder to this repo's root.

---
## 5. Configure the run

Edit `tmp/demo-user-data.yaml`—this file controls names, image tags, replica counts, load profile, and the UUID. Every field can be tweaked so you can dial in lighter or heavier runs without touching the config. Example:

```yaml
uuid: demo-run-001
namespacePrefix: app-demo
appName: movie-night
backendName: demo-backend
backendServiceName: demo-backend
backendReplicas: 2
frontendName: demo-frontend
frontendServiceName: demo-frontend
frontendReplicas: 1
loadGeneratorName: demo-load
loadGeneratorServiceName: demo-load
loadGeneratorReplicas: 1
backendImage: kind.local/demo-app-backend:latest
frontendImage: kind.local/demo-app-frontend:latest
loadGeneratorImage: kind.local/demo-app-load:latest
frontendPublicUrl: http://localhost:8081
loadGeneratorActions: book_ticket,cancel_ticket,give_feedback
enableLoad: true
baselinePause: 90s
loadPause: 180s
loadGeneratorBaseRps: "4"
loadGeneratorRampFactor: "2"
loadGeneratorRampIntervalSeconds: "30"
loadGeneratorRunDurationSeconds: "240"
prometheusName: demo-prometheus
prometheusServiceName: demo-prometheus
prometheusImage: prom/prometheus:v2.54.0
```

Remember to change the `uuid` before each run; the namespace will be `${namespacePrefix}-${uuid}` (e.g. `app-demo-demo-run-001`).

### Ramp pattern (You can tweak these as per your desired load numers in `tmp/demo-user-data.yaml`
- Starts at `BASE_RPS` requests per second.
- Every `RAMP_INTERVAL_SECONDS`, multiplies the rate by `RAMP_FACTOR`.
  * Example with defaults (`BASE_RPS=2`, `RAMP_FACTOR=1.35`, `RAMP_INTERVAL_SECONDS=45`):
    - 0 seconds: ~2 rps (2 × 1.35^0)
    - 45 seconds: ~2.7 rps (2 × 1.35^1)
    - 90 seconds: ~3.6 rps (2 × 1.35^2)
    - 135 seconds: ~4.9 rps (2 × 1.35^3)
    - 180 seconds: ~6.6 rps (2 × 1.35^4)

### Key knobs worth adjusting:
- **backendName/backendServiceName/backendReplicas**: override the backend Deployment & Service names and replica count without touching the templates.
- **frontendName/frontendServiceName/frontendReplicas**: same for the frontend.
- **loadGeneratorName/loadGeneratorServiceName/loadGeneratorReplicas**: control the load generator resources and how many workers run in parallel.
- **frontendPublicUrl**: injected into the frontend config so the browser hits the right backend endpoint.
- **loadGeneratorActions**: comma-separated list of actions the Python client will randomly execute.
- **enableLoad**: set to `false` if you want to deploy only the app stack (useful for manual demos).
- **baselinePause/loadPause**: how long kube-burner waits between jobs—baseline lets you click manually; load keeps the generator running before you stop it.
- **loadGeneratorBaseRps/RampFactor/RampInterval/RunDuration**: tune the exponential request curve. These are the levers for targeting different load levels or demonstrating a breaking point.
- **prometheusName/prometheusServiceName/prometheusImage**: tweak the in-cluster Prometheus deployment if you want a different name or image tag.

> Tip: keep multiple copies of the file (for example `demo-user-data-light.yaml`, `demo-user-data-stress.yaml`) and copy the one you want to `tmp/demo-user-data.yaml` before each run.

---
## 6. Run the demo

```bash
./bin/kube-burner init \
  -c config/demo-run.yml \
  --user-data tmp/demo-user-data.yaml \
  --uuid demo-run-001
```

kube-burner workflow:
1. Pre-pull images via a temporary DaemonSet.
2. Deploy backend and frontend.
3. Pause for the baseline window (`baselinePause`).
4. Deploy the load generator; it drives traffic until `loadPause` elapses (or `RUN_DURATION_SECONDS` is hit). Afterwards the pods stay up so dashboards remain accessible until you scale them down.


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

- **Prometheus UI**
  ```bash
  kubectl port-forward svc/demo-prometheus -n app-demo-demo-run-001 9090:9090
  ```
  Browse <http://localhost:9090> to explore the baked-in dashboards or run ad-hoc queries to interpret the metrics (e.g., **app_interactions_total, app_active_sessions, http_request_duration_seconds_bucket, lg_sent_requests_total**).

##### OPTIONAL:

- **Load generator metrics (during load window)**
  ```bash
  kubectl port-forward svc/demo-load -n app-demo-demo-run-001 2112:2112
  curl http://localhost:2112/metrics | grep lg_
  ```

These endpoints expose everything you need: HTTP server metrics, interaction counts, load-generator gauges/counters, and the in-cluster Prometheus scrape results.

## When you’re done showing the spike, scale the load generator back to zero so the app stays up but traffic stops:

```bash
kubectl scale deployment demo-load -n app-demo-demo-run-001 --replicas=0
```

#### You can scale it back up later if you want to restart the load without rerunning kube-burner.
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
