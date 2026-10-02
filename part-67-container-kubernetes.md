# Part 67: Container and Kubernetes Automation
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 67.1 Docker Automation

```bash
#!/bin/bash
# docker_ops.sh - Docker build, run, and management

set -euo pipefail

DOCKER_REGISTRY="${DOCKER_REGISTRY:-docker.io}"
DOCKER_NAMESPACE="${DOCKER_NAMESPACE:-myorg}"

docker_log() {
    printf '%s [DOCKER] %s\n' "$(date '+%Y-%m-%dT%H:%M:%S')" "$1" | tee -a "${DOCKER_LOG:-/tmp/docker.log}"
}

# ─── Build ──────────────────────────────────────────────────────────────
docker_build() {
    local name=$1 tag=${2:-latest} context=${3:-.}
    local dockerfile=${4:-Dockerfile}
    local full_image="${DOCKER_REGISTRY}/${DOCKER_NAMESPACE}/${name}:${tag}"

    docker_log "Building: $full_image"

    local build_args=()
    [[ -n "${BUILD_DATE:-}" ]] && build_args+=(--build-arg "BUILD_DATE=$BUILD_DATE")
    [[ -n "${GIT_SHA:-}" ]] && build_args+=(--build-arg "GIT_SHA=$GIT_SHA")

    docker build \
        -f "$dockerfile" \
        -t "$full_image" \
        --label "org.opencontainers.image.revision=${GIT_SHA:-unknown}" \
        --label "org.opencontainers.image.created=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
        "${build_args[@]}" \
        "$context"

    docker_log "Build complete: $full_image"
    echo "$full_image"
}

docker_tag_push() {
    local source_image=$1
    shift
    local tags=("$@")

    for tag in "${tags[@]}"; do
        docker tag "$source_image" "$tag"
        docker push "$tag"
        docker_log "Pushed: $tag"
    done
}

docker_pull_with_retry() {
    local image=$1 retries=${2:-3}

    local i
    for (( i=1; i<=retries; i++ )); do
        if docker pull "$image"; then
            return 0
        fi
        docker_log "Pull attempt $i/$retries failed: $image"
        sleep $(( i * 2 ))
    done
    return 1
}

# ─── Cleanup ────────────────────────────────────────────────────────────
docker_cleanup() {
    local age_hours=${1:-24}

    docker_log "Cleaning images older than ${age_hours}h"

    # Remove stopped containers
    docker container prune -f --filter "until=${age_hours}h"

    # Remove dangling images
    docker image prune -f --filter "until=${age_hours}h"

    # Remove unused volumes
    docker volume prune -f

    local freed; freed=$(docker system df --format '{{.ReclaimableSize}}' 2>/dev/null | head -3)
    docker_log "Cleanup complete"
}

docker_cleanup_old_images() {
    local name=$1 keep=${2:-5}

    # Keep latest N images by tag date
    docker images --format '{{.Repository}}:{{.Tag}} {{.CreatedAt}}' | \
        grep "^${DOCKER_REGISTRY}/${DOCKER_NAMESPACE}/${name}:" | \
        sort -k2 -r | \
        tail -n "+$(( keep + 1 ))" | \
        awk '{print $1}' | \
        xargs -r docker rmi 2>/dev/null || true
}

# ─── Vulnerability Scan ───────────────────────────────────────────────
docker_scan() {
    local image=$1 severity=${2:-HIGH,CRITICAL}
    local output_file=${3:-/tmp/scan_results.json}

    if command -v trivy &>/dev/null; then
        trivy image \
            --format json \
            --output "$output_file" \
            --severity "$severity" \
            --exit-code 0 \
            "$image"

        local critical; critical=$(jq '[.Results[]?.Vulnerabilities[]? | select(.Severity=="CRITICAL")] | length' "$output_file" 2>/dev/null || echo 0)
        local high; high=$(jq '[.Results[]?.Vulnerabilities[]? | select(.Severity=="HIGH")] | length' "$output_file" 2>/dev/null || echo 0)

        docker_log "Scan results: CRITICAL=$critical HIGH=$high  ($image)"
        echo "CRITICAL=$critical HIGH=$high"

        (( critical == 0 ))
    elif command -v grype &>/dev/null; then
        grype "$image" --fail-on critical
    else
        docker_log "WARN: No vulnerability scanner (trivy/grype) available"
    fi
}
```

