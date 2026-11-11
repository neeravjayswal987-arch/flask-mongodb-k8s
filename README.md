# Flask + MongoDB on Kubernetes (Minikube)

A Python Flask API backed by an authenticated MongoDB, deployed on Minikube with
a Deployment, StatefulSet, Services, PV/PVC, HPA, and resource limits.

| Endpoint | Method | Description |
|---|---|---|
| `/` | GET | `Welcome to the Flask app! The current time is: <Date and Time>` |
| `/data` | POST | Insert the JSON body into MongoDB |
| `/data` | GET | Return all stored documents |

## 1. Architecture

```
 your machine ──NodePort 30007──► flask-service ──► flask-app pods (2–5, HPA @70% CPU)
                                                          │  mongodb://user:pass@mongodb:27017
                                                          ▼
                                   mongodb (headless Service, cluster-internal only)
                                                          ▼
                                   mongodb-0 (StatefulSet, auth enabled)
                                                          ▼
                                   PVC mongo-pvc ◄──► PV mongo-pv (hostPath /data/mongo)
```

## 2. Repository layout

```
flask-mongodb-app/   app.py, requirements.txt, Dockerfile, .dockerignore
k8s/                 00-secret … 07-flask-hpa (apply with `kubectl apply -f k8s/`)
screenshots/         evidence for the testing section
README.md
```

## 3. Prerequisites

Docker, Minikube, kubectl, a Docker Hub account (or any registry).

```powershell
minikube start --driver=docker --cpus=4 --memory=4096
minikube addons enable metrics-server      # required for the HPA
```

## 4. Build and push the image

```powershell
cd flask-mongodb-app
docker build -t neerav15/flask-mongodb-app:1.0 .
docker login
docker push neerav15/flask-mongodb-app:1.0
cd ..
```

The image name is already set in `k8s/05-flask-deployment.yaml`.

*(Offline alternative: `minikube image load neerav15/flask-mongodb-app:1.0`.)*

## 5. Deploy

Optionally change the password in `k8s/00-secret.yaml` first (letters/digits only).

```powershell
kubectl apply -f k8s/
kubectl get pods,svc,pv,pvc,hpa
kubectl rollout status statefulset/mongodb
kubectl rollout status deployment/flask-app
```

## 6. Access and verify

```powershell
minikube service flask-service --url     # prints e.g. http://127.0.0.1:XXXXX (keep this terminal open)
$URL = "http://127.0.0.1:XXXXX"

Invoke-RestMethod $URL/
Invoke-RestMethod -Method Post -ContentType "application/json" -Body '{"sampleKey":"sampleValue"}' $URL/data
Invoke-RestMethod $URL/data
```

Expected: the welcome message, `status: Data inserted`, and the stored document(s).

![App working in the cluster](screenshots/app-working.png)

## 7. How DNS resolution works inside the cluster

Every cluster has a DNS server (CoreDNS) running in `kube-system`. When a Service
is created, CoreDNS publishes a record for it:

```
<service>.<namespace>.svc.cluster.local
```

Each pod's `/etc/resolv.conf` points at CoreDNS and includes search domains
(`<ns>.svc.cluster.local svc.cluster.local cluster.local`), so a pod in the same
namespace can use the short name **`mongodb`**, which expands to
`mongodb.default.svc.cluster.local`.

- For a normal Service the name resolves to the Service's virtual ClusterIP; kube-proxy then load-balances to a backing pod.
- Our `mongodb` Service is **headless** (`clusterIP: None`), so the name resolves directly to the pod IP(s), and the StatefulSet pod also gets a stable name: `mongodb-0.mongodb.default.svc.cluster.local`.

Flask therefore connects with:

```
mongodb://<user>:<password>@mongodb:27017/flask_db?authSource=admin
```

built in the Deployment from the Secret (`MONGODB_URI`). No pod IPs are hard-coded, so the
connection survives pod restarts and IP changes. Check it yourself:

```powershell
kubectl exec deploy/flask-app -- python -c "import socket; print(socket.gethostbyname('mongodb'))"
```

## 8. Resource requests and limits

- **Request** = what the scheduler *reserves* for the container. A pod is only placed on a node with that much unreserved capacity, and it gives the container a guaranteed baseline. The HPA's CPU percentage is calculated against the request.
- **Limit** = the maximum the container may use. CPU above the limit is *throttled*; memory above the limit gets the container *OOMKilled*. This stops one pod from starving others and keeps the node stable.

Configured for both Flask and MongoDB:

| | CPU | Memory |
|---|---|---|
| Request | 200m (0.2 CPU) | 250M |
| Limit | 500m (0.5 CPU) | 500M |

Because the HPA target (70%) is relative to the 200m request, scaling triggers at
roughly 140m average CPU per pod. MongoDB is started with
`--wiredTigerCacheSizeGB 0.25` so its cache fits under the 500M limit.

## 9. Design choices and alternatives

| Choice | Why | Alternative considered |
|---|---|---|
| MongoDB as a **StatefulSet** | Stable pod identity and storage for a database | Deployment: pods are interchangeable, poor fit for stateful storage |
| `MONGO_INITDB_ROOT_*` env vars from a **Secret** | Official image enables auth automatically on first init; credentials not hardcoded in images/manifests | `--auth` plus init scripts (more moving parts); plain env vars (insecure) |
| **Headless** Service for MongoDB | Cluster-internal only, stable per-pod DNS | ClusterIP also works; NodePort/LoadBalancer rejected because the DB must not be exposed |
| **NodePort** for Flask | Simplest way to reach the app from the local machine on Minikube | LoadBalancer (needs `minikube tunnel`); Ingress (extra addon, overkill here) |
| Static **PV + PVC** (hostPath) | Required by the brief; data survives pod restarts | Dynamic provisioning via `volumeClaimTemplates` (preferable in production) |
| **HPA v2** on CPU, 2–5 replicas | Meets the 70% / min 2 / max 5 requirement | KEDA / custom metrics (unnecessary here) |
| Single MongoDB replica | Keeps local setup simple | Replica set for HA (would be the production choice) |
| `python:3.9-slim`, non-root user | Small image, safer runtime | Alpine (musl issues), full python image (large) |
| Flask dev server | Matches the provided app | Gunicorn for production traffic |

NOTE: Containers run in UTC, so the app shows UTC time, not IST.

## 10. Testing

### 10.1 Database authentication

```powershell
# Without credentials: should fail with "requires authentication"
kubectl exec -it mongodb-0 -- mongosh --quiet --eval "db.getSiblingDB('flask_db').data.find().toArray()"

# With credentials: should return the inserted document
  kubectl exec -it mongodb-0 -- mongosh -u admin -p ChangeMe123 --authenticationDatabase admin --quiet --eval "db.getSiblingDB('flask_db').data.find().toArray()"
```

Result: Without credentials MongoDB rejects the query (`command find requires authentication`). With credentials, the stored documents are returned.

![DB auth test](screenshots/db-auth.png)

### 10.2 Persistence

```powershell
kubectl delete pod mongodb-0          # StatefulSet recreates it
kubectl wait --for=condition=ready pod/mongodb-0 --timeout=120s
Invoke-RestMethod $URL/data                      # earlier data should still be there
```

Result: After deleting `mongodb-0`, the StatefulSet recreated it and `/data` still returned the earlier documents, confirming the PV/PVC keeps the data.

![Persistence test](screenshots/persistence.png)

### 10.3 Autoscaling (high-traffic simulation)

Terminal 1 – watch the HPA:
```powershell
kubectl get hpa flask-hpa -w
```
Terminal 2 – watch pods:
```powershell
kubectl get pods -l app=flask-app -w
```
Terminal 3 – generate load (start 3–4 generators; add more if CPU stays low):
```powershell
  1..8 | ForEach-Object { kubectl run load-generator-$_ --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://flask-service:5000/ >/dev/null; done" }
```
Stop the load and watch the scale-down (takes ~5 min stabilization window):
```powershell
  1..8 | ForEach-Object { kubectl delete pod load-generator-$_ --ignore-not-found }
```

