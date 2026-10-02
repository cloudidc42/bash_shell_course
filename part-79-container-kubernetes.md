# Part 79: Container and Kubernetes Automation
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 79.1 Docker Helpers

```bash
#!/bin/bash
# docker_helpers.sh - Docker automation functions

set -euo pipefail

docker_build() {
    local tag=$1 context=${2:-.} dockerfile=${3:-Dockerfile}
    local -a build_args=()

    shift 3
    while [[ $# -gt 0 ]]; do
        build_args+=(--build-arg "$1")
        shift
    done

    docker build \
        --file "${context}/${dockerfile}" \
        --tag "$tag" \
        "${build_args[@]}" \
        "$context"
    echo "Built: $tag"
}

docker_run_detach() {
    local name=$1 image=$2
    shift 2
    local -a extra_args=("$@")

    docker run -d \
        --name "$name" \
        --restart unless-stopped \
        "${extra_args[@]}" \
        "$image"
    echo "Started container: $name"
}

docker_exec() {
    local container=$1
    shift
    docker exec -it "$container" "$@"
}

docker_logs_follow() {
    local container=$1 tail=${2:-100}
    docker logs --follow --tail "$tail" "$container"
}

docker_stop_rm() {
    local container=$1
    docker stop "$container" 2>/dev/null || true
    docker rm   "$container" 2>/dev/null || true
    echo "Removed: $container"
}

docker_container_ip() {
    local container=$1 network=${2:-bridge}
    docker inspect "$container" \
        --format "{{.NetworkSettings.Networks.${network}.IPAddress}}" 2>/dev/null
}

docker_container_status() {
    local container=$1
    docker inspect "$container" \
        --format '{{.State.Status}}' 2>/dev/null || echo 'not found'
}

docker_wait_healthy() {
    local container=$1 timeout_sec=${2:-120}
    local deadline=$(( $(date +%s) + timeout_sec ))

    echo -n "Waiting for $container to be healthy"
    while (( $(date +%s) < deadline )); do
        local health; health=$(docker inspect "$container" \
            --format '{{.State.Health.Status}}' 2>/dev/null || echo '')
        if [[ "$health" == "healthy" ]]; then
            echo " OK"
            return 0
        fi
        echo -n '.'
        sleep 2
    done
    echo " TIMEOUT"
    return 1
}

docker_prune_all() {
    echo "Pruning stopped containers..."
    docker container prune -f
    echo "Pruning unused images..."
    docker image prune -f
    echo "Pruning unused volumes..."
    docker volume prune -f
    echo "Pruning unused networks..."
    docker network prune -f
    echo "Prune complete"
}

docker_image_size() {
    local image=$1
    docker image inspect "$image" \
        --format '{{.Size}}' 2>/dev/null | \
        awk '{printf "%.1f MB\n", $1/1024/1024}'
}

docker_ps_table() {
    docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
}
```

---

## 79.2 Docker Compose Helpers