---

## 67.2 Docker Compose Operations

```bash
#!/bin/bash
# compose_ops.sh - Docker Compose lifecycle management

COMPOSE_FILE="${COMPOSE_FILE:-docker-compose.yml}"
COMPOSE_PROJECT="${COMPOSE_PROJECT:-app}"

compose_cmd() {
    if command -v docker &>/dev/null && docker compose version &>/dev/null 2>&1; then
        docker compose -f "$COMPOSE_FILE" -p "$COMPOSE_PROJECT" "$@"
    elif command -v docker-compose &>/dev/null; then
        docker-compose -f "$COMPOSE_FILE" -p "$COMPOSE_PROJECT" "$@"
    else
        echo "ERROR: docker compose not available" >&2
        return 1
    fi
}

compose_up() {
    local services=("$@")

    docker_log "Starting services: ${services[*]:-all}"

    if (( ${#services[@]} > 0 )); then
        compose_cmd up -d --remove-orphans "${services[@]}"
    else
        compose_cmd up -d --remove-orphans
    fi
}

compose_down() {
    local remove_volumes=${1:-false}

    if [[ "$remove_volumes" == true ]]; then
        compose_cmd down --volumes --remove-orphans
    else
        compose_cmd down --remove-orphans
    fi
    docker_log "Services stopped"
}

compose_restart() {
    local service=$1
    compose_cmd restart "$service"
    docker_log "Restarted: $service"
}

compose_wait_healthy() {
    local service=$1 timeout=${2:-120}
    local deadline=$(( $(date +%s) + timeout ))

    docker_log "Waiting for $service to be healthy..."

    while (( $(date +%s) < deadline )); do
        local health
        health=$(compose_cmd ps --format json 2>/dev/null | \
            jq -r --arg svc "$service" 'select(.Service==$svc) | .Health // .State' 2>/dev/null | head -1)

        case "$health" in
            healthy|running)
                docker_log "$service is $health"
                return 0
                ;;
            unhealthy|exited)
                docker_log "ERROR: $service is $health"
                return 1
                ;;
        esac
        sleep 5
    done

    docker_log "ERROR: $service health check timed out after ${timeout}s"
    return 1
}

compose_logs() {
    local service=$1 lines=${2:-100}
    compose_cmd logs --tail="$lines" --no-log-prefix "$service"
}

compose_exec() {
    local service=$1; shift
    compose_cmd exec -T "$service" "$@"
}

compose_scale() {
    local service=$1 replicas=$2
    compose_cmd up -d --scale "${service}=${replicas}" "$service"
    docker_log "Scaled $service to $replicas replicas"
}

compose_status() {
    echo "=== Docker Compose Status ==="
    compose_cmd ps --format 'table {{.Name}}\t{{.Status}}\t{{.Ports}}'
}
```

---

## 67.3 Container Monitoring

