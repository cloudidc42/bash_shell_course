# Part 57: Container Orchestration with Bash
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 57.1 Docker Management

```bash
#!/bin/bash
# docker_mgmt.sh - Docker container lifecycle management

DOCKER_REGISTRY="${DOCKER_REGISTRY:-docker.io}"
COMPOSE_FILE="${COMPOSE_FILE:-docker-compose.yml}"

# ─── Image Management ───────────────────────────────────────────
docker_build_image() {
    local name=$1 tag=${2:-latest} context=${3:-.} dockerfile=${4:-Dockerfile}

    echo "Building: ${name}:${tag}"

    docker build \
        --file "$dockerfile" \
        --tag "${name}:${tag}" \
        --build-arg BUILD_DATE="$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
        --build-arg VCS_REF="$(git rev-parse --short HEAD 2>/dev/null || echo 'unknown')" \
        --label "org.opencontainers.image.created=$(date -u '+%Y-%m-%dT%H:%M:%SZ')" \
        --label "org.opencontainers.image.revision=$(git rev-parse HEAD 2>/dev/null || echo '')" \
        "$context"
}

docker_push_image() {
    local name=$1 tag=${2:-latest}
    local full_name="${DOCKER_REGISTRY}/${name}:${tag}"

    docker tag "${name}:${tag}" "$full_name"
    docker push "$full_name"
    echo "Pushed: $full_name"
}

docker_pull_with_retry() {
    local image=$1
    local max_attempts=${2:-3}

    local attempt=1
    while (( attempt <= max_attempts )); do
        if docker pull "$image"; then
            return 0
        fi
        echo "Pull failed (attempt $attempt/$max_attempts)"
        (( attempt++ ))
        sleep $(( attempt * 2 ))
    done
    return 1
}

docker_image_digest() {
    local image=$1
    docker inspect --format '{{index .RepoDigests 0}}' "$image" 2>/dev/null
}

docker_cleanup_images() {
    local keep_days=${1:-7}

    echo "Removing images older than $keep_days days..."

    docker images --format '{{.ID}} {{.CreatedAt}}' | while read -r id created_at; do
        local created_epoch
        created_epoch=$(date -d "$created_at" +%s 2>/dev/null || echo 0)
        local cutoff
        cutoff=$(date -d "$keep_days days ago" +%s 2>/dev/null || echo 0)

        if (( created_epoch < cutoff )); then
            docker rmi "$id" 2>/dev/null && echo "Removed: $id"
        fi
    done

    docker image prune -f
}

# ─── Container Lifecycle ────────────────────────────────────────
container_run() {
    local name=$1 image=$2
    shift 2
    local -A opts=()

    while [[ $# -gt 0 ]]; do
        opts["${1%%=*}"]="${1#*=}"
        shift
    done

    local args=(
        "--name" "$name"
        "--detach"
        "--restart" "${opts[restart]:-unless-stopped}"
    )

    [[ -n "${opts[network]:-}" ]] && args+=("--network" "${opts[network]}")
    [[ -n "${opts[ports]:-}" ]] && {
        IFS=',' read -ra port_list <<< "${opts[ports]}"
        for p in "${port_list[@]}"; do
            args+=("-p" "$p")
        done
    }
    [[ -n "${opts[volumes]:-}" ]] && {
        IFS=',' read -ra vol_list <<< "${opts[volumes]}"
        for v in "${vol_list[@]}"; do
            args+=("-v" "$v")
        done
    }
    [[ -n "${opts[env]:-}" ]] && {
        IFS=',' read -ra env_list <<< "${opts[env]}"
        for e in "${env_list[@]}"; do
            args+=("-e" "$e")
        done
    }
    [[ -n "${opts[memory]:-}" ]] && args+=("--memory" "${opts[memory]}")
    [[ -n "${opts[cpus]:-}" ]] && args+=("--cpus" "${opts[cpus]}")

    docker run "${args[@]}" "$image"
    echo "Started container: $name"
}

container_wait_healthy() {
    local name=$1 timeout=${2:-60}
    local elapsed=0

    echo -n "Waiting for $name to be healthy"

    while (( elapsed < timeout )); do
        local health
        health=$(docker inspect --format '{{.State.Health.Status}}' "$name" 2>/dev/null)

        case "$health" in
            healthy)   echo " OK"; return 0 ;;
            unhealthy) echo " UNHEALTHY"; return 1 ;;
        esac

        echo -n "."
        sleep 2
        (( elapsed += 2 ))
    done

    echo " TIMEOUT"
    return 1
}

container_exec_check() {
    local name=$1
    shift
    local cmd="$*"

    docker exec "$name" bash -c "$cmd"
}

# ─── Container Networking ──────────────────────────────────────
docker_network_create() {
    local name=$1 subnet=${2:-} driver=${3:-bridge}

    local args=("--driver" "$driver")
    [[ -n "$subnet" ]] && args+=("--subnet" "$subnet")

    if ! docker network inspect "$name" &>/dev/null; then
        docker network create "${args[@]}" "$name"
        echo "Created network: $name"
    else
        echo "Network exists: $name"
    fi
}

docker_network_connect() {
    local container=$1 network=$2 alias=${3:-}

    local args=()
    [[ -n "$alias" ]] && args+=("--alias" "$alias")

    docker network connect "${args[@]}" "$network" "$container"
}

# ─── Volume Management ────────────────────────────────────────
docker_volume_backup() {
    local volume=$1 backup_dir=${2:-/tmp/vol_backups}
    local timestamp
    timestamp=$(date +%Y%m%d_%H%M%S)

    mkdir -p "$backup_dir"

    docker run --rm \
        -v "${volume}:/data:ro" \
        -v "${backup_dir}:/backup" \
        alpine tar czf "/backup/${volume}_${timestamp}.tar.gz" -C /data .

    echo "Backed up: $backup_dir/${volume}_${timestamp}.tar.gz"
}

docker_volume_restore() {
    local volume=$1 backup_file=$2

    [[ ! -f "$backup_file" ]] && { echo "Backup not found: $backup_file"; return 1; }

    local backup_dir; backup_dir=$(dirname "$backup_file")
    local backup_name; backup_name=$(basename "$backup_file")

    docker run --rm \
        -v "${volume}:/data" \
        -v "${backup_dir}:/backup:ro" \
        alpine sh -c "cd /data && tar xzf /backup/$backup_name"

    echo "Restored $volume from $backup_file"
}
```

