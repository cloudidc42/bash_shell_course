# Part 25: Shell Scripting for DevOps - Docker & Kubernetes
## หลักสูตร Bash/Shell Script ระดับ Intermediate

---

## 25.1 Docker Automation

```bash
# ─── Basic Docker Commands ────────────────────────────────────
docker ps                       # running containers
docker ps -a                    # all containers
docker images                   # local images
docker pull image:tag           # pull image
docker rmi image:tag            # remove image
docker rm container_id          # remove container
docker stop container_id        # stop container
docker logs container_id        # view logs
docker logs -f container_id     # follow logs
docker exec -it container bash  # shell into container

# ─── Docker in Scripts ────────────────────────────────────────

# Check if Docker is running
check_docker() {
    docker info &>/dev/null || { echo "Docker is not running"; exit 1; }
}

# Wait for container to be healthy
wait_for_healthy() {
    local container=$1
    local timeout=${2:-60}
    local elapsed=0
    
    while true; do
        local health
        health=$(docker inspect --format='{{.State.Health.Status}}' "$container" 2>/dev/null || echo "none")
        
        case $health in
            healthy) return 0 ;;
            unhealthy) echo "Container unhealthy: $container"; return 1 ;;
            none) return 0 ;; # no healthcheck
        esac
        
        sleep 2
        (( elapsed += 2 ))
        
        if (( elapsed >= timeout )); then
            echo "Timeout waiting for $container to be healthy"
            return 1
        fi
        
        echo "Waiting for $container... ($elapsed/${timeout}s)"
    done
}

# Check if container exists and is running
container_running() {
    docker ps --filter "name=$1" --format "{{.Names}}" | grep -q "^$1$"
}

# Get container IP
container_ip() {
    docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' "$1"
}

# ─── Docker cleanup ───────────────────────────────────────────
docker_cleanup() {
    echo "Cleaning up Docker resources..."
    
    # Remove stopped containers
    docker container prune -f
    
    # Remove unused images
    docker image prune -f
    
    # Remove unused volumes
    docker volume prune -f
    
    # Remove unused networks
    docker network prune -f
    
    # Nuclear option (remove everything)
    # docker system prune -af --volumes
    
    echo "Cleanup complete"
    docker system df  # show disk usage
}

# ─── Build and tag ────────────────────────────────────────────
build_and_tag() {
    local image_name=$1
    local version=${2:-$(git describe --tags --always)}
    local registry=${3:-}
    
    local full_name="${registry:+$registry/}${image_name}"
    
    echo "Building $full_name:$version"
    
    docker build \
        --build-arg VERSION="$version" \
        --build-arg BUILD_DATE="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
        --build-arg VCS_REF="$(git rev-parse HEAD)" \
        -t "${full_name}:${version}" \
        -t "${full_name}:latest" \
        .
    
    echo "Built: ${full_name}:${version}"
    
    # Push if registry specified
    if [[ -n "$registry" ]]; then
        docker push "${full_name}:${version}"
        docker push "${full_name}:latest"
        echo "Pushed to $registry"
    fi
}

# ─── Docker Compose helpers ───────────────────────────────────
compose_up() {
    local env=${1:-development}
    local compose_files=("-f" "docker-compose.yml")
    
    [[ -f "docker-compose.${env}.yml" ]] && \
        compose_files+=("-f" "docker-compose.${env}.yml")
    
    docker compose "${compose_files[@]}" up -d
    
    echo "Waiting for services to be healthy..."
    docker compose "${compose_files[@]}" ps
}

compose_down() {
    docker compose down --remove-orphans
}

compose_logs() {
    local service=${1:-}
    docker compose logs -f --tail=100 $service
}

# ─── Multi-stage build script ─────────────────────────────────
cat > Dockerfile << 'EOF'
# Build stage
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

# Runtime stage
FROM node:18-alpine AS runtime
WORKDIR /app

# Non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules

USER appuser
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "dist/server.js"]
EOF
```

---

## 25.2 Kubernetes Automation