```bash
#!/bin/bash
# container_monitor.sh - Container runtime metrics

monitor_container_stats() {
    local interval=${1:-10} duration=${2:-300}
    local iterations=$(( duration / interval ))

    printf '%-30s %8s %8s %10s %10s\n' 'Container' 'CPU%' 'Mem%' 'MemUsed' 'NetI/O'
    printf '%-30s %8s %8s %10s %10s\n' '---------' '----' '----' '-------' '------'

    local i
    for (( i=0; i<iterations; i++ )); do
        docker stats --no-stream --format \
            '{{.Name}}\t{{.CPUPerc}}\t{{.MemPerc}}\t{{.MemUsage}}\t{{.NetIO}}' 2>/dev/null | \
        while IFS=$'\t' read -r name cpu mem mem_usage net; do
            printf '%-30s %8s %8s %10s %10s\n' \
                "${name:0:30}" "$cpu" "$mem" "${mem_usage%%/*}" "${net%%/*/}"
        done
        echo "---"
        sleep "$interval"
    done
}

find_unhealthy_containers() {
    docker ps --format '{{.Names}}\t{{.Status}}' | \
        grep -v 'healthy\|Up [0-9]' | \
        awk '{print $1, "STATUS:", $2}'
}

container_resource_usage() {
    local threshold_cpu=${1:-80} threshold_mem=${2:-80}

    echo "=== High Resource Containers ==="
    docker stats --no-stream --format '{{.Name}} {{.CPUPerc}} {{.MemPerc}}' 2>/dev/null | \
    while read -r name cpu mem; do
        local cpu_val=${cpu//%/}
        local mem_val=${mem//%/}
        if (( ${cpu_val%.*} > threshold_cpu || ${mem_val%.*} > threshold_mem )); then
            printf '  %-30s CPU: %s  MEM: %s  [WARNING]\n' "$name" "$cpu" "$mem"
        fi
    done
}

container_log_tail() {
    local container=$1 lines=${2:-50} follow=${3:-false}

    if [[ "$follow" == true ]]; then
        docker logs -f --tail="$lines" "$container"
    else
        docker logs --tail="$lines" "$container" 2>&1
    fi
}
```

---

## 67.4 Kubernetes Automation

```bash
#!/bin/bash
# k8s_ops.sh - Kubernetes resource management

K8S_NAMESPACE="${K8S_NAMESPACE:-default}"
KUBECTL="${KUBECTL:-kubectl}"

k8s_log() {
    printf '%s [K8S] %s\n' "$(date '+%Y-%m-%dT%H:%M:%S')" "$1" | tee -a "${K8S_LOG:-/tmp/k8s.log}"
}

k() {
    "$KUBECTL" --namespace="$K8S_NAMESPACE" "$@"
}

# ─── Apply and Wait ────────────────────────────────────────────────────
k8s_apply() {
    local manifest=$1 timeout=${2:-300}

    k8s_log "Applying: $manifest"
    k apply -f "$manifest"

    k8s_log "Waiting for rollout..."
    k wait --for=condition=Available deployment \
        --all --timeout="${timeout}s" 2>/dev/null || true

    k8s_log "Apply complete: $manifest"
}

k8s_rollout_watch() {
    local deployment=$1 timeout=${2:-300}

    k8s_log "Watching rollout: $deployment"

    k rollout status deployment/"$deployment" --timeout="${timeout}s"
    local exit_code=$?

    if (( exit_code == 0 )); then
        k8s_log "Rollout complete: $deployment"
    else
        k8s_log "Rollout failed: $deployment"
        k rollout history deployment/"$deployment" 2>/dev/null
        return 1
    fi
}

k8s_rollout_undo() {
    local deployment=$1 revision=${2:-}

    k8s_log "Rolling back: $deployment"
    if [[ -n "$revision" ]]; then
        k rollout undo deployment/"$deployment" --to-revision="$revision"
    else
        k rollout undo deployment/"$deployment"
    fi
    k8s_rollout_watch "$deployment"
}

# ─── Image Update ─────────────────────────────────────────────────────────
k8s_set_image() {
    local deployment=$1 container=$2 image=$3

    k8s_log "Updating image: ${deployment}/${container} -> $image"
    k set image deployment/"$deployment" "${container}=${image}" --record 2>/dev/null || \
    k set image deployment/"$deployment" "${container}=${image}"
    k8s_rollout_watch "$deployment"
}

# ─── Port Forward ────────────────────────────────────────────────────────
k8s_port_forward() {
    local resource=$1 local_port=$2 remote_port=$3
    local pid_file="/tmp/k8s_pf_${resource//\//_}.pid"

    k port-forward "$resource" "${local_port}:${remote_port}" &
    echo $! > "$pid_file"
    k8s_log "Port forward: ${local_port} -> ${resource}:${remote_port} (pid: $!)"
    sleep 2  # Let it establish
}

k8s_stop_port_forward() {
    local resource=$1
    local pid_file="/tmp/k8s_pf_${resource//\//_}.pid"

    if [[ -f "$pid_file" ]]; then
        kill "$(cat "$pid_file")" 2>/dev/null || true
        rm -f "$pid_file"
        k8s_log "Stopped port forward: $resource"
    fi
}

# ─── Exec into Pod ─────────────────────────────────────────────────────
k8s_exec() {
    local deployment=$1; shift
    local pod; pod=$(k get pods -l "app=${deployment}" -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)

    if [[ -z "$pod" ]]; then
        k8s_log "ERROR: No pods found for deployment: $deployment"
        return 1
    fi

    k exec -it "$pod" -- "$@"
}

k8s_exec_noninteractive() {
    local deployment=$1; shift
    local pod; pod=$(k get pods -l "app=${deployment}" -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
    k exec "$pod" -- "$@"
}

# ─── Resource Watch ──────────────────────────────────────────────────────
k8s_watch_resources() {
    local interval=${1:-10}
    while true; do
        clear
        echo "=== K8s Resources in $K8S_NAMESPACE ($(date)) ==="
        echo ""
        echo "--- Deployments ---"
        k get deployments -o wide 2>/dev/null
        echo ""
        echo "--- Pods ---"
        k get pods 2>/dev/null
        sleep "$interval"
    done
}
```

