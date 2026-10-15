# Wanderlust — Helm Deployment

* This repo contains a Helm chart that deploys the Wanderlust app (MongoDB, Redis, backend, frontend) to Kubernetes. It replaces the original `docker-compose.yml` setup for cluster deployments.
* Source Code: https://github.com/DevOps-Playbook/MERN-Stack-Application

<img width="1429" height="660" alt="Screenshot 2026-10-16 at 12 00 03 AM" src="https://github.com/user-attachments/assets/a5feefac-1479-4d54-ab4f-92bacce722ee" />


## Components

| Service  | Image                          | Internal Service Name | Port |
|----------|---------------------------------|------------------------|------|
| mongo    | `mongo:latest`                  | `mongo`                | 27017 |
| redis    | `redis:7.0.5-alpine`            | `redis`                | 6379 |
| backend  | your built/pushed backend image | `backend`               | 5002 → 5000 |
| frontend | your built/pushed frontend image| `frontend`              | 5173 |

Service names match the original Docker container names (`mongo`, `redis`), so connection strings like `mongodb://mongo/wanderlust` and `redis://redis:6379` work unchanged inside the cluster.

## Prerequisites

- A Kubernetes cluster and `kubectl` configured against it
- `helm` v3 installed
- Backend and frontend Docker images built and pushed to a container registry (Helm does not build images from source — unlike `docker-compose build`)

## 1. Configure `values.yaml`

Before installing, set the real image locations:

```yaml
backend:
  image:
    repository: <your-registry>/backend
    tag: <your-tag>

frontend:
  image:
    repository: <your-registry>/frontend
    tag: <your-tag>
```

Environment variables (originally in `backend/.env.docker` and `frontend/.env.docker`) are already populated in `values.yaml`:

```yaml
backend:
  env:
    MONGODB_URI: "mongodb://mongo/wanderlust"
    REDIS_URL: "redis://redis:6379"

frontend:
  env:
    VITE_API_PATH: ""
```

Update these if your environment variables change.

## 2. Install the chart

```bash
helm upgrade --install wanderlust .
```

This creates Deployments and Services for all four components, plus a PersistentVolumeClaim for MongoDB data.

## 3. Verify the deployment

```bash
kubectl get pods
kubectl get svc
```
<img width="1209" height="285" alt="Screenshot 2026-10-16 at 12 00 49 AM" src="https://github.com/user-attachments/assets/62aea1d2-c3da-43aa-a6ac-14e824757796" />


All four pods (`backend`, `frontend`, `mongo`, `redis`) should reach `Running` / `1/1` status within a few seconds to a minute (depending on image pull time).

Check backend logs to confirm it connected to Mongo/Redis:

```bash
kubectl logs -l app=backend
```
<img width="1209" height="285" alt="Screenshot 2026-10-16 at 12 01 16 AM" src="https://github.com/user-attachments/assets/a57db18e-9fbc-4a56-8e9e-af1c2d28a43b" />


## 4. Access the app

Since the Services are `ClusterIP` (internal-only by default), use port-forwarding to reach them from your machine:

```bash
kubectl port-forward svc/frontend 5173:5173
kubectl port-forward svc/backend 5002:5002
```

Then open `http://localhost:5173` in a browser.

**If using Google Cloud Shell:** `localhost` links won't open directly — use Cloud Shell's **Web Preview** button instead, and set the preview port to `5173`.

> To expose the app externally without port-forwarding, set `ingress.enabled: true` in `values.yaml` and configure `ingress.frontendHost` / `ingress.backendHost`.

## 5. Import seed data into MongoDB

Get the current Mongo pod name:

```bash
kubectl get pods -l app=mongo
```

Copy the data file into the pod, then run `mongoimport` inside it:

```bash
kubectl cp ./path/to/sample_posts.json <mongo-pod-name>:/tmp/sample_posts.json

kubectl exec -it <mongo-pod-name> -- mongoimport --db wanderlust --collection posts --file /tmp/sample_posts.json --jsonArray
```

Verify the import:

```bash
kubectl exec -it <mongo-pod-name> -- mongosh wanderlust --eval 'db.posts.countDocuments()'
```

> Note: pod names change on every restart/redeploy (the random suffix). Re-run `kubectl get pods -l app=mongo` to get the current name if these commands fail with "pod not found."

## Troubleshooting

- **`ImagePullBackOff` on backend/frontend** — the image repository in `values.yaml` is still a placeholder or not accessible from the cluster. Confirm the image was pushed and is public/accessible, then `helm upgrade --install wanderlust .` again.
- **`kubectl cp` fails with "doesn't exist in local filesystem"** — the local path given doesn't match where the file actually is. Search for it first:
  ```bash
  find ~ -iname '*sample_posts*' 2>/dev/null
  ```
- **Frontend loads but can't reach backend** — check `VITE_API_PATH` in `values.yaml`; an empty value assumes same-origin requests, which won't work since frontend and backend are separate Services/ports.

## Upgrading

After changing `values.yaml` or chart templates:

```bash
helm upgrade --install wanderlust .
```

## Uninstalling

```bash
helm uninstall wanderlust
```

Note: this does not delete the MongoDB PVC by default — remove it manually if you want to wipe data:

```bash
kubectl delete pvc mongo-data
```