---

## 57.2 Docker Compose Operations

```bash
#!/bin/bash
# compose_ops.sh - Docker Compose orchestration

# ─── Service Operations ──────────────────────────────────────────
compose_deploy() {
    local project=${1:-$(basename "$PWD")}
    local compose_file=${2:-docker-compose.yml}

    echo "Deploying project: $project"

    docker compose \
        --project-name "$project" \
        --file "$compose_file" \
        up \
        --detach \
        --remove-orphans \
        --pull always

    echo "Waiting for services to be healthy..."
    compose_wait_healthy "$project" 120
}

compose_wait_healthy() {
    local project=$1 timeout=${2:-60}
    local elapsed=0

    while (( elapsed < timeout )); do
        local unhealthy
        unhealthy=$(docker compose --project-name "$project" ps \
            --format json 2>/dev/null | grep -c '"Health":"unhealthy"' || echo 0)

        local starting
        starting=$(docker compose --project-name "$project" ps \
            --format json 2>/dev/null | grep -c '"Health":"starting"' || echo 0)

        if (( unhealthy > 0 )); then
            echo "Unhealthy containers detected"
            return 1
        fi

        if (( starting == 0 )); then
            echo "All services healthy"
            return 0
        fi

        echo "Services starting: $starting remaining..."
        sleep 5
        (( elapsed += 5 ))
    done

    echo "Timeout waiting for healthy state"
    return 1
}

compose_rolling_update() {
    local project=$1 service=$2 new_image=$3

    echo "Rolling update: $service -> $new_image"

    local current_scale
    current_scale=$(docker compose --project-name "$project" ps "$service" --quiet | wc -l)
    local new_scale=$(( current_scale + 1 ))

    docker compose --project-name "$project" \
        up --scale "${service}=${new_scale}" --no-recreate --detach "$service"

    sleep 10

    docker compose --project-name "$project" \
        up --scale "${service}=${current_scale}" --no-recreate --detach "$service"

    echo "Rolling update complete"
}
```

