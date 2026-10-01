# Part 46: Container and Kubernetes Automation
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 46.1 Docker Automation

```bash
#!/bin/bash
# docker_automation.sh - Docker lifecycle management

# ─── Container Lifecycle ───────────────────────────────────────
docker_run_safe() {
    local name=$1
    local image=$2
    shift 2
    local args=("$@")

    if docker ps -a --format '{{.Names}}' | grep -q "^${name}$"; then
        echo "Removing existing container: $name"
        docker rm -f "$name" 2>/dev/null
    fi

    echo "Starting: $name ($image)"
    docker run --name "$name" "${args[@]}" "$image"
}

container_wait_healthy() {
    local container=$1
    local timeout=${2:-60}
    local start=$SECONDS

    echo -n "Waiting for $container to be healthy..."
    while (( SECONDS - start < timeout )); do
        local health
        health=$(docker inspect --format='{{.State.Health.Status}}' "$container" 2>/dev/null)

        case "$health" in
            healthy)   echo " OK"; return 0 ;;
            unhealthy) echo " UNHEALTHY"; return 1 ;;
            "")
                docker inspect --format='{{.State.Running}}' "$container" 2>/dev/null | \
                    grep -q true && echo " Running (no healthcheck)" && return 0
                ;;
        esac

        echo -n "."
        sleep 2
    done

    echo " TIMEOUT"
    return 1
}

# ─── Docker Compose Helpers ────────────────────────────────────
compose_up() {
    local compose_file=${1:-docker-compose.yml}
    local env_file=${2:-.env}

    local args=(--file "$compose_file")
    [[ -f "$env_file" ]] && args+=(--env-file "$env_file")

    docker compose "${args[@]}" up --detach --remove-orphans
}

compose_down() {
    local compose_file=${1:-docker-compose.yml}
    local volumes=${2:-false}

    local args=(--file "$compose_file" down)
    $volumes && args+=(--volumes)

    docker compose "${args[@]}"
}

compose_logs() {
    local service=${1:-}
    local lines=${2:-100}
    local compose_file=${3:-docker-compose.yml}

    local args=(--file "$compose_file" logs --tail="$lines" --follow)
    [[ -n "$service" ]] && args+=("$service")

    docker compose "${args[@]}"
}

# ─── Image Management ──────────────────────────────────────────
build_image() {
    local tag=$1
    local context=${2:-.}
    local dockerfile=${3:-Dockerfile}
    local build_args=("${@:4}")

    local ba_flags=()
    for arg in "${build_args[@]}"; do
        ba_flags+=(--build-arg "$arg")
    done

    docker build \
        --tag "$tag" \
        --file "${context}/${dockerfile}" \
        --label "build.date=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
        --label "build.version=$(git rev-parse --short HEAD 2>/dev/null || echo 'unknown')" \
        "${ba_flags[@]}" \
        "$context"
}

push_image() {
    local image=$1
    local registry=$2
    local tag=${3:-latest}

    local remote_tag="${registry}/${image}:${tag}"
    docker tag "${image}:${tag}" "$remote_tag"
    docker push "$remote_tag"
    echo "Pushed: $remote_tag"
}

cleanup_images() {
    local keep_days=${1:-7}

    echo "Removing dangling images..."
    docker image prune -f

    echo "Removing unused images older than ${keep_days} days..."
    docker images --format '{{.Repository}}:{{.Tag}} {{.CreatedAt}}' | \
        awk -v cutoff="$(date -d "${keep_days} days ago" +%s 2>/dev/null || date -v-${keep_days}d +%s)" \
        '{cmd="date -d \"" $2 " " $3 "\" +%s 2>/dev/null"; cmd | getline ts; close(cmd); if(ts < cutoff) print $1}' | \
        xargs -r docker rmi 2>/dev/null
}

# ─── Docker Stats Dashboard ────────────────────────────────────
container_stats() {
    echo "=== Container Status ==="
    docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}\t{{.Size}}"

    echo ""
    echo "=== Resource Usage ==="
    docker stats --no-stream --format \
        "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}"
}
```

---

## 46.2 Kubernetes Automation