**Results:**

Load was generated by 8 `busybox` pods (`wget` loop against `flask-service`).

| Stage | CPU % / target | Replicas | Notes |
|---|---|---|---|
| Baseline | 1% / 70% | 2 | no load |
| Load started | 59% / 70% | 2 | CPU rising |
| Scale-out | 95% / 70% | 2 → 4 | HPA scaled out |
| Under load | 67% / 70% | 4 | highest replica count reached (max allowed 5) |
| After load stopped | 9% / 70% | 2 | scaled back down after ~5 min |

Screenshots:

![HPA before load](screenshots/hpa-before.png)
![HPA scaling up](screenshots/hpa-scaling-up.png)
![Pods scaled out under load](screenshots/pods-scaled.png)
![HPA scaling down](screenshots/hpa-scaling-down.png)


### 10.4 Issues encountered

1. **App time shown in UTC instead of IST.**
   When running locally (Part 1) the page showed IST, but inside Kubernetes it showed a time about 5 hours 30 minutes behind. I confirmed with `kubectl exec deploy/flask-app -- date` that containers run in UTC by default, so the app was correct and showing the container's clock. I kept UTC because servers conventionally run in UTC. If IST is needed, the app can use an explicit timezone (`datetime.now(timezone(timedelta(hours=5, minutes=30)))`).

2. **`Unable to connect to the remote server` when calling the service.**
   On Windows with the Docker driver, `minikube service flask-service --url` creates a tunnel that only works while its terminal stays open, and it prints a new port every time it is restarted. After closing the window, requests to the old port failed. Fix: re-run `minikube service flask-service --url`, keep that terminal open, and set `$URL` to the newly printed address.

3. **PowerShell syntax differences.**
   `curl` in PowerShell is an alias for `Invoke-WebRequest` and does not accept the Linux-style flags, so I used `Invoke-RestMethod` for the requests. Also, two commands typed on one line caused a parser error, so they were run on separate lines (or separated with `;`).

4. **`Authentication failed` in the first authenticated MongoDB test.**
   My first attempt used a wrong password in the `mongosh` command, so MongoDB rejected it (the no-credentials query correctly failed with `command find requires authentication`). I confirmed the credentials stored in the Kubernetes Secret (`mongo-secret`) and re-ran the command with the correct password, which returned the stored documents. This also confirmed that authentication is genuinely enforced.
   
- 

### 10.5 Troubleshooting guide (common problems and fixes)

These did not occur during my run, but are the usual failure points for this setup:

- **HPA shows `<unknown>/70%`**: metrics-server is not enabled or not ready yet. Run `minikube addons enable metrics-server` and wait about a minute.
- **`ImagePullBackOff`**: wrong image name, or the image was not pushed to the registry or loaded into Minikube.
- **Flask `Authentication failed`**: MongoDB only creates its root user when the data directory is empty on first start. Delete the StatefulSet, PVC, PV and `/data/mongo` on the node, then redeploy.
- **No scale-up under load**: the load is too light, so start more load generators.


## 11. Cleanup

```powershell
kubectl delete -f k8s/      # removes all Kubernetes resources
minikube stop               # stops the cluster
```

Note: the PV uses `persistentVolumeReclaimPolicy: Retain`, so the data under `/data/mongo`
on the Minikube node survives this cleanup. To wipe it completely, use
`minikube ssh "sudo rm -rf /data/mongo"` or `minikube delete`.

## Part 1 cookie point: why use a Python virtual environment?

- **Dependency isolation**: each project gets its own package versions (no conflicts between projects).
- **Reproducibility**: `requirements.txt` captures exactly what the app needs.
- **No system pollution / no admin rights needed**: the OS Python stays clean.
- **Easy clean-up and portability**: delete the folder to reset; the same setup works on any machine or in CI.