---

## 67.5 Helm Operations

```bash
#!/bin/bash
# helm_ops.sh - Helm chart management

HELM_TIMEOUT="${HELM_TIMEOUT:-10m}"

helm_log() {
    printf '%s [HELM] %s\n' "$(date '+%Y-%m-%dT%H:%M:%S')" "$1" | tee -a "${HELM_LOG:-/tmp/helm.log}"
}

helm_install_or_upgrade() {
    local release=$1 chart=$2 namespace=${3:-default}
    shift 3
    local values_files=()
    local set_args=()

    while [[ $# -gt 0 ]]; do
        case $1 in
            -f) values_files+=(-f "$2"); shift 2 ;;
            --set) set_args+=(--set "$2"); shift 2 ;;
            *) shift ;;
        esac
    done

    helm_log "Deploy: $release ($chart) -> ns/$namespace"

    if helm status "$release" -n "$namespace" &>/dev/null 2>&1; then
        helm upgrade "$release" "$chart" \
            -n "$namespace" \
            --timeout "$HELM_TIMEOUT" \
            --wait \
            "${values_files[@]}" \
            "${set_args[@]}" \
            --atomic
        helm_log "Upgraded: $release"
    else
        helm install "$release" "$chart" \
            -n "$namespace" \
            --create-namespace \
            --timeout "$HELM_TIMEOUT" \
            --wait \
            "${values_files[@]}" \
            "${set_args[@]}"
        helm_log "Installed: $release"
    fi
}

helm_rollback() {
    local release=$1 revision=${2:-0} namespace=${3:-default}

    helm_log "Rolling back $release to revision $revision"
    helm rollback "$release" "$revision" -n "$namespace" --wait
    helm_log "Rollback complete: $release"
}

helm_diff() {
    local release=$1 chart=$2 namespace=${3:-default}
    shift 3

    if helm plugin list 2>/dev/null | grep -q 'diff'; then
        helm diff upgrade "$release" "$chart" -n "$namespace" "$@"
    else
        helm_log "WARN: helm-diff plugin not installed"
        helm get values "$release" -n "$namespace"
    fi
}

helm_status_all() {
    local namespace=${1:-}

    if [[ -n "$namespace" ]]; then
        helm list -n "$namespace" --output table
    else
        helm list --all-namespaces --output table
    fi
}

helm_cleanup_failed() {
    local namespace=${1:-default}

    helm list -n "$namespace" --failed --short 2>/dev/null | \
    while read -r release; do
        helm_log "Removing failed release: $release"
        helm uninstall "$release" -n "$namespace"
    done
}
```

---

## 67.6 Kubernetes Health Checks