```bash
# ─── kubectl basics ───────────────────────────────────────────
# Contexts
kubectl config get-contexts
kubectl config use-context my-cluster
kubectl config current-context

# Resources
kubectl get pods
kubectl get pods -n namespace
kubectl get pods --all-namespaces
kubectl get pods -o wide
kubectl get deployments
kubectl get services
kubectl get nodes

# Describe
kubectl describe pod pod-name
kubectl describe deployment my-app

# Logs
kubectl logs pod-name
kubectl logs -f pod-name                   # follow
kubectl logs pod-name -c container-name    # specific container
kubectl logs --previous pod-name           # previous container

# Exec
kubectl exec -it pod-name -- bash
kubectl exec pod-name -- ls /app

# ─── kubectl in scripts ───────────────────────────────────────
# Check cluster is accessible
check_cluster() {
    kubectl cluster-info &>/dev/null || { echo "Cluster not accessible"; exit 1; }
}

# Wait for deployment to rollout
wait_for_rollout() {
    local deployment=$1
    local namespace=${2:-default}
    local timeout=${3:-300}
    
    kubectl rollout status "deployment/$deployment" \
        -n "$namespace" \
        --timeout="${timeout}s"
}

# Get pod by label
get_pod() {
    local label_selector=$1
    local namespace=${2:-default}
    
    kubectl get pods -n "$namespace" \
        -l "$label_selector" \
        -o jsonpath='{.items[0].metadata.name}' 2>/dev/null
}

# Scale deployment
scale() {
    local deployment=$1
    local replicas=$2
    local namespace=${3:-default}
    
    kubectl scale deployment "$deployment" \
        --replicas="$replicas" \
        -n "$namespace"
    
    wait_for_rollout "$deployment" "$namespace"
}

# ─── Deployment script ────────────────────────────────────────
deploy() {
    local app=$1
    local image=$2
    local namespace=${3:-default}
    
    echo "Deploying $app with image $image to $namespace"
    
    # Check if deployment exists
    if kubectl get deployment "$app" -n "$namespace" &>/dev/null; then
        # Update image
        kubectl set image "deployment/$app" \
            "${app}=${image}" \
            -n "$namespace"
    else
        # Create from manifest
        kubectl apply -f "k8s/${app}.yaml" -n "$namespace"
    fi
    
    # Wait for rollout
    if wait_for_rollout "$app" "$namespace"; then
        echo "✓ Deployed successfully: $app"
        kubectl get deployment "$app" -n "$namespace"
    else
        echo "✗ Deployment failed, rolling back..."
        kubectl rollout undo "deployment/$app" -n "$namespace"
        return 1
    fi
}

# ─── Kubernetes manifest generation ──────────────────────────
generate_deployment() {
    local app=$1
    local image=$2
    local replicas=${3:-1}
    local port=${4:-8080}
    
    cat << YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${app}
  labels:
    app: ${app}
spec:
  replicas: ${replicas}
  selector:
    matchLabels:
      app: ${app}
  template:
    metadata:
      labels:
        app: ${app}
    spec:
      containers:
      - name: ${app}
        image: ${image}
        ports:
        - containerPort: ${port}
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /health
            port: ${port}
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: ${port}
          initialDelaySeconds: 15
          periodSeconds: 20
YAML
}

# ─── Secrets management ───────────────────────────────────────
# Create secret from .env file
create_secret_from_env() {
    local secret_name=$1
    local env_file=$2
    local namespace=${3:-default}
    
    kubectl create secret generic "$secret_name" \
        --from-env-file="$env_file" \
        -n "$namespace" \
        --dry-run=client -o yaml | \
        kubectl apply -f -
}

# Decode secret
decode_secret() {
    local secret_name=$1
    local key=$2
    local namespace=${3:-default}
    
    kubectl get secret "$secret_name" -n "$namespace" \
        -o jsonpath="{.data.${key}}" | base64 -d
}

# ─── Namespace operations ─────────────────────────────────────
setup_namespace() {
    local ns=$1
    
    # Create namespace if not exists
    kubectl get namespace "$ns" &>/dev/null || \
        kubectl create namespace "$ns"
    
    # Label for istio if available
    kubectl label namespace "$ns" istio-injection=enabled --overwrite 2>/dev/null || true
}

cleanup_namespace() {
    local ns=$1
    
    echo "Cleaning up namespace: $ns"
    kubectl delete all --all -n "$ns"
    kubectl delete namespace "$ns"
}
```

---

## 25.3 CI/CD Pipeline Script