```bash
#!/bin/bash
# compose_helpers.sh - Docker Compose automation

COMPOSE_FILE="${COMPOSE_FILE:-docker-compose.yml}"
COMPOSE_PROJECT="${COMPOSE_PROJECT:-}"

_compose() {
    local -a cmd=(docker compose --file "$COMPOSE_FILE")
    [[ -n "$COMPOSE_PROJECT" ]] && cmd+=(--project-name "$COMPOSE_PROJECT")
    "${cmd[@]}" "$@"
}

compose_up() {
    local -a services=("$@")
    _compose up -d --remove-orphans "${services[@]}"
}

compose_down() {
    _compose down --remove-orphans
}

compose_rebuild() {
    local -a services=("$@")
    _compose up -d --build --remove-orphans "${services[@]}"
}

compose_scale() {
    local service=$1 replicas=$2
    _compose up -d --scale "${service}=${replicas}" "$service"
}

compose_exec() {
    local service=$1
    shift
    _compose exec "$service" "$@"
}

compose_logs() {
    local service=${1:-}
    local tail=${2:-100}
    _compose logs --follow --tail "$tail" ${service:+"$service"}
}

compose_wait_service() {
    local service=$1 check_fn=${2:-} timeout_sec=${3:-120}
    local deadline=$(( $(date +%s) + timeout_sec ))

    echo -n "Waiting for service: $service"
    while (( $(date +%s) < deadline )); do
        local status; status=$(_compose ps --status running --services 2>/dev/null | grep -c "^${service}$" || echo 0)
        if (( status > 0 )); then
            if [[ -z "$check_fn" ]] || "$check_fn" "$service"; then
                echo " OK"
                return 0
            fi
        fi
        echo -n '.'
        sleep 3
    done
    echo " TIMEOUT"
    return 1
}

compose_rolling_restart() {
    local service=$1
    local replicas; replicas=$(_compose ps --quiet "$service" 2>/dev/null | wc -l)

    echo "Rolling restart: $service ($replicas replicas)"
    for (( i=1; i<=replicas; i++ )); do
        _compose restart "$service"
        sleep 5
    done
}
```

---

## 79.3 kubectl Helpers

```bash
#!/bin/bash
# kubectl_helpers.sh - Kubernetes automation via kubectl

KUBECTL_NS="${KUBECTL_NS:-default}"
KUBECTL_CTX="${KUBECTL_CTX:-}"

_kubectl() {
    local -a cmd=(kubectl)
    [[ -n "$KUBECTL_CTX" ]] && cmd+=(--context "$KUBECTL_CTX")
    cmd+=(--namespace "$KUBECTL_NS")
    "${cmd[@]}" "$@"
}

k_get() {
    _kubectl get "$@"
}

k_apply() {
    local manifest=$1
    if [[ -f "$manifest" ]]; then
        _kubectl apply -f "$manifest"
    else
        echo "$manifest" | _kubectl apply -f -
    fi
}

k_delete() {
    _kubectl delete "$@" --ignore-not-found=true
}

k_rollout_wait() {
    local resource=$1 timeout=${2:-300s}
    _kubectl rollout status "$resource" --timeout="$timeout"
}

k_rollout_restart() {
    local deployment=$1
    _kubectl rollout restart deployment/"$deployment"
    k_rollout_wait deployment/"$deployment"
    echo "Rollout complete: $deployment"
}

k_scale() {
    local deployment=$1 replicas=$2
    _kubectl scale deployment/"$deployment" --replicas="$replicas"
    echo "Scaled $deployment to $replicas replicas"
}

k_exec() {
    local pod=$1
    shift
    _kubectl exec -it "$pod" -- "$@"
}

k_logs() {
    local pod=$1 container=${2:-} tail=${3:-100}
    _kubectl logs --tail="$tail" --follow \
        ${container:+--container="$container"} "$pod"
}

k_wait_pod_ready() {
    local label_selector=$1 timeout_sec=${2:-180}
    _kubectl wait pod \
        --for=condition=Ready \
        --selector="$label_selector" \
        --timeout="${timeout_sec}s"
}

k_wait_deployment() {
    local name=$1 timeout_sec=${2:-300}
    _kubectl rollout status deployment/"$name" --timeout="${timeout_sec}s"
}

k_get_pod_by_label() {
    local selector=$1
    _kubectl get pods --selector="$selector" \
        --output=jsonpath='{.items[0].metadata.name}' 2>/dev/null
}

k_describe_failed_pods() {
    echo "=== Failed/Pending Pods ==="
    _kubectl get pods --field-selector='status.phase!=Running,status.phase!=Succeeded' \
        --output=wide 2>/dev/null
}

k_resource_usage() {
    echo "=== Node Resource Usage ==="
    _kubectl top nodes 2>/dev/null || echo "metrics-server not available"
    echo ""
    echo "=== Pod Resource Usage ==="
    _kubectl top pods --sort-by=memory 2>/dev/null || echo "metrics-server not available"
}

k_copy_to_pod() {
    local local_path=$1 pod=$2 pod_path=$3 container=${4:-}
    _kubectl cp "$local_path" "${pod}:${pod_path}" \
        ${container:+--container="$container"}
}

k_copy_from_pod() {
    local pod=$1 pod_path=$2 local_path=$3 container=${4:-}
    _kubectl cp "${pod}:${pod_path}" "$local_path" \
        ${container:+--container="$container"}
}

k_switch_namespace() {
    local ns=$1
    KUBECTL_NS=$ns
    echo "Switched to namespace: $ns"
}

k_switch_context() {
    local ctx=$1
    kubectl config use-context "$ctx"
    KUBECTL_CTX=$ctx
    echo "Switched to context: $ctx"
}

k_list_contexts() {
    kubectl config get-contexts
}
```