---

## 57.3 Container Health Monitoring

```bash
#!/bin/bash
# container_monitor.sh - Container health and resource monitoring

MONITOR_INTERVAL="${MONITOR_INTERVAL:-30}"
ALERT_THRESHOLD_CPU="${ALERT_THRESHOLD_CPU:-90}"
ALERT_THRESHOLD_MEM="${ALERT_THRESHOLD_MEM:-90}"

# ─── Resource Stats ─────────────────────────────────────────────
collect_container_stats() {
    docker stats --no-stream --format \
        '{{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.NetIO}}\t{{.BlockIO}}' \
        2>/dev/null | while IFS=$'\t' read -r name cpu mem_usage mem_pct net_io block_io; do
            echo "${name}|${cpu//%/}|${mem_pct//%/}|${mem_usage}|${net_io}|${block_io}"
        done
}

monitor_containers() {
    echo "=== Container Monitor: $(date) ==="
    printf "%-30s %6s %6s %s\n" "NAME" "CPU%" "MEM%" "STATUS"
    echo "$(printf '%.0s-' {1..60})"

    local alerts=()

    while IFS='|' read -r name cpu mem mem_usage net block; do
        local status="OK"
        local cpu_int="${cpu%%.*}"
        local mem_int="${mem%%.*}"

        (( cpu_int >= ALERT_THRESHOLD_CPU )) && {
            status="CPU HIGH"
            alerts+=("$name: CPU at ${cpu}%")
        }
        (( mem_int >= ALERT_THRESHOLD_MEM )) && {
            status="MEM HIGH"
            alerts+=("$name: Memory at ${mem}%")
        }

        printf "%-30s %6s %6s %s\n" "${name:0:29}" "${cpu}%" "${mem}%" "$status"
    done < <(collect_container_stats)

    if [[ ${#alerts[@]} -gt 0 ]]; then
        echo ""
        echo "=== Alerts ==="
        printf "  %s\n" "${alerts[@]}"
    fi
}

container_resource_history() {
    local container=$1 samples=${2:-10} interval=${3:-2}
    local history_file="/tmp/container_stats_${container}.log"

    for (( i=1; i<=samples; i++ )); do
        docker stats --no-stream --format \
            '{{.CPUPerc}}\t{{.MemPerc}}' "$container" 2>/dev/null | \
            while IFS=$'\t' read -r cpu mem; do
                echo "$(date +%s) ${cpu//%/} ${mem//%/}"
            done >> "$history_file"
        sleep "$interval"
    done

    awk '{
        count++; cpu_sum += $2; mem_sum += $3
        if (NR==1 || $2 > cpu_max) cpu_max = $2
        if (NR==1 || $3 > mem_max) mem_max = $3
    }
    END {
        printf "Samples: %d\n", count
        printf "CPU avg: %.1f%% max: %.1f%%\n", cpu_sum/count, cpu_max
        printf "Mem avg: %.1f%% max: %.1f%%\n", mem_sum/count, mem_max
    }' "$history_file"
}

analyze_container_logs() {
    local container=$1 lines=${2:-1000}

    docker logs --tail "$lines" "$container" 2>&1 | awk '
    BEGIN { errors=0; warnings=0; total=0 }
    { total++
      if (tolower($0) ~ /error|exception|fatal|panic/) errors++
      else if (tolower($0) ~ /warn|warning/) warnings++
    }
    END {
        printf "Total lines: %d\n", total
        printf "Errors: %d (%.1f%%)\n", errors, (total>0 ? errors*100/total : 0)
        printf "Warnings: %d (%.1f%%)\n", warnings, (total>0 ? warnings*100/total : 0)
    }'
}
```

---

## 57.4 Kubernetes with kubectl