```bash
#!/bin/bash
# pipeline.sh - CI/CD pipeline

set -euo pipefail

# ─── Configuration ────────────────────────────────────────────
APP_NAME="${APP_NAME:?APP_NAME required}"
REGISTRY="${REGISTRY:-docker.io/myorg}"
ENVIRONMENT="${1:-staging}"
VERSION=$(git describe --tags --always --dirty 2>/dev/null || echo "dev-$(date +%Y%m%d)")
IMAGE="${REGISTRY}/${APP_NAME}:${VERSION}"

# Colors
info()  { echo -e "\033[32m[INFO]\033[0m $*"; }
warn()  { echo -e "\033[33m[WARN]\033[0m $*"; }
error() { echo -e "\033[31m[ERROR]\033[0m $*" >&2; exit 1; }

# ─── Pipeline stages ──────────────────────────────────────────
stage() {
    local name=$1
    echo ""
    echo "═══════════════════════════════════════"
    echo "  Stage: $name"
    echo "═══════════════════════════════════════"
}

# Test
run_tests() {
    stage "Test"
    
    info "Running unit tests..."
    npm test
    
    info "Running integration tests..."
    docker compose -f docker-compose.test.yml up --abort-on-container-exit
    
    info "Tests passed!"
}

# Build
build() {
    stage "Build"
    
    info "Building image: $IMAGE"
    
    docker build \
        --build-arg VERSION="$VERSION" \
        --cache-from "${REGISTRY}/${APP_NAME}:latest" \
        -t "$IMAGE" \
        -t "${REGISTRY}/${APP_NAME}:latest" \
        .
    
    info "Scanning for vulnerabilities..."
    docker run --rm \
        -v /var/run/docker.sock:/var/run/docker.sock \
        aquasec/trivy image --exit-code 1 --severity HIGH,CRITICAL "$IMAGE" || \
        warn "Vulnerability scan found issues (continuing)"
    
    info "Build complete: $IMAGE"
}

# Push
push() {
    stage "Push"
    
    info "Pushing to registry..."
    docker push "$IMAGE"
    docker push "${REGISTRY}/${APP_NAME}:latest"
    
    info "Pushed: $IMAGE"
}

# Deploy
deploy() {
    stage "Deploy to $ENVIRONMENT"
    
    local namespace="$APP_NAME-$ENVIRONMENT"
    
    info "Setting up namespace: $namespace"
    setup_namespace "$namespace"
    
    info "Deploying $IMAGE"
    deploy_app "$APP_NAME" "$IMAGE" "$namespace"
    
    info "Running smoke tests..."
    run_smoke_tests "$namespace"
    
    info "Deployment successful!"
}

# Smoke tests
run_smoke_tests() {
    local namespace=$1
    
    local pod
    pod=$(get_pod "app=$APP_NAME" "$namespace")
    
    [[ -n "$pod" ]] || error "No pods found for $APP_NAME"
    
    # Wait for pod to be ready
    kubectl wait --for=condition=ready pod/"$pod" \
        -n "$namespace" --timeout=60s
    
    # Check health endpoint
    local health
    health=$(kubectl exec "$pod" -n "$namespace" -- \
        wget -qO- http://localhost:8080/health 2>/dev/null || echo "FAILED")
    
    echo "$health" | jq -e '.status == "healthy"' &>/dev/null || \
        error "Health check failed: $health"
    
    info "Smoke tests passed"
}

# Rollback
rollback() {
    stage "Rollback"
    
    local namespace="$APP_NAME-$ENVIRONMENT"
    
    warn "Rolling back $APP_NAME in $namespace"
    kubectl rollout undo "deployment/$APP_NAME" -n "$namespace"
    wait_for_rollout "$APP_NAME" "$namespace"
    
    info "Rollback complete"
}

# ─── Main ─────────────────────────────────────────────────────
info "Pipeline: $APP_NAME v$VERSION → $ENVIRONMENT"

case "${PIPELINE_STAGE:-all}" in
    test)    run_tests ;;
    build)   build ;;
    push)    push ;;
    deploy)  deploy ;;
    all)
        run_tests
        build
        push
        deploy
        ;;
esac

info "Pipeline complete!"
```

---

## 25.4 Docker Compose Production Template

```yaml
# docker-compose.prod.yml
version: '3.9'

services:
  app:
    image: ${REGISTRY}/myapp:${VERSION}
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      NODE_ENV: production
      DATABASE_URL: ${DATABASE_URL}
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  db:
    image: postgres:15-alpine
    restart: unless-stopped
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --appendonly yes
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - app

volumes:
  pgdata:
  redisdata:
```

---

## 25.5 Exercises

### Exercise 1: Docker Registry Manager
สร้าง script ที่:
- Push/pull images
- Clean up old tags
- List images by size
- Report disk usage

### Exercise 2: Kubernetes Deployer
สร้าง complete deployment tool:
- Blue/green deployment
- Canary deployment
- Auto-rollback on failure
- Deployment history

### Exercise 3: Container Security Scanner
สร้าง security scanner:
- Scan images for vulnerabilities
- Check container configurations
- Report compliance issues
- Block deployments with critical CVEs

---

## สรุป Part 25

✅ Docker automation (build, tag, push, cleanup)  
✅ Wait for healthy containers  
✅ Docker Compose helpers  
✅ kubectl automation  
✅ Kubernetes deployment scripts  
✅ Manifest generation  
✅ Secrets management  
✅ Complete CI/CD pipeline  

---

**→ Part 26: SSH Automation & Remote Management**