---

## 79.4 Helm Helpers

```bash
#!/bin/bash
# helm_helpers.sh - Helm chart operations

HELM_NS="${HELM_NS:-default}"

_helm() {
    helm --namespace "$HELM_NS" "$@"
}

helm_install() {
    local release=$1 chart=$2 values_file=${3:-}
    shift 3
    local extra_args=("$@")

    local -a cmd=(_helm install "$release" "$chart")
    [[ -n "$values_file" ]] && cmd+=(--values "$values_file")
    cmd+=(--atomic --wait --timeout 5m "${extra_args[@]}")

    "${cmd[@]}"
    echo "Installed release: $release"
}

helm_upgrade() {
    local release=$1 chart=$2 values_file=${3:-}
    shift 3
    local extra_args=("$@")

    local -a cmd=(_helm upgrade "$release" "$chart" --install)
    [[ -n "$values_file" ]] && cmd+=(--values "$values_file")
    cmd+=(--atomic --wait --timeout 5m "${extra_args[@]}")

    "${cmd[@]}"
    echo "Upgraded release: $release"
}

helm_rollback() {
    local release=$1 revision=${2:-0}
    _helm rollback "$release" "$revision"
    echo "Rolled back: $release to revision ${revision:-previous}"
}

helm_status() {
    local release=$1
    _helm status "$release" 2>/dev/null
}

helm_uninstall() {
    local release=$1
    _helm uninstall "$release" --wait 2>/dev/null || true
    echo "Uninstalled: $release"
}

helm_list_releases() {
    _helm list --all-namespaces 2>/dev/null
}

helm_get_values() {
    local release=$1
    _helm get values "$release" 2>/dev/null
}

helm_diff() {
    local release=$1 chart=$2 values_file=${3:-}
    _helm diff upgrade "$release" "$chart" \
        ${values_file:+--values "$values_file"} 2>/dev/null || \
    echo "helm-diff plugin not installed"
}

helm_add_repo() {
    local name=$1 url=$2
    helm repo add "$name" "$url"
    helm repo update
    echo "Added repo: $name ($url)"
}
```

---

## 79.5 Kubernetes Deployment Workflow

