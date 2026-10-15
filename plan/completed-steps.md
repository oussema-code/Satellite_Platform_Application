# 🚀 Satellite Platform - Hands-On DevOps Guide & Runbook

This guide contains the exact step-by-step commands executed to take the Satellite Platform Application from bare source code to a hardened, zero-cost, local Kubernetes production architecture.

---

## 📑 Table of Contents
1. [Phase 1: Local Containerization & Health Verification](#phase-1-local-containerization--health-verification)
2. [Phase 2: Local Kubernetes (k3s) & Ingress Controller Setup](#phase-2-local-kubernetes-k3s--ingress-controller-setup)
3. [Phase 3: Database & Messaging Tier (MongoDB, Redis, RabbitMQ)](#phase-3-database--messaging-tier)
4. [Phase 4: Deploying Spring Boot Backend into k3s](#phase-4-deploying-spring-boot-backend-into-k3s)
5. [Phase 5: Native Jenkins CI Pipeline & DevSecOps (Trivy)](#phase-5-native-jenkins-ci-pipeline--devsecops-trivy)
6. [Phase 5.5: Full Microservices Containerization & Optimization](#phase-55-full-microservices-containerization--optimization)
7. [Phase 6: GitOps Continuous Delivery with ArgoCD](#phase-6-gitops-continuous-delivery-with-argocd)
8. [Phase 7: Drift Detection & Automated Self-Healing Verification](#phase-7-drift-detection--automated-self-healing-verification)
9. [Debugging Playbook: Commands, Gotchas & Diagnostic Flowchart](#debugging-playbook-commands-gotchas--diagnostic-flowchart)
10. [Cluster Health Check (Verify Everything)](#cluster-health-check-verify-everything)

---

## Phase 1: Local Containerization & Health Verification

### Step 1.1: Parameterize Environment Variables
Never commit credentials. Create your `.env` from the template:
```bash
cp .env.example .env
```
Ensure it is ignored by Git:
```bash
git status --ignored | grep .env || echo ".env" >> .gitignore
```

### Step 1.2: Compile the Spring Boot JAR
Compile the JAR on the host machine using Maven:
```bash
cd Backend
mvn clean package -DskipTests
cd ..
```

### Step 1.3: Build the Hardened Docker Image
Build the container with Eclipse Temurin JRE 17 Alpine and non-root user `spring`:
```bash
docker build -t satellite-backend:latest -f Backend/Dockerfile Backend/
```

### Step 1.4: Verify Full Stack Locally with Docker Compose
Start the health-gated stack:
```bash
docker compose up -d
```
Check that all containers are healthy:
```bash
docker compose ps
```
Verify Spring Boot Actuator:
```bash
curl http://localhost:8080/actuator/health
# Expected: {"status":"UP"}
```
Stop the Compose stack to free ports (8080, 27017, 6379, 5672) for Kubernetes:
```bash
docker compose down
```

---

## Phase 2: Local Kubernetes (k3s) & Ingress Controller Setup

### Step 2.1: Install k3s with Traefik Disabled
Disable Traefik to use the industry-standard Ingress-NGINX:
```bash
sudo mkdir -p /etc/rancher/k3s
echo -e "disable:\n  - traefik" | sudo tee /etc/rancher/k3s/config.yaml
curl -sfL https://get.k3s.io | sh -
```

### Step 2.2: Configure Non-Root `kubectl` Access
Set up kubeconfig for your Linux user:
```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
chmod 600 ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc
```
Verify the cluster node:
```bash
kubectl get nodes
# Expected: Ready
```

### Step 2.3: Install Helm 3 CLI
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Step 2.4: Deploy Official Ingress-NGINX Controller
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```
Verify the ingress controller pod and ports (80/443):
```bash
kubectl get pods,svc -n ingress-nginx
```

---

## Phase 3: Database & Messaging Tier

### Step 3.1: Create Application Namespace
```bash
kubectl create namespace satellite-platform
```

### Step 3.2: Deploy MongoDB via Bitnami Helm
Deploy MongoDB with 5Gi persistent storage on k3s `local-path`:
```bash
set -a; source .env; set +a
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm install mongodb bitnami/mongodb \
  --namespace satellite-platform \
  --set auth.rootPassword="${MONGO_ROOT_PASSWORD}" \
  --set auth.usernames[0]="${MONGO_ROOT_USER}" \
  --set auth.passwords[0]="${MONGO_ROOT_PASSWORD}" \
  --set auth.databases[0]="${MONGO_DATABASE}" \
  --set persistence.size=5Gi
```

### Step 3.3: Deploy Redis Standalone via Bitnami Helm
```bash
helm install redis bitnami/redis \
  --namespace satellite-platform \
  --set architecture=standalone \
  --set auth.enabled=false \
  --set master.persistence.size=2Gi
```

### Step 3.4: Deploy RabbitMQ via Native Kubernetes Manifest
*(Note: Bitnami Helm rabbitmq chart paywalled its Debian tags causing `Init:ErrImagePull`. We use a native StatefulSet manifest with `rabbitmq:3-management-alpine`)*

1. Delete any orphaned PVC if uninstalled from Bitnami:
   ```bash
   kubectl delete pvc data-rabbitmq-0 -n satellite-platform --ignore-not-found
   ```
2. Create the Secret from `.env`:
   ```bash
   set -a; source .env; set +a
   kubectl create secret generic rabbitmq-secret \
     --namespace satellite-platform \
     --from-literal=rabbitmq-user="${RABBITMQ_USER}" \
     --from-literal=rabbitmq-password="${RABBITMQ_PASS}" \
     --dry-run=client -o yaml | kubectl apply -f -
   ```
3. Apply the RabbitMQ manifest:
   ```bash
   kubectl apply -f k8s/infra/rabbitmq.yaml
   ```

### Step 3.5: Verify Infrastructure
```bash
kubectl get pods,pvc,svc -n satellite-platform
```
*Expected:* `mongodb`, `redis-master-0`, and `rabbitmq-0` are all `Running (1/1)` and their PVCs are `Bound`.

---

## Phase 4: Deploying Spring Boot Backend into k3s

### Step 4.1: Import the Local Docker Image into k3s Containerd
k3s uses containerd (`k3s ctr`), not the Docker daemon. Import the locally built image:
```bash
docker tag satellite_platform_application-backend:latest satellite-backend:latest
docker save satellite-backend:latest | sudo k3s ctr images import -
```

### Step 4.2: Create the Backend Kubernetes Secret
Dynamically generate the secret from `.env` (using `authSource=satellitedb` so MongoDB authenticates user `admin` on `satellitedb`):
```bash
set -a; source .env; set +a
kubectl create secret generic backend-secret \
  --namespace satellite-platform \
  --from-literal=MONGO_URI="mongodb://${MONGO_ROOT_USER}:${MONGO_ROOT_PASSWORD}@mongodb:27017/${MONGO_DATABASE}?authSource=${MONGO_DATABASE}" \
  --from-literal=RABBITMQ_USER="${RABBITMQ_USER}" \
  --from-literal=RABBITMQ_PASS="${RABBITMQ_PASS}" \
  --from-literal=JWT_SECRET="${JWT_SECRET}" \
  --from-literal=ADMIN_PASSWORD="${ADMIN_PASSWORD}" \
  --dry-run=client -o yaml | kubectl apply -f -
```

### Step 4.3: Apply Backend Manifests
Deploy the PVC, ConfigMap, Deployment, Service, and Ingress:
```bash
kubectl apply -f k8s/backend/
```

### Step 4.4: Watch the Pod Rollout
```bash
kubectl get pods -n satellite-platform -w
```
*Expected:* Pod `satellite-backend-...` transitions to `Running (1/1)` with 0 restarts.

### Step 4.5: Test Public Access through Ingress-NGINX
Test via port 80 using the Host header:
```bash
curl -H "Host: api.satellite.localhost" http://localhost/actuator/health
```
*Expected Output:*
```json
{"status":"UP","groups":["liveness","readiness"]}
```

---

## Phase 5: Native Jenkins CI Pipeline & DevSecOps (Trivy)

### Step 5.1: Install Native Jenkins on Ubuntu 24.04 (Noble)
To avoid Docker-in-Docker socket translation complexity, Jenkins runs natively as a systemd service:
```bash
# 1. Fetch current 2026 GPG signing key (Ubuntu 24.04 rejects expired 2023 keys)
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian/jenkins.io-2026.key

# 2. Add the official LTS repository entry
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# 3. Install OpenJDK 21, Maven, and Jenkins
sudo apt-get update
sudo apt-get install -y openjdk-21-jdk maven jenkins
```

### Step 5.2: Configure Linux Permissions (Docker Socket Access)
Grant the `jenkins` system user access to the host Docker daemon:
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```
*Note:* Systemd services do NOT inherit newly added Linux groups until the service is restarted.

### Step 5.3: Install Trivy Security Scanner
```bash
sudo apt-get install -y wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update && sudo apt-get install -y trivy
```

### Step 5.4: Seed Trivy Vulnerability Databases
Trivy downloads its general CVE DB (~120MB) and Java/Maven index (~935MB) to scan dependencies completely offline:
```bash
sudo mkdir -p /var/lib/jenkins/.cache
sudo cp -r ~/.cache/trivy /var/lib/jenkins/.cache/
sudo chown -R jenkins:jenkins /var/lib/jenkins/.cache/trivy
```

### Step 5.5: Create Enterprise Declarative `Jenkinsfile`
The pipeline is committed directly to Git (`branch: oussema`) and configured in Jenkins UI via "Pipeline script from SCM":
* **Stage 1: Pre-Flight Tooling Validation:** Validates Java, Maven, Docker, and Trivy versions.
* **Stage 2: Build & Test (Maven):** `mvn clean package -DskipTests` inside `Backend/`.
* **Stage 3: Source Security Audit (Trivy FS):** Scans `Backend/pom.xml` dependencies and text files for secrets.
* **Stage 4: Build Docker Image:** `docker build -t satellite-backend:${BUILD_NUMBER} -t satellite-backend:latest`.
* **Stage 5: Container Vulnerability Scan (Trivy Image):** Layer-by-layer audit of the container with `--timeout 15m`.
* **Stage 6: Artifact Verification:** Confirms image registered in the local Docker daemon.

---

## Phase 5.5: Full Microservices Containerization & Optimization

### Step 5.5.1: Containerize GEE Service (`gee_app_with_cache_logic`)
1. **Hardened Dockerfile (`gee_app_with_cache_logic/Dockerfile`):**
   * Base image: `python:3.12-slim` (minimal attack surface).
   * Created non-root Linux user `appuser:10001`.
   * Installed dependencies in a virtual environment (`/opt/venv`).
   * Production WSGI/ASGI entrypoint: `uvicorn gee_app:create_app --factory --host 0.0.0.0 --port 5000`.
2. **Security `.dockerignore`:**
   * Excluded `.git`, `__pycache__`, local `.env`, and critically excluded Google Cloud service account keys (`*.json`) to prevent credential leakage in Docker images.
3. **Health Endpoint:**
   * Added `/health` probe route in `gee_app/routes/api_routes.py`.
4. **Kubernetes Manifests (`k8s/gee/`):**
   * `configmap.yaml`: Configured `PORT: "5000"`, `REDIS_HOST: "redis-master"`, `CACHE_EXPIRY: "86400"`.
   * `pvc.yaml`: 2Gi `local-path` persistent volume mounted at `/app/cache`.
   * `deployment.yaml` & `service.yaml`: ClusterIP service exposing port 5000 with liveness and readiness probes.

### Step 5.5.2: Containerize Image Processor (`image_porcessing`)
1. **Hardened Dockerfile (`image_porcessing/Dockerfile`):**
   * Base image: `python:3.12-slim`.
   * Replaced heavy GUI OpenCV package with `opencv-python-headless` in `requirements.txt` to eliminate missing X11/GL library crashes.
   * Runs as non-root user `appuser:10002`.
   * Entrypoint: `uvicorn main:app --host 0.0.0.0 --port 8000`.
2. **Health Endpoint:**
   * Added `/health` endpoint in `main.py` returning `{"status": "UP", "service": "image-processor"}`.
3. **Kubernetes Manifests (`k8s/image-processor/`):**
   * `pvc.yaml`: 2Gi persistent volume mounted at `/app/storage`.
   * `deployment.yaml` & `service.yaml`: ClusterIP service exposing port 8000 with liveness and readiness probes.

### Step 5.5.3: Containerize Next.js Frontend (`FrontEnd`)
1. **Multi-Stage Dockerfile (`FrontEnd/Dockerfile`):**
   * Stage 1 (`deps`): Node 20 Alpine, caching `package-lock.json` with fallback `npm ci || npm install`.
   * Stage 2 (`builder`): Inlines `NEXT_PUBLIC_API_URL=http://api.satellite.localhost` and runs `npm run build` with `output: 'standalone'`.
   * Stage 3 (`runner`): Minimal Alpine runner executing `node server.js` as non-root user `nextjs:1001` on port 3000.
2. **Health Route:**
   * Created Next.js Route Handler at `src/app/api/health/route.ts` returning HTTP 200 `{"status": "UP"}`.
3. **Kubernetes Manifests (`k8s/frontend/`):**
   * `deployment.yaml` & `service.yaml`: ClusterIP service exposing port 3000.
   * `ingress.yaml`: Ingress-NGINX rule routing `Host: satellite.localhost` to `satellite-frontend:3000`.

### Step 5.5.4: Inter-Service Mesh Wiring & Local Registry Import
1. **Updated Backend ConfigMap (`k8s/backend/configmap.yaml`):**
   * `PYTHON_BASE_URL`: `http://gee-service:5000`
   * `APP1_URL`: `http://image-processor:8000`
2. **Import Docker Images into k3s containerd:**
   ```bash
   docker save satellite-frontend:latest | sudo k3s ctr images import -
   docker save satellite-gee:latest | sudo k3s ctr images import -
   docker save satellite-image-processor:latest | sudo k3s ctr images import -
   ```

---

## Phase 6: GitOps Continuous Delivery with ArgoCD

### Step 6.1: Deploy ArgoCD into k3s
Deployed the official ArgoCD control plane into its own dedicated namespace:
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Step 6.2: Retrieve Admin Credentials & Access Web UI
Decoded the auto-generated initial password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```
Started local port-forwarding to access the dashboard securely:
```bash
kubectl port-forward svc/argocd-server -n argocd 8085:443
# Access Web UI at: https://localhost:8085 (Username: admin)
```

### Step 6.3: Configure Forked Git Remote
Renamed original remote to `upstream` and configured personal fork as `origin`:
```bash
git remote rename origin upstream
git remote add origin https://github.com/oussema-code/Satellite_Platform_Application.git
git push -u origin oussema
```

### Step 6.4: Declarative Application Manifest (`argocd/satellite-platform-application.yaml`)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: satellite-platform
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/oussema-code/Satellite_Platform_Application.git
    targetRevision: oussema
    path: k8s
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: satellite-platform
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Applied to Kubernetes:
```bash
kubectl apply -f argocd/satellite-platform-application.yaml
```

---

## Phase 7: Drift Detection & Automated Self-Healing Verification

### Step 7.1: Verify Initial GitOps Sync
```bash
kubectl get applications -n argocd
# Output:
# NAME                 SYNC STATUS   HEALTH STATUS
# satellite-platform   Synced        Healthy
```

### Step 7.2: Drift Detection & Self-Healing Live Test
1. **Tamper with Cluster State:**
   Manually scale frontend deployment outside of Git:
   ```bash
   kubectl scale deployment satellite-frontend --replicas=3 -n satellite-platform
   ```
2. **Verify Instant Reconciliation:**
   Inspect deployment events:
   ```bash
   kubectl describe deployment satellite-frontend -n satellite-platform | tail -n 10
   ```
   * Event 1: `deployment-controller` scaled up replica set from 1 to 3.
   * Event 2 (at the exact same second): ArgoCD detected `OutOfSync` against Git (`replicas: 1`) and scaled it back down from 3 to 1!
3. **Inspect ArgoCD Audit Trail:**
   ```bash
   kubectl get events -n argocd --sort-by='.metadata.creationTimestamp' | tail -n 6
   ```
   Audit confirmed: `Synced -> OutOfSync` followed by `OutOfSync -> Synced` in **under 1 second**.

---

## Debugging Playbook: Commands, Gotchas & Diagnostic Flowchart

### 🧭 The 4-Step Diagnostic Flowchart (Edge to Core)

When diagnosing any Kubernetes or ArgoCD failure, follow this exact sequence:

```
[ Step 1: High-Level State ]
  kubectl describe application satellite-platform -n argocd
      │
      ├─► Message indicates in-cluster DNS error?
      │     ▼
      │   [ Step 2: Check CoreDNS ]
      │     kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
      │     ▼
      │   [ Step 3: Check Host Network Uplink ]
      │     resolvectl status | grep -E "DNS Servers|Current DNS Server"
      │
      └─► Message indicates API connection timeout (10.43.0.1:443)?
            ▼
          [ Step 4: Check Kubernetes Internal Endpoints ]
            kubectl get endpoints kubernetes
            ip -4 addr show scope global
```

---

### 🛠️ The 4 Standard Commands to Debug Any Pod:

1. **View live container logs:**
   ```bash
   kubectl logs -l app=satellite-backend -n satellite-platform --tail=50
   ```
2. **View logs from crashed container (`CrashLoopBackOff`):**
   ```bash
   kubectl logs -l app=satellite-backend -n satellite-platform --previous --tail=50
   ```
3. **View lifecycle events, probes, and mount errors:**
   ```bash
   kubectl describe pod -l app=satellite-backend -n satellite-platform
   ```
4. **Trigger a clean rolling restart:**
   ```bash
   kubectl rollout restart deployment/satellite-backend -n satellite-platform
   ```

---

### ⚠️ Gotchas & How We Fixed Them:

#### Gotcha 1: Spring Boot `Permission denied: logs/app.log`
* **Log:** `java.io.FileNotFoundException: logs/app.log (Permission denied)`
* **Why:** Alpine created user `spring` with UID **100**, but Kubernetes `deployment.yaml` had `runAsUser: 1001`.
* **Fix Command:** Update `k8s/backend/deployment.yaml` with `runAsUser: 100`, `runAsGroup: 101`, and mount `emptyDir` scratch volumes at `/app/logs` and `/app/tmp`:
  ```bash
  kubectl apply -f k8s/backend/deployment.yaml
  ```

#### Gotcha 2: MongoDB `Authentication failed (Error 18)`
* **Log:** `com.mongodb.MongoCommandException: Command failed with error 18 (AuthenticationFailed)`
* **Why:** Bitnami created user `admin` in database `satellitedb`, but the connection string asked MongoDB to check system database `admin` (`authSource=admin`).
* **Fix Command:** Recreate the secret with `authSource=satellitedb`:
  ```bash
  kubectl create secret generic backend-secret \
    --namespace satellite-platform \
    --from-literal=MONGO_URI="mongodb://admin:admin123@mongodb:27017/satellitedb?authSource=satellitedb" \
    --dry-run=client -o yaml | kubectl apply -f -
  kubectl rollout restart deployment/satellite-backend -n satellite-platform
  ```

#### Gotcha 3: GEE Logging `Permission denied: /app.log`
* **Log:** `PermissionError: [Errno 13] Permission denied: '/app.log'`
* **Why:** Flask code used `os.path.join(app.root_path, '..', 'app.log')` which resolved to container filesystem root `/app.log`. Since the container runs as non-root `appuser:10001`, Linux denied write access.
* **Fix:** Configured logger to write to `/app/app.log` (owned by `appuser`) and added a fallback `StreamHandler(sys.stdout)`.

#### Gotcha 4: Laptop WiFi IP Drift & Kubernetes API Timeout (`10.43.0.1:443`)
* **Log:** `dial tcp 10.43.0.1:443: connect: no route to host` / `i/o timeout`
* **Why:** Laptop changed WiFi networks (e.g. from `192.168.0.146` to `192.168.68.32`). The internal virtual service `kubernetes` (`10.43.0.1:443`) was still forwarding packets to the old node IP (`192.168.0.146:6443`).
* **Fix:**
  ```bash
  sudo systemctl restart k3s
  kubectl rollout restart statefulset/argocd-application-controller -n argocd
  ```

#### Gotcha 5: CoreDNS Upstream DNS Cache Staleness (`10.43.0.10:53`)
* **Log:** `dial tcp: lookup github.com on 10.43.0.10:53: server misbehaving` / `read udp: i/o timeout to 192.168.0.1:53`
* **Why:** CoreDNS was started on an older network and mounted `/etc/resolv.conf` pointing to `192.168.0.1`. When laptop switched to mobile hotspot (`172.20.10.1`), CoreDNS continued querying the unreachable router.
* **Fix:** Recreate CoreDNS and ArgoCD repo-server to reload host network configuration:
  ```bash
  kubectl rollout restart deployment/coredns -n kube-system
  kubectl rollout restart deployment/argocd-repo-server -n argocd
  ```

#### Gotcha 6: containerd vs Docker Daemon Image Isolation in k3s
* **Log:** `ImagePullBackOff` / `ErrImagePull` on newly built local images.
* **Why:** k3s uses an embedded `containerd` runtime, completely separate from the host's Docker daemon. Building with `docker build` registers images in Docker, not in containerd.
* **Fix:** Import built images directly into k3s containerd namespace:
  ```bash
  docker save <image:tag> | sudo k3s ctr images import -
  ```
  Ensure deployment YAML has `imagePullPolicy: IfNotPresent`.

#### Gotcha 7: MongoDB RollingUpdate PVC Lock Collision
* **Log:** Pod pending or crashed with `WiredTiger.lock` held by previous container.
* **Why:** Deployments default to `strategy: RollingUpdate`, creating the new pod before terminating the old one. Because the PVC uses `ReadWriteOnce`, the second pod cannot mount the volume while the first pod holds it.
* **Fix:** Set deployment strategy to `type: Recreate` for single-replica stateful databases.

---

## Cluster Health Check (Verify Everything)

Run this one-liner to verify all workloads, volumes, and network ingress:
```bash
kubectl get pods,pvc,svc,ingress -n satellite-platform
```

**Expected Healthy State:**
* `pod/gee-service-...` (1/1 Running)
* `pod/image-processor-...` (1/1 Running)
* `pod/mongodb-...` (1/1 Running)
* `pod/rabbitmq-0` (1/1 Running)
* `pod/redis-master-0` (1/1 Running)
* `pod/satellite-backend-...` (1/1 Running)
* `pod/satellite-frontend-...` (1/1 Running)
* 6 PVCs in `Bound` status (`backend-storage`, `gee-storage`, `image-processor-storage`, `mongodb`, `rabbitmq-data`, `redis-data`)
* 2 Ingress resources:
  * `frontend-ingress`: `satellite.localhost` $\rightarrow$ `satellite-frontend:3000`
  * `backend-ingress`: `api.satellite.localhost` $\rightarrow$ `satellite-backend:8080`