```bash
#!/bin/bash
# k8s_health.sh - Kubernetes cluster and workload health

k8s_health_check() {
    local namespace=${1:-default}
    local issues=0

    echo "=== Kubernetes Health Check ==="
    echo "Namespace: $namespace"
    echo ""

    # Check deployments
    echo "--- Deployments ---"
    kubectl get deployments -n "$namespace" -o json 2>/dev/null | \
    jq -r '.items[] | \n        .metadata.name as $name |\n        .spec.replicas as $desired |\n        .status.readyReplicas as $ready |\n        if ($ready // 0) < $desired then\n            "[WARN] \($name): \($ready // 0)/\($desired) ready"\n        else\n            "[OK]   \($name): \($ready // 0)/\($desired) ready"\n        end' | while read -r line; do
            echo "  $line"
            [[ "$line" =~ WARN ]] && (( issues++ )) || true
        done

    echo ""

    # Check pods
    echo "--- Unhealthy Pods ---"
    kubectl get pods -n "$namespace" --field-selector='status.phase!=Running,status.phase!=Succeeded' \
        -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}' 2>/dev/null | \
    while IFS=$'\t' read -r name phase; do
        echo "  [WARN] $name: $phase"
        (( issues++ ))
    done

    echo ""

    # Check nodes
    echo "--- Nodes ---"
    kubectl get nodes -o json 2>/dev/null | \
    jq -r '.items[] | .metadata.name as $n | .status.conditions[] | select(.type=="Ready") | if .status=="True" then "[OK]   " + $n else "[WARN] " + $n + " (NotReady)" end' | \
    while read -r line; do
        echo "  $line"
        [[ "$line" =~ WARN ]] && (( issues++ )) || true
    done

    echo ""
    echo "Total issues: $issues"
    (( issues == 0 ))
}

k8s_check_pvcs() {
    local namespace=${1:-default}

    echo "--- PersistentVolumeClaims ---"
    kubectl get pvc -n "$namespace" -o json 2>/dev/null | \
    jq -r '.items[] | "  " + (if .status.phase=="Bound" then "[OK]   " else "[WARN] " end) + .metadata.name + ": " + .status.phase'
}

k8s_check_services() {
    local namespace=${1:-default}

    echo "--- Services with no Endpoints ---"
    kubectl get endpoints -n "$namespace" -o json 2>/dev/null | \
    jq -r '.items[] | select(.subsets == null or (.subsets | length) == 0) | "  [WARN] " + .metadata.name + ": no endpoints"'
}

k8s_pod_logs_errors() {
    local namespace=${1:-default} lines=${2:-50}

    echo "--- Recent Pod Errors ---"
    kubectl get pods -n "$namespace" -o name 2>/dev/null | \
    while read -r pod; do
        kubectl logs -n "$namespace" "$pod" --tail="$lines" 2>/dev/null | \
            grep -iE 'error|exception|fatal|panic' | \
            head -3 | \
            sed "s/^/  [$pod] /"
    done
}
```

---

## 67.7 Namespace and RBAC Setup

```bash
#!/bin/bash
# k8s_setup.sh - Namespace and RBAC provisioning

k8s_create_namespace() {
    local ns=$1
    local labels=${2:-}

    if kubectl get namespace "$ns" &>/dev/null 2>&1; then
        k8s_log "Namespace exists: $ns"
    else
        kubectl create namespace "$ns"
        k8s_log "Created namespace: $ns"
    fi

    [[ -n "$labels" ]] && kubectl label namespace "$ns" $labels --overwrite
}

k8s_apply_quota() {
    local ns=$1
    local cpu_limit=${2:-4} memory_limit=${3:-8Gi}
    local cpu_request=${4:-1} memory_request=${5:-1Gi}
    local pod_limit=${6:-20}

    kubectl apply -f - <<QUOTA
apiVersion: v1
kind: ResourceQuota
metadata:
  name: default-quota
  namespace: $ns
spec:
  hard:
    requests.cpu: "$cpu_request"
    requests.memory: $memory_request
    limits.cpu: "$cpu_limit"
    limits.memory: $memory_limit
    pods: "$pod_limit"
QUOTA

    k8s_log "Applied quota to namespace: $ns"
}

k8s_create_service_account() {
    local name=$1 ns=${2:-default}

    kubectl create serviceaccount "$name" -n "$ns" --dry-run=client -o yaml | \
        kubectl apply -f -
    k8s_log "ServiceAccount: $name in $ns"
}

k8s_create_role_binding() {
    local sa=$1 role=$2 ns=${3:-default}

    kubectl create rolebinding "${sa}-${role}" \
        --role="$role" \
        --serviceaccount="${ns}:${sa}" \
        -n "$ns" \
        --dry-run=client -o yaml | kubectl apply -f -

    k8s_log "RoleBinding: $sa -> $role in $ns"
}

k8s_setup_namespace() {
    local ns=$1

    k8s_create_namespace "$ns"
    k8s_apply_quota "$ns"
    k8s_create_service_account "app-sa" "$ns"
    k8s_create_role_binding "app-sa" "view" "$ns"
    k8s_log "Namespace setup complete: $ns"
}
```

