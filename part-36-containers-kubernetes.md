# Part 36: Containers & Kubernetes Automation
## หลักสูตร Bash/Shell Script ระดับ Advanced

---

## 36.1 Docker Automation

```bash
#!/bin/bash
# docker_manager.sh

monitor_containers() {
    local interval=${1:-10}
    
    while true; do
        clear
        echo "=== Container Health: $(date) ==="
        echo ""
        
        docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}" 2>/dev/null
        echo ""
        
        docker ps --filter "health=unhealthy" --format "{{.Names}}" | \
            while read -r name; do
                echo "UNHEALTHY: $name"
                docker restart "$name"
                echo "  Restarted: $name"
            done
        
        echo ""
        echo "=== Resource Usage ==="
        docker stats --no-stream --format \
            "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}" \
            2>/dev/null
        
        sleep "$interval"
    done
}

cleanup_docker() {
    echo "=== Docker Cleanup ==="
    
    local stopped
    stopped=$(docker ps -a -q --filter status=exited 2>/dev/null)
    if [[ -n "$stopped" ]]; then
        echo "$stopped" | xargs docker rm -v
        echo "Removed stopped containers"
    fi
    
    local dangling
    dangling=$(docker images -q --filter dangling=true 2>/dev/null)
    if [[ -n "$dangling" ]]; then
        echo "$dangling" | xargs docker rmi
        echo "Removed dangling images"
    fi
    
    docker volume prune -f
    docker network prune -f
    docker system df
}

deploy_stack() {
    local compose_file=$1
    local env_file=${2:-.env}
    local stack_name=${3:-myapp}
    
    echo "Deploying stack: $stack_name"
    
    docker-compose -f "$compose_file" pull
    
    docker-compose -f "$compose_file" \
        --env-file "$env_file" \
        -p "$stack_name" \
        up -d \
        --remove-orphans \
        --force-recreate
    
    local timeout=60
    local elapsed=0
    
    while (( elapsed < timeout )); do
        local unhealthy
        unhealthy=$(docker ps --filter "label=com.docker.compose.project=$stack_name" \
            --filter "health=unhealthy" -q)
        
        local starting
        starting=$(docker ps --filter "label=com.docker.compose.project=$stack_name" \
            --filter "health=starting" -q)
        
        if [[ -z "$unhealthy" ]] && [[ -z "$starting" ]]; then
            echo "All containers healthy"
            break
        fi
        
        echo "Waiting for containers... (${elapsed}s)"
        sleep 5
        (( elapsed += 5 ))
    done
    
    echo "Stack deployed: $stack_name"
}

build_and_push() {
    local image_name=$1
    local tag=${2:-latest}
    local registry=${3:-docker.io}
    local dockerfile=${4:-Dockerfile}
    
    local full_name="${registry}/${image_name}:${tag}"
    
    docker build \
        --file "$dockerfile" \
        --tag "$full_name" \
        --build-arg "BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
        --build-arg "GIT_SHA=$(git rev-parse --short HEAD 2>/dev/null || echo 'unknown')" \
        --label "build.date=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
        --label "build.version=$tag" \
        .
    
    docker push "$full_name"
    echo "Done: $full_name"
}

aggregate_logs() {
    local output_dir=${1:-/var/log/containers}
    local rotate_days=${2:-7}
    
    mkdir -p "$output_dir"
    
    docker ps -q | while read -r container_id; do
        local name
        name=$(docker inspect --format '{{.Name}}' "$container_id" | tr -d '/')
        local log_file="${output_dir}/${name}.log"
        
        if [[ -f "$log_file" ]] && \
           [[ $(find "$log_file" -mtime +"$rotate_days") ]]; then
            mv "$log_file" "${log_file}.$(date +%Y%m%d)"
            gzip "${log_file}.$(date +%Y%m%d)"
        fi
        
        docker logs --since "1h" "$container_id" >> "$log_file" 2>&1
    done
}
```

---

## 36.2 Kubernetes Automation