```bash
#!/bin/bash
# k8s_automation.sh - Kubernetes cluster management

K8S_NAMESPACE="${K8S_NAMESPACE:-default}"
K8S_CONTEXT="${K8S_CONTEXT:-}"

kubectl_cmd() {
    local cmd_args=("$@")
    [[ -n "$K8S_CONTEXT" ]] && cmd_args=(--context "$K8S_CONTEXT" "${cmd_args[@]}")
    kubectl --namespace "$K8S_NAMESPACE" "${cmd_args[@]}"
}

# ─── Pod Management ────────────────────────────────────────────
wait_for_deployment() {
    local deployment=$1
    local timeout=${2:-300}

    echo "Waiting for deployment/$deployment to be ready..."
    kubectl_cmd rollout status deployment/"$deployment" \
        --timeout="${timeout}s"
}

get_pod_logs() {
    local selector=$1
    local container=${2:-}
    local lines=${3:-100}

    local pod
    pod=$(kubectl_cmd get pods -l "$selector" \
        --field-selector=status.phase=Running \
        -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)

    [[ -z "$pod" ]] && { echo "No running pod found for: $selector" >&2; return 1; }

    local args=(logs "$pod" --tail="$lines" --follow=false)
    [[ -n "$container" ]] && args+=(--container "$container")
    kubectl_cmd "${args[@]}"
}

exec_in_pod() {
    local selector=$1
    shift
    local cmd=("$@")

    local pod
    pod=$(kubectl_cmd get pods -l "$selector" \
        --field-selector=status.phase=Running \
        -o jsonpath='{.items[0].metadata.name}')

    kubectl_cmd exec "$pod" -- "${cmd[@]}"
}

pod_health_check() {
    local namespace=${1:-default}

    echo "=== Pod Health ($namespace) ==="
    kubectl --namespace "$namespace" get pods \
        --output=custom-columns='NAME:.metadata.name,STATUS:.status.phase,RESTARTS:.status.containerStatuses[0].restartCount,AGE:.metadata.creationTimestamp' \
        2>/dev/null | sort

    echo ""
    echo "=== Problem Pods ==="
    kubectl --namespace "$namespace" get pods \
        --field-selector='status.phase!=Running,status.phase!=Succeeded' \
        --output=wide 2>/dev/null
}

# ─── Deployment Updates ────────────────────────────────────────
update_image() {
    local deployment=$1
    local container=$2
    local image=$3

    echo "Updating $deployment/$container → $image"
    kubectl_cmd set image "deployment/${deployment}" "${container}=${image}"
    wait_for_deployment "$deployment"
}

rolling_restart() {
    local deployment=$1

    echo "Restarting deployment/$deployment..."
    kubectl_cmd rollout restart "deployment/${deployment}"
    wait_for_deployment "$deployment"
}

scale_deployment() {
    local deployment=$1
    local replicas=$2

    echo "Scaling $deployment to $replicas replicas..."
    kubectl_cmd scale deployment "$deployment" --replicas="$replicas"
    kubectl_cmd rollout status deployment/"$deployment"
}

# ─── Namespace Management ──────────────────────────────────────
create_namespace() {
    local namespace=$1
    local labels=${2:-}

    if kubectl get namespace "$namespace" &>/dev/null; then
        echo "Namespace exists: $namespace"
        return 0
    fi

    kubectl create namespace "$namespace"

    if [[ -n "$labels" ]]; then
        kubectl label namespace "$namespace" $labels
    fi

    echo "Created namespace: $namespace"
}

namespace_resource_quota() {
    local namespace=$1
    local cpu_limit=${2:-4}
    local memory_limit=${3:-8Gi}
    local pod_limit=${4:-20}

    kubectl apply -f - << EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: default-quota
  namespace: ${namespace}
spec:
  hard:
    requests.cpu: "${cpu_limit}"
    requests.memory: "${memory_limit}"
    limits.cpu: "$((cpu_limit * 2))"
    limits.memory: "${memory_limit}"
    pods: "${pod_limit}"
EOF
    echo "Applied resource quota to namespace: $namespace"
}

# ─── ConfigMap & Secret Management ────────────────────────────
create_configmap_from_dir() {
    local name=$1
    local dir=$2

    kubectl_cmd create configmap "$name" \
        --from-file="$dir" \
        --dry-run=client \
        --output=yaml | kubectl_cmd apply -f -
    echo "Applied configmap: $name"
}

create_secret_from_env() {
    local name=$1
    local env_file=$2

    local literal_args=()
    while IFS='=' read -r key value; do
        [[ "$key" =~ ^#|^$ ]] && continue
        literal_args+=(--from-literal="${key}=${value}")
    done < "$env_file"

    kubectl_cmd create secret generic "$name" \
        "${literal_args[@]}" \
        --dry-run=client \
        --output=yaml | kubectl_cmd apply -f -
    echo "Applied secret: $name"
}
```

