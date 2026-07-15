# SkillPulse

SkillPulse is a lightweight learning / practice tracker: add skills, log practice sessions (hours, notes), and view simple dashboard metrics. It uses a Go + Gin backend and a static HTML/CSS/JS frontend, and is designed to run locally via Docker Compose or in Kubernetes using the provided manifests.

Links
- Source: https://github.com/apurva-kri/SkillPulse (fork with Kubernetes manifests)
- Original author: LondheShubham153/SkillPulse

Quick summary
- Backend: Go + Gin (REST API)
- Frontend: static HTML/CSS/JS (single-page)
- Data store: MySQL
- Dev / run: Docker Compose (local) or Kubernetes (manifests in kubernetes/k8s)

Stack
- Language(s): Go (backend), HTML/CSS/JavaScript (frontend)
- Framework / runtime: Gin (Go web framework)
- Notable components: MySQL (database), nginx (static serving/reverse-proxy), Docker, Kubernetes manifests (for cluster deployment)

Repository layout (top-level)
```
.env.example            # example env vars
DEPLOYMENT.md           # detailed deployment/runbook
docker-compose.yml      # local compose stack
frontend/               # static frontend (index.html, css/, js/)
backend/                # Go backend (main.go, handlers/, models/, go.mod, Dockerfile)
mysql/                  # MySQL init files / helpers
nginx/                  # nginx configs
terraform/              # optional infra as-code
kubernetes/k8s/         # Kubernetes manifests for namespace, PV, PVC, secrets, configmap, deployments, services
```

How it runs together
- Locally: docker-compose starts MySQL, backend, nginx/frontend and other helpers. Frontend hits the backend REST API under /api.
- In Kubernetes: manifests create a namespace, persistent storage for MySQL, secrets/configmap, a MySQL deployment and init job, backend & frontend deployments and a service that exposes the app. The backend listens on container port 8080; frontend on 80.

Quick start — Docker Compose (local)
1. Clone and prepare env:
   ```bash
   git clone https://github.com/apurva-kri/SkillPulse.git
   cd SkillPulse
   cp .env.example .env
   # edit .env if needed (DB credentials, DOCKERHUB username, etc.)
   ```

2. Start the stack:
   ```bash
   docker-compose up -d
   ```

3. Verify containers and logs:
   ```bash
   docker-compose ps
   docker-compose logs -f backend
   ```

4. Open the frontend in your browser (see docker-compose ports / nginx config; commonly http://localhost/ or localhost:80). Backend API default port is 8080.

Running the backend locally (no Docker)
1. Ensure MySQL is reachable (docker-compose can run it).
2. From repo root:
   ```bash
   cd backend
   go mod download
   go run main.go
   ```
3. Backend listens on the PORT env var (default 8080 if not set).

Environment variables
- See `.env.example` for the main variables used by the compose setup:
  - MYSQL / DB: DB_HOST (db), DB_PORT (3306), DB_USER, DB_PASSWORD, DB_NAME, MYSQL_ROOT_PASSWORD
  - DOCKERHUB / deployment variables and DOCKERHUB username are referenced in DEPLOYMENT.md
- In Kubernetes, secrets.yml and config-map.yml are used to provide these values to pods (see `kubernetes/k8s/secrets.yml` and `kubernetes/k8s/config-map.yml`).

API (examples)
- GET /api/skills           — list skills
- POST /api/skills          — create a skill (JSON body)
- GET /api/skills/:id       — get skill
- DELETE /api/skills/:id    — delete skill
- POST /api/skills/:id/log  — create practice log for a skill
- GET /api/dashboard        — aggregated metrics
- GET /health               — health check

(See backend/handlers for exact request/response shapes.)

Kubernetes deployment (manifests included)
Manifests live at: `kubernetes/k8s/`

Key files:
- kubernetes/k8s/namespace.yml
- kubernetes/k8s/persistent-volume.yml
- kubernetes/k8s/pv-claim.yml
- kubernetes/k8s/secrets.yml
- kubernetes/k8s/config-map.yml
- kubernetes/k8s/mysql-deployment.yml
- kubernetes/k8s/mysql-init.yml (init helper/job)
- kubernetes/k8s/backend-deployment.yml
  - image example in the manifest: `trainwithshubham/skillpulse-backend:latest`
  - containerPort: 8080
- kubernetes/k8s/frontend-deployment.yml
  - image example: `twsdocker1/skillpulse:latest`
  - containerPort: 80
- kubernetes/k8s/service.yml
  - includes services for mysql (port 3306), backend (port 8080) and frontend (NodePort / port 80)

Apply order (recommended)
1. Create namespace
   ```bash
   kubectl apply -f kubernetes/k8s/namespace.yml
   ```

2. Create storage and claims
   ```bash
   kubectl apply -f kubernetes/k8s/persistent-volume.yml
   kubectl apply -f kubernetes/k8s/pv-claim.yml
   ```

3. Create secrets and configmap
   ```bash
   kubectl apply -f kubernetes/k8s/secrets.yml
   kubectl apply -f kubernetes/k8s/config-map.yml
   ```

4. Deploy MySQL and init
   ```bash
   kubectl apply -f kubernetes/k8s/mysql-deployment.yml
   # If there is a mysql-init job / init manifest:
   kubectl apply -f kubernetes/k8s/mysql-init.yml
   ```

5. Deploy backend & frontend
   ```bash
   kubectl apply -f kubernetes/k8s/backend-deployment.yml
   kubectl apply -f kubernetes/k8s/frontend-deployment.yml
   ```

6. Expose services (if not already applied)
   ```bash
   kubectl apply -f kubernetes/k8s/service.yml
   ```

Check status
```bash
kubectl get ns
kubectl get all -n skillpulse
kubectl get pvc -n skillpulse
kubectl logs -l app=backend -n skillpulse
kubectl logs -l app=myssql -n skillpulse   # or mysql pod name
```

Accessing the app in Kubernetes
- If the frontend Service is NodePort, request the node's IP + nodePort. To temporarily access locally via port-forward:
  - Frontend:
    ```bash
    kubectl port-forward svc/frontend 8080:80 -n skillpulse
    # then open http://localhost:8080
    ```
  - Backend:
    ```bash
    kubectl port-forward svc/backend 8080:8080 -n skillpulse
    # API available at http://localhost:8080/api/...
    ```

Notes and tips for Kubernetes
- The manifests include example container images; replace them with your own registry tags if you build/push images (see DEPLOYMENT.md for CI/push recommendations).
- secrets.yml in the k8s folder contains base64-encoded values — ensure they match your intended DB passwords and tokens before applying.
- The mysql-init manifest runs an init container/job to set up the DB; monitor its logs for schema / seed errors.
- If using a cloud provider, replace the PersistentVolume spec with a cloud-backed storage class (or use dynamic provisioner).

Troubleshooting
- Pods CrashLoopBackOff: check `kubectl logs <pod>` and describe `kubectl describe pod <pod>`.
- MySQL connection refused: check service name (the backend expects DB host as `my`/`db` depending