```bash
#!/bin/bash
# kubectl_automation.sh

create_namespace() {
    local ns=$1
    local labels=${2:-""}
    
    if kubectl get namespace "$ns" &>/dev/null; then
        echo "Namespace exists: $ns"
        return 0
    fi
    
    kubectl create namespace "$ns"
    [[ -n "$labels" ]] && kubectl label namespace "$ns" $labels
    echo "Created namespace: $ns"
}

pod_dashboard() {
    local namespace=${1:---all-namespaces}
    
    while true; do
        clear
        echo "=== Pod Dashboard: $(date) ==="
        echo ""
        
        if [[ "$namespace" == "--all-namespaces" ]]; then
            kubectl get pods --all-namespaces -o wide --sort-by='.metadata.namespace' 2>/dev/null
        else
            kubectl get pods -n "$namespace" -o wide 2>/dev/null
        fi
        
        echo ""
        echo "=== Problem Pods ==="
        kubectl get pods --all-namespaces \
            --field-selector="status.phase!=Running,status.phase!=Succeeded" \
            2>/dev/null | head -20
        
        sleep 5
    done
}

rolling_deploy() {
    local deployment=$1
    local namespace=$2
    local new_image=$3
    local timeout=${4:-300}
    
    echo "Rolling deploy: $deployment → $new_image"
    
    kubectl set image "deployment/$deployment" \
        "*=$new_image" \
        -n "$namespace"
    
    if kubectl rollout status "deployment/$deployment" \
        -n "$namespace" \
        --timeout="${timeout}s"; then
        echo "Deployment successful"
    else
        echo "Deployment failed, rolling back..."
        kubectl rollout undo "deployment/$deployment" -n "$namespace"
        return 1
    fi
}

auto_scale() {
    local deployment=$1
    local namespace=$2
    local min_replicas=${3:-2}
    local max_replicas=${4:-10}
    local cpu_threshold=${5:-70}
    
    kubectl autoscale deployment "$deployment" \
        -n "$namespace" \
        --min="$min_replicas" \
        --max="$max_replicas" \
        --cpu-percent="$cpu_threshold"
    
    echo "HPA created for $deployment"
    kubectl get hpa "$deployment" -n "$namespace"
}

create_secret_from_env() {
    local secret_name=$1
    local namespace=$2
    local env_file=$3
    
    local args=()
    while IFS= read -r line; do
        [[ "$line" =~ ^# ]] && continue
        [[ -z "$line" ]] && continue
        
        local key value
        key="${line%%=*}"
        value="${line#*=}"
        
        args+=("--from-literal=${key}=${value}")
    done < "$env_file"
    
    kubectl create secret generic "$secret_name" \
        -n "$namespace" \
        "${args[@]}" \
        --dry-run=client -o yaml | \
        kubectl apply -f -
    
    echo "Secret created/updated: $secret_name"
}

debug_pod() {
    local pod_name=$1
    local namespace=${2:-default}
    local container=${3:-}
    
    echo "=== Pod Debug Info: $pod_name ==="
    echo ""
    
    echo "--- Pod Events ---"
    kubectl get events -n "$namespace" \
        --field-selector "involvedObject.name=$pod_name" \
        --sort-by='.lastTimestamp' | tail -20
    echo ""
    
    echo "--- Pod Describe ---"
    kubectl describe pod "$pod_name" -n "$namespace"
    echo ""
    
    echo "--- Recent Logs ---"
    kubectl logs "$pod_name" \
        -n "$namespace" \
        ${container:+-c "$container"} \
        --tail=50 \
        --previous 2>/dev/null || \
    kubectl logs "$pod_name" \
        -n "$namespace" \
        ${container:+-c "$container"} \
        --tail=50
}

cluster_health() {
    echo "=== Cluster Health Check ==="
    echo ""
    
    echo "--- Nodes ---"
    kubectl get nodes -o wide 2>/dev/null
    echo ""
    
    echo "--- System Pods ---"
    kubectl get pods -n kube-system 2>/dev/null | grep -v "Running\|Completed"
    echo ""
    
    echo "--- Resource Usage ---"
    kubectl top nodes 2>/dev/null || echo "(metrics-server not available)"
    echo ""
    
    echo "--- Failed Pods ---"
    kubectl get pods --all-namespaces --field-selector="status.phase=Failed" 2>/dev/null
    
    echo ""
    echo "--- Pending Pods ---"
    kubectl get pods --all-namespaces --field-selector="status.phase=Pending" 2>/dev/null
}
```

---

## 36.3 Container Registry Management

```bash
#!/bin/bash
# registry_manager.sh

promote_image() {
    local source_image=$1
    local source_tag=$2
    local target_registry=$3
    local target_tag=$4
    
    local source="${source_image}:${source_tag}"
    local target="${target_registry}/${source_image##*/}:${target_tag}"
    
    echo "Promoting: $source → $target"
    
    docker pull "$source"
    docker tag "$source" "$target"
    docker push "$target"
    
    echo "Promoted successfully"
}

scan_image() {
    local image=$1
    
    if command -v trivy &>/dev/null; then
        trivy image --severity HIGH,CRITICAL "$image"
    elif command -v grype &>/dev/null; then
        grype "$image"
    else
        echo "No scanner available (install trivy or grype)"
        return 1
    fi
}

build_multiarch() {
    local image=$1
    local tag=$2
    local platforms=${3:-"linux/amd64,linux/arm64"}
    
    docker buildx create --use --name multiarch 2>/dev/null || true
    
    docker buildx build \
        --platform "$platforms" \
        --tag "${image}:${tag}" \
        --push \
        .
    
    echo "Multi-arch build: ${image}:${tag} (${platforms})"
}
```

---

## 36.4 Exercises

### Exercise 1: Container Auto-Healer
สร้าง service:
- Monitor container health
- Auto-restart on failure
- Alert after N restarts
- Write restart history

### Exercise 2: K8s Deployment Pipeline
สร้าง deployment automation:
- Build image with git SHA tag
- Push to registry
- Update k8s deployment
- Verify rollout success
- Auto-rollback on failure

### Exercise 3: Log Collection System
สร้าง centralized logging:
- Collect from all containers
- Parse and structure
- Search interface (CLI)
- Retention policy

---

## สรุป Part 36

✅ Docker container monitoring  
✅ Image cleanup automation  
✅ Multi-container stack deployment  
✅ Build and push automation  
✅ Kubernetes rolling deployments  
✅ Auto-scaling configuration  
✅ Secret management  
✅ Pod debugging tools  
✅ Cluster health checks  

---

**→ Part 37: API Integration & Webhook Automation**