---

## 67.8 Multi-cluster Management

```bash
#!/bin/bash
# multi_cluster.sh - Multi-cluster kubectl context management

list_clusters() {
    kubectl config get-contexts -o name 2>/dev/null
}

switch_cluster() {
    local context=$1
    kubectl config use-context "$context"
    k8s_log "Switched to cluster: $context"
}

run_on_all_clusters() {
    local cmd=("$@")
    local contexts; readarray -t contexts < <(list_clusters)

    for ctx in "${contexts[@]}"; do
        echo "=== Cluster: $ctx ==="
        kubectl --context="$ctx" "${cmd[@]}" 2>/dev/null || echo "  [FAILED]"
        echo ""
    done
}

compare_clusters() {
    local ctx1=$1 ctx2=$2 namespace=${3:-default}

    echo "=== Deployment Comparison ==="
    printf '%-40s %-20s %-20s\n' 'Deployment' "$ctx1" "$ctx2"
    printf '%-40s %-20s %-20s\n' '----------' '------' '------'

    local deps1 deps2
    deps1=$(kubectl --context="$ctx1" get deploy -n "$namespace" -o jsonpath='{.items[*].metadata.name}' 2>/dev/null)
    deps2=$(kubectl --context="$ctx2" get deploy -n "$namespace" -o jsonpath='{.items[*].metadata.name}' 2>/dev/null)

    for dep in $deps1; do
        local img1 img2
        img1=$(kubectl --context="$ctx1" get deploy "$dep" -n "$namespace" \
            -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null)
        img2=$(kubectl --context="$ctx2" get deploy "$dep" -n "$namespace" \
            -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null)

        local marker=""
        [[ "$img1" != "$img2" ]] && marker="<-- DIFF"

        printf '%-40s %-20s %-20s %s\n' \
            "$dep" \
            "${img1##*:}" \
            "${img2##*:}" \
            "$marker"
    done
}
```

---

## 67.9 Exercises

### Exercise 1: Docker CI Pipeline
สร้าง pipeline ที่:
- Multi-stage Dockerfile
- Layer cache optimization
- Vulnerability scan (trivy)
- Push to registry on main branch

### Exercise 2: Kubernetes GitOps
สร้าง system ที่:
- Watch Git repo for manifest changes
- Auto-apply changed manifests
- Rollback on health check failure
- Slack notification on deploy

### Exercise 3: Cluster Cost Monitor
สร้าง tool ที่:
- Resource request vs limit per namespace
- Idle/underutilized workloads
- Node utilization
- Cost report ($/CPU/Mem)

---

## สรุป Part 67

✅ Docker image build/tag/push with OCI labels; retry pull
✅ Docker cleanup: container prune, image prune, volume prune
✅ Vulnerability scanning via trivy/grype with CRITICAL/HIGH counts
✅ Docker Compose: up/down/restart/scale, healthy wait, exec, status
✅ Container runtime monitoring: stats, unhealthy detection, resource usage
✅ kubectl helpers: apply+wait, rollout watch/undo, set image, port-forward, exec
✅ Helm install-or-upgrade idempotent, rollback, diff, cleanup failed
✅ K8s health: deployments, pods, nodes, PVCs, services, pod error logs
✅ Namespace setup: create, resource quota, service account, role binding
✅ Multi-cluster: list, switch, run-on-all, compare deployments

---

**→ Part 68: Monitoring, Alerting, and Observability**