---

## 46.3 Container Security Scanning

```bash
#!/bin/bash
# container_security.sh - Security scanning automation

scan_image_trivy() {
    local image=$1
    local severity=${2:-HIGH,CRITICAL}
    local format=${3:-table}
    local output_file=${4:-}

    if ! command -v trivy &>/dev/null; then
        echo "trivy not installed. Installing..."
        curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | \
            sh -s -- -b /usr/local/bin
    fi

    local trivy_args=(
        image
        --severity "$severity"
        --format "$format"
        --no-progress
    )

    [[ -n "$output_file" ]] && trivy_args+=(--output "$output_file")

    trivy "${trivy_args[@]}" "$image"
}

check_image_policies() {
    local image=$1
    local max_critical=${2:-0}
    local max_high=${3:-5}

    local scan_result
    scan_result=$(trivy image --format json --no-progress "$image" 2>/dev/null)

    local critical_count high_count
    critical_count=$(echo "$scan_result" | jq '[.Results[].Vulnerabilities[]? | select(.Severity=="CRITICAL")] | length')
    high_count=$(echo "$scan_result" | jq '[.Results[].Vulnerabilities[]? | select(.Severity=="HIGH")] | length')

    echo "Vulnerabilities: CRITICAL=$critical_count HIGH=$high_count"

    local policy_passed=true
    (( critical_count > max_critical )) && { echo "POLICY FAIL: Critical count $critical_count > $max_critical"; policy_passed=false; }
    (( high_count > max_high ))         && { echo "POLICY FAIL: High count $high_count > $max_high"; policy_passed=false; }

    $policy_passed && echo "Policy check: PASSED" || { echo "Policy check: FAILED"; return 1; }
}

docker_bench_check() {
    echo "Running Docker Bench Security..."
    docker run --rm \
        --net=host \
        --pid=host \
        --userns=host \
        --cap-add=audit_control \
        -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
        -v /etc:/etc:ro \
        -v /usr/bin/containerd:/usr/bin/containerd:ro \
        -v /usr/bin/runc:/usr/bin/runc:ro \
        -v /usr/lib/systemd:/usr/lib/systemd:ro \
        -v /var/lib:/var/lib:ro \
        -v /var/run/docker.sock:/var/run/docker.sock:ro \
        docker/docker-bench-security 2>/dev/null || \
        echo "Docker Bench requires privileged access"
}
```

---

## 46.4 Helm Chart Automation

```bash
#!/bin/bash
# helm_automation.sh - Helm chart management

helm_install_or_upgrade() {
    local release=$1
    local chart=$2
    local namespace=${3:-default}
    local values_file=${4:-}
    shift 4
    local set_values=("$@")

    local helm_args=(
        upgrade "$release" "$chart"
        --install
        --namespace "$namespace"
        --create-namespace
        --wait
        --timeout 5m
    )

    [[ -n "$values_file" ]] && helm_args+=(--values "$values_file")

    for kv in "${set_values[@]}"; do
        helm_args+=(--set "$kv")
    done

    echo "Installing/upgrading: $release ($chart) in $namespace"
    helm "${helm_args[@]}"
}

helm_uninstall() {
    local release=$1
    local namespace=${2:-default}
    helm uninstall "$release" --namespace "$namespace"
}

helm_diff_upgrade() {
    local release=$1
    local chart=$2
    local namespace=${3:-default}
    local values_file=${4:-}

    if ! helm plugin list | grep -q diff; then
        helm plugin install https://github.com/databus23/helm-diff
    fi

    local helm_args=(diff upgrade "$release" "$chart" --namespace "$namespace")
    [[ -n "$values_file" ]] && helm_args+=(--values "$values_file")

    helm "${helm_args[@]}"
}

install_cert_manager() {
    local version=${1:-v1.13.0}

    helm repo add jetstack https://charts.jetstack.io --force-update

    helm_install_or_upgrade cert-manager jetstack/cert-manager cert-manager \
        "" \
        "installCRDs=true" \
        "global.leaderElection.namespace=cert-manager"

    echo "cert-manager installed"
}

install_ingress_nginx() {
    local service_type=${1:-LoadBalancer}

    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx --force-update

    helm_install_or_upgrade ingress-nginx ingress-nginx/ingress-nginx ingress-nginx \
        "" \
        "controller.service.type=${service_type}"

    echo "ingress-nginx installed"
}
```