```bash
#!/bin/bash
# k8s_deploy.sh - Full Kubernetes deployment workflow

deploy_to_k8s() {
    local app=$1 image=$2 tag=$3
    local manifest_dir=${4:-./k8s}
    local namespace=${5:-default}

    KUBECTL_NS=$namespace

    echo "=== Deploying $app:$tag to namespace $namespace ==="

    # 1. Update image tag in manifests (using sed)
    local tmp_dir; tmp_dir=$(mktemp -d)
    cp -r "$manifest_dir"/* "$tmp_dir"/

    find "$tmp_dir" -name '*.yaml' -o -name '*.yml' | \
    while read -r f; do
        sed -i "s|${image}:[^[:space:]]*|${image}:${tag}|g" "$f"
    done

    # 2. Apply manifests
    echo "Applying manifests..."
    k_apply "$tmp_dir"
    rm -rf "$tmp_dir"

    # 3. Wait for rollout
    echo "Waiting for rollout..."
    k_rollout_wait deployment/"$app" 300s

    # 4. Verify pods running
    local ready; ready=$(k_get pods \
        --selector="app=$app" \
        --field-selector='status.phase=Running' \
        --output=jsonpath='{.items}' 2>/dev/null | jq 'length' 2>/dev/null || echo 0)

    echo "Running pods: $ready"
    (( ready > 0 ))
}

k8s_smoke_test() {
    local svc=$1 path=${2:-/health} expected_code=${3:-200}
    local namespace=${KUBECTL_NS:-default}

    # Port-forward for testing
    local local_port=$(( RANDOM % 10000 + 30000 ))
    kubectl port-forward --namespace "$namespace" \
        service/"$svc" "${local_port}:80" &
    local pf_pid=$!
    sleep 2

    local http_code
    http_code=$(curl -s -o /dev/null -w '%{http_code}' \
        "http://127.0.0.1:${local_port}${path}" 2>/dev/null || echo 0)

    kill "$pf_pid" 2>/dev/null || true

    if [[ "$http_code" == "$expected_code" ]]; then
        echo "Smoke test passed: $svc$path ($http_code)"
        return 0
    else
        echo "Smoke test FAILED: $svc$path (expected $expected_code, got $http_code)" >&2
        return 1
    fi
}

k8s_namespace_create() {
    local ns=$1
    kubectl create namespace "$ns" --dry-run=client -o yaml | kubectl apply -f -
    echo "Namespace ready: $ns"
}

k8s_secret_create() {
    local name=$1 namespace=${2:-default}
    shift 2
    local -a literals=("$@")

    local -a cmd=(kubectl create secret generic "$name" --namespace "$namespace")
    for kv in "${literals[@]}"; do
        cmd+=(--from-literal="$kv")
    done
    cmd+=(--dry-run=client -o yaml)

    "${cmd[@]}" | kubectl apply -f -
    echo "Secret created/updated: $name"
}

k8s_configmap_from_dir() {
    local name=$1 dir=$2 namespace=${3:-default}
    kubectl create configmap "$name" \
        --from-file="$dir" \
        --namespace "$namespace" \
        --dry-run=client -o yaml | kubectl apply -f -
    echo "ConfigMap created/updated: $name"
}
```

---

## 79.6 Exercises

### Exercise 1: Zero-downtime Deploy
สร้าง script ที่:
- Build image พร้อม tag
- Push to registry
- Rolling update deployment
- Health check หลัง deploy
- Auto-rollback ถ้า health fail

### Exercise 2: Multi-env Promotion
สร้าง promotion pipeline:
- dev → staging → production
- ต้อง smoke test ผ่านก่อนโปรมโอต
- Helm values แตกต่างกันตาม environment

### Exercise 3: Cluster Health Report
สร้าง report ที่:
- Pod status ทุก namespace
- OOMKilled / CrashLoopBackOff detection
- Resource usage vs requests/limits
- Export to JSON สำหรับ dashboard

---

## สรุป Part 79

✅ Docker: build, run detach, exec, logs, stop/rm, IP, wait_healthy, prune, ps_table
┅ Compose: up/down/rebuild, scale, exec, logs, wait_service, rolling_restart
┅ kubectl: get/apply/delete, rollout wait/restart, scale, exec, logs, wait_pod_ready
┅ kubectl: get_pod_by_label, describe_failed, resource_usage, copy to/from pod
┅ kubectl: switch_namespace/context, list_contexts
┅ Helm: install/upgrade (atomic+wait), rollback, status, uninstall, diff, add_repo
┅ k8s_deploy: patch image tag, apply manifests, rollout wait, pod count verify
┅ k8s_smoke_test: port-forward → curl → expected HTTP code
┅ namespace create, secret create, configmap from dir (idempotent apply)

---

**→ Part 80: CI/CD Pipeline Scripting**