```bash
#!/bin/bash
# k8s_ops.sh - Kubernetes operations

KUBECTL_NAMESPACE="${KUBECTL_NAMESPACE:-default}"

# ─── Deployment Management ─────────────────────────────────────
k8s_rollout_deploy() {
    local deployment=$1 image=$2 namespace=${3:-$KUBECTL_NAMESPACE}

    kubectl set image "deployment/$deployment" \
        "${deployment}=$image" \
        --namespace "$namespace"

    kubectl rollout status "deployment/$deployment" \
        --namespace "$namespace" \
        --timeout=300s
}

k8s_rollback() {
    local deployment=$1 revision=${2:-0} namespace=${3:-$KUBECTL_NAMESPACE}

    if (( revision > 0 )); then
        kubectl rollout undo "deployment/$deployment" \
            --to-revision="$revision" --namespace "$namespace"
    else
        kubectl rollout undo "deployment/$deployment" --namespace "$namespace"
    fi

    kubectl rollout status "deployment/$deployment" \
        --namespace "$namespace" --timeout=120s
}

k8s_scale() {
    local deployment=$1 replicas=$2 namespace=${3:-$KUBECTL_NAMESPACE}
    kubectl scale "deployment/$deployment" --replicas="$replicas" --namespace "$namespace"
}

k8s_wait_pods_ready() {
    local selector=$1 namespace=${2:-$KUBECTL_NAMESPACE} timeout=${3:-120}
    kubectl wait pods --selector "$selector" --for condition=Ready \
        --namespace "$namespace" --timeout="${timeout}s"
}

k8s_debug_pod() {
    local pod=$1 namespace=${2:-$KUBECTL_NAMESPACE}

    kubectl describe pod "$pod" --namespace "$namespace"
    kubectl logs "$pod" --namespace "$namespace" --tail=50 2>&1
    kubectl get events --namespace "$namespace" \
        --field-selector "involvedObject.name=$pod" --sort-by='.lastTimestamp'
}

k8s_exec_pod() {
    local selector=$1 namespace=${2:-$KUBECTL_NAMESPACE}
    shift 2
    local cmd="$*"

    local pod
    pod=$(kubectl get pods --selector "$selector" --namespace "$namespace" \
        --field-selector 'status.phase=Running' \
        --output jsonpath='{.items[0].metadata.name}' 2>/dev/null)

    [[ -z "$pod" ]] && { echo "No running pod: $selector"; return 1; }
    kubectl exec "$pod" --namespace "$namespace" -- bash -c "$cmd"
}

k8s_cleanup_completed() {
    local namespace=${1:-$KUBECTL_NAMESPACE}

    kubectl delete jobs --field-selector 'status.conditions[0].type=Complete' \
        --namespace "$namespace" 2>/dev/null
    kubectl delete pods --field-selector 'status.phase=Succeeded' \
        --namespace "$namespace" 2>/dev/null
    kubectl delete pods --field-selector 'status.phase=Failed' \
        --namespace "$namespace" 2>/dev/null
}
```

---

## 57.5 Exercises

### Exercise 1: Zero-Downtime Deployment
สร้าง deployment script ที่:
- Rolling update with health checks
- Automatic rollback on failure
- Zero-downtime via traffic draining

### Exercise 2: Multi-Container App
Deploy stack ที่ประกอบด้วย:
- Web frontend container
- API backend container
- Database container with volume
- Nginx reverse proxy

### Exercise 3: Container Chaos Testing
สร้าง chaos testing framework ที่:
- Kill random containers
- Verify auto-recovery
- Test failover scenarios
- Measure MTTR

---

## สรุป Part 57

✅ Docker image build, push, pull with retry
✅ Container lifecycle: run with resource limits, health wait loop
✅ Docker networking and volume backup/restore
✅ Compose deploy, rolling update, service restart watchdog
✅ Container stats: CPU/memory thresholds with alert detection
✅ Resource history with awk avg/max summary
✅ Log analysis with error/warning rate calculation
✅ Kubernetes: rollout deploy, rollback, scale, pod wait/debug/exec

---

**→ Part 58: Database Operations and Automation**