---

## 46.5 Multi-Cluster Management

```bash
#!/bin/bash
# multi_cluster.sh - Manage multiple Kubernetes clusters

declare -A CLUSTERS=()

register_cluster() {
    local name=$1
    local context=$2
    local region=${3:-}
    CLUSTERS["$name"]="${context}:${region}"
}

foreach_cluster() {
    local func=$1
    shift
    local args=("$@")

    for cluster_name in "${!CLUSTERS[@]}"; do
        local info="${CLUSTERS[$cluster_name]}"
        local context="${info%%:*}"
        local region="${info#*:}"

        echo "=== Cluster: $cluster_name ($context) ==="
        K8S_CONTEXT="$context" "$func" "${args[@]}"
    done
}

cluster_status() {
    echo "Nodes:"
    kubectl --context "${K8S_CONTEXT:-}" get nodes \
        --output=custom-columns='NAME:.metadata.name,STATUS:.status.conditions[-1].type,CPU:.status.allocatable.cpu,MEMORY:.status.allocatable.memory'

    echo ""
    echo "Namespaces:"
    kubectl --context "${K8S_CONTEXT:-}" get namespaces \
        --output=custom-columns='NAME:.metadata.name,STATUS:.status.phase'
}

deploy_to_all_clusters() {
    local manifest=$1

    for cluster_name in "${!CLUSTERS[@]}"; do
        local info="${CLUSTERS[$cluster_name]}"
        local context="${info%%:*}"

        echo "Deploying to: $cluster_name ($context)"
        kubectl --context "$context" apply -f "$manifest"
    done
}

# ─── Kubeconfig Management ─────────────────────────────────────
merge_kubeconfigs() {
    local configs=("$@")
    local merged_config="${HOME}/.kube/config.merged"

    KUBECONFIG=$(IFS=':'; echo "${configs[*]}") kubectl config view --flatten > "$merged_config"
    echo "Merged configs → $merged_config"
    echo "Set: export KUBECONFIG=$merged_config"
}

switch_cluster() {
    local cluster_name=$1
    local info="${CLUSTERS[$cluster_name]:-}"

    [[ -z "$info" ]] && { echo "Unknown cluster: $cluster_name" >&2; return 1; }

    local context="${info%%:*}"
    kubectl config use-context "$context"
    export K8S_CONTEXT="$context"
    echo "Switched to cluster: $cluster_name ($context)"
}
```

---

## 46.6 Exercises

### Exercise 1: Auto-Scaling Manager
สร้าง script ที่:
- Monitor CPU/memory metrics
- Auto-scale deployments based on thresholds
- Log scaling events
- Respect min/max replica bounds

### Exercise 2: GitOps Reconciler
สร้าง mini GitOps loop ที่:
- Watch Git repo for changes
- Apply manifests on change
- Report drift detection
- Rollback on failure

### Exercise 3: Multi-Tenant Namespace Provisioner
สร้าง script ที่:
- Create namespace per tenant
- Apply RBAC, quotas, network policies
- Generate kubeconfig for tenant
- Audit tenant usage

---

## สรุป Part 46

✅ Docker container lifecycle management
✅ Docker Compose orchestration helpers
✅ Image build, push, and cleanup automation
✅ Kubernetes deployment management
✅ Pod health checks and log streaming
✅ Namespace creation with resource quotas
✅ ConfigMap/Secret management
✅ Container security scanning (Trivy)
✅ Helm chart install/upgrade/diff
✅ Multi-cluster management framework

---

**→ Part 47: Database Administration Automation**
