# Part 69: Infrastructure as Code with Bash
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 69.1 Idempotent Resource Management

```bash
#!/bin/bash
# iac_core.sh - Infrastructure as Code primitives

set -euo pipefail

IAC_STATE_FILE="${IAC_STATE_FILE:-/var/lib/iac/state.json}"
IAC_LOG="${IAC_LOG:-/var/log/iac.log}"

mkdir -p "$(dirname "$IAC_STATE_FILE")" "$(dirname "$IAC_LOG")"
[[ -f "$IAC_STATE_FILE" ]] || echo '{}' > "$IAC_STATE_FILE"

iac_log() {
    printf '%s [IAC] %s\n' "$(date '+%Y-%m-%dT%H:%M:%S')" "$1" | tee -a "$IAC_LOG"
}

# ─── State Management ──────────────────────────────────────────────────
state_get() {
    local resource_id=$1 key=${2:-.}
    jq -r ".\"${resource_id}\"${key}" "$IAC_STATE_FILE" 2>/dev/null
}

state_set() {
    local resource_id=$1 data=$2
    local tmp; tmp=$(mktemp)
    jq ".\"${resource_id}\" = $data" "$IAC_STATE_FILE" > "$tmp"
    mv "$tmp" "$IAC_STATE_FILE"
}

state_delete() {
    local resource_id=$1
    local tmp; tmp=$(mktemp)
    jq "del(.\"${resource_id}\")" "$IAC_STATE_FILE" > "$tmp"
    mv "$tmp" "$IAC_STATE_FILE"
}

state_list() {
    jq -r 'keys[]' "$IAC_STATE_FILE"
}

# ─── Resource Declaration Pattern ───────────────────────────────────────────
declare_resource() {
    local type=$1 name=$2
    shift 2
    local props="{}"

    while [[ $# -ge 2 ]]; do
        props=$(echo "$props" | jq --arg k "$1" --arg v "$2" '. + {($k): $v}')
        shift 2
    done

    echo "$props" | jq --arg type "$type" --arg name "$name" \
        '. + {"_type":$type,"_name":$name}'
}

plan_resource() {
    local resource_id=$1 desired_json=$2

    local current; current=$(state_get "$resource_id" 2>/dev/null || echo 'null')

    if [[ "$current" == "null" ]]; then
        echo "  + CREATE  $resource_id"
    elif [[ "$current" == "$desired_json" ]]; then
        echo "  ~ UNCHANGED $resource_id"
    else
        echo "  ~ UPDATE  $resource_id"
        diff <(echo "$current" | jq -S .) <(echo "$desired_json" | jq -S .) || true
    fi
}

apply_resource() {
    local resource_id=$1 desired_json=$2
    local apply_func=${3:-}

    local current; current=$(state_get "$resource_id" 2>/dev/null || echo 'null')

    if [[ "$current" == "$desired_json" ]]; then
        iac_log "No change: $resource_id"
        return 0
    fi

    if [[ -n "$apply_func" ]]; then
        "$apply_func" "$resource_id" "$desired_json" "$current"
    fi

    state_set "$resource_id" "$desired_json"
    iac_log "Applied: $resource_id"
}
```

---

## 69.2 Terraform Wrapper

```bash
#!/bin/bash
# terraform_ops.sh - Terraform lifecycle management

TF_DIR="${TF_DIR:-./terraform}"
TF_VARS_FILE="${TF_VARS_FILE:-terraform.tfvars}"
TF_LOG_FILE="${TF_LOG:-/tmp/terraform.log}"

tf_log() {
    printf '%s [TF] %s\n' "$(date '+%Y-%m-%dT%H:%M:%S')" "$1" | tee -a "$TF_LOG_FILE"
}

tf_init() {
    local backend_config=${1:-}

    tf_log "Initializing Terraform in $TF_DIR"

    local init_args=(-input=false -no-color)
    [[ -n "$backend_config" ]] && init_args+=(-backend-config="$backend_config")

    terraform -chdir="$TF_DIR" init "${init_args[@]}"
    tf_log "Init complete"
}

tf_validate() {
    tf_log "Validating configuration"
    terraform -chdir="$TF_DIR" validate -no-color
    tf_log "Validation passed"
}

tf_plan() {
    local environment=${1:-default} output_file=${2:-/tmp/tf.plan}
    local var_file="${TF_DIR}/env/${environment}.tfvars"

    tf_log "Planning: $environment"

    local plan_args=(-input=false -no-color -out="$output_file")
    [[ -f "$var_file" ]] && plan_args+=(-var-file="$var_file")
    [[ -f "${TF_DIR}/${TF_VARS_FILE}" ]] && plan_args+=(-var-file="${TF_DIR}/${TF_VARS_FILE}")

    terraform -chdir="$TF_DIR" plan "${plan_args[@]}" 2>&1 | tee -a "$TF_LOG_FILE"
    tf_log "Plan saved: $output_file"
    echo "$output_file"
}

tf_apply() {
    local plan_file=${1:-}
    local auto_approve=${2:-false}

    tf_log "Applying infrastructure"

    local apply_args=(-no-color)
    [[ "$auto_approve" == true ]] && apply_args+=(-auto-approve)

    if [[ -n "$plan_file" ]]; then
        terraform -chdir="$TF_DIR" apply "${apply_args[@]}" "$plan_file"
    else
        terraform -chdir="$TF_DIR" apply "${apply_args[@]}"
    fi

    tf_log "Apply complete"
}

tf_destroy() {
    local environment=${1:-default}
    local auto_approve=${2:-false}

    tf_log "DESTROY requested: $environment"

    if [[ "$auto_approve" != true ]]; then
        read -r -p "Type 'yes' to confirm destroy of $environment: " confirm
        [[ "$confirm" != "yes" ]] && { tf_log "Destroy cancelled"; return 1; }
    fi

    local destroy_args=(-no-color)
    [[ "$auto_approve" == true ]] && destroy_args+=(-auto-approve)

    terraform -chdir="$TF_DIR" destroy "${destroy_args[@]}"
    tf_log "Destroy complete: $environment"
}

tf_output() {
    local key=${1:-}
    if [[ -n "$key" ]]; then
        terraform -chdir="$TF_DIR" output -raw "$key" 2>/dev/null
    else
        terraform -chdir="$TF_DIR" output -json 2>/dev/null
    fi
}

tf_import() {
    local resource_addr=$1 resource_id=$2
    tf_log "Importing: $resource_addr = $resource_id"
    terraform -chdir="$TF_DIR" import "$resource_addr" "$resource_id"
}
```

---

## 69.3 AWS Infrastructure Automation

```bash
#!/bin/bash
# aws_infra.sh - AWS resource management

AWS_REGION="${AWS_REGION:-us-east-1}"

aws_log() {
    printf '%s [AWS] %s\n' "$(date '+%Y-%m-%dT%H:%M:%S')" "$1"
}

# ─── EC2 ─────────────────────────────────────────────────────────────────
ec2_create_instance() {
    local name=$1 ami=$2 type=${3:-t3.micro}
    local subnet_id=${4:-} sg_id=${5:-} key_name=${6:-}
    local user_data_file=${7:-}

    aws_log "Creating EC2: $name ($type, $ami)"

    local run_args=(
        --image-id "$ami"
        --instance-type "$type"
        --region "$AWS_REGION"
        --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$name}]"
    )

    [[ -n "$subnet_id" ]] && run_args+=(--subnet-id "$subnet_id")
    [[ -n "$sg_id" ]] && run_args+=(--security-group-ids "$sg_id")
    [[ -n "$key_name" ]] && run_args+=(--key-name "$key_name")
    [[ -n "$user_data_file" ]] && run_args+=(--user-data "file://${user_data_file}")

    local instance_id
    instance_id=$(aws ec2 run-instances "${run_args[@]}" \
        --query 'Instances[0].InstanceId' --output text)

    aws_log "Created EC2: $instance_id ($name)"

    # Wait for running
    aws ec2 wait instance-running --instance-ids "$instance_id" --region "$AWS_REGION"
    aws_log "EC2 running: $instance_id"

    echo "$instance_id"
}

ec2_get_instance_by_name() {
    local name=$1
    aws ec2 describe-instances \
        --region "$AWS_REGION" \
        --filters "Name=tag:Name,Values=$name" "Name=instance-state-name,Values=running" \
        --query 'Reservations[0].Instances[0].InstanceId' \
        --output text 2>/dev/null
}

ec2_wait_for_ssh() {
    local ip=$1 timeout=${2:-120}
    local deadline=$(( $(date +%s) + timeout ))

    aws_log "Waiting for SSH on $ip"
    while (( $(date +%s) < deadline )); do
        if timeout 3 bash -c "echo >/dev/tcp/${ip}/22" 2>/dev/null; then
            aws_log "SSH ready: $ip"
            return 0
        fi
        sleep 5
    done
    aws_log "ERROR: SSH timeout for $ip"
    return 1
}

# ─── VPC ─────────────────────────────────────────────────────────────────
create_vpc_with_subnets() {
    local name=$1 cidr=${2:-10.0.0.0/16}

    aws_log "Creating VPC: $name ($cidr)"

    local vpc_id
    vpc_id=$(aws ec2 create-vpc \
        --cidr-block "$cidr" \
        --region "$AWS_REGION" \
        --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=$name}]" \
        --query 'Vpc.VpcId' --output text)

    aws ec2 modify-vpc-attribute --vpc-id "$vpc_id" --enable-dns-hostnames \
        --region "$AWS_REGION"

    # Create Internet Gateway
    local igw_id
    igw_id=$(aws ec2 create-internet-gateway \
        --region "$AWS_REGION" \
        --query 'InternetGateway.InternetGatewayId' --output text)
    aws ec2 attach-internet-gateway --vpc-id "$vpc_id" --internet-gateway-id "$igw_id" \
        --region "$AWS_REGION"

    # Create public subnet
    local subnet_id
    subnet_id=$(aws ec2 create-subnet \
        --vpc-id "$vpc_id" \
        --cidr-block "${cidr/0\/16/0/24}" \
        --region "$AWS_REGION" \
        --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=${name}-public}]" \
        --query 'Subnet.SubnetId' --output text)

    aws_log "VPC ready: $vpc_id  subnet: $subnet_id"
    echo "{\"vpc_id\":\"$vpc_id\",\"subnet_id\":\"$subnet_id\",\"igw_id\":\"$igw_id\"}"
}

# ─── S3 ─────────────────────────────────────────────────────────────────
create_s3_bucket() {
    local bucket_name=$1 region=${2:-$AWS_REGION}
    local versioning=${3:-false} encryption=${4:-true}

    # Check if exists
    if aws s3api head-bucket --bucket "$bucket_name" 2>/dev/null; then
        aws_log "Bucket exists: $bucket_name"
        echo "$bucket_name"
        return 0
    fi

    aws_log "Creating S3 bucket: $bucket_name"

    if [[ "$region" == "us-east-1" ]]; then
        aws s3api create-bucket --bucket "$bucket_name" --region "$region"
    else
        aws s3api create-bucket --bucket "$bucket_name" --region "$region" \
            --create-bucket-configuration LocationConstraint="$region"
    fi

    # Block public access
    aws s3api put-public-access-block --bucket "$bucket_name" \
        --public-access-block-configuration \
        BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

    # Versioning
    [[ "$versioning" == true ]] && \
        aws s3api put-bucket-versioning --bucket "$bucket_name" \
            --versioning-configuration Status=Enabled

    # Encryption
    [[ "$encryption" == true ]] && \
        aws s3api put-bucket-encryption --bucket "$bucket_name" \
            --server-side-encryption-configuration \
            '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

    aws_log "S3 bucket created: $bucket_name"
    echo "$bucket_name"
}
```

---

## 69.4 Configuration Management

```bash
#!/bin/bash
# config_mgmt.sh - Configuration file management

CONFIG_TEMPLATE_DIR="${CONFIG_TEMPLATE_DIR:-./templates}"
CONFIG_OUTPUT_DIR="${CONFIG_OUTPUT_DIR:-/etc/app}"

render_template() {
    local template=$1 output=$2
    shift 2
    local env_vars=("$@")

    # Export variables for envsubst
    for var in "${env_vars[@]}"; do
        export "$var"
    done

    local output_dir; output_dir=$(dirname "$output")
    mkdir -p "$output_dir"

    envsubst < "$template" > "$output"
    iac_log "Rendered: $template -> $output"

    # Unset
    for var in "${env_vars[@]}"; do
        unset "${var%%=*}"
    done
}

config_apply_template() {
    local template=$1 output=$2
    local backup=true

    if [[ -f "$output" && "$backup" == true ]]; then
        cp "$output" "${output}.bak.$(date +%Y%m%d%H%M%S)"
    fi

    render_template "$template" "$output"

    if command -v diff &>/dev/null && [[ -f "${output}.bak."* ]]; then
        local latest_bak; latest_bak=$(ls -t "${output}.bak."* 2>/dev/null | head -1)
        if [[ -n "$latest_bak" ]]; then
            diff "$latest_bak" "$output" && iac_log "No change: $output" || iac_log "Updated: $output"
        fi
    fi
}

detect_config_drift() {
    local template_dir=$1 target_dir=$2

    echo "=== Configuration Drift Report ==="
    local drifted=0

    for template in "$template_dir"/*.conf "$template_dir"/*.yaml "$template_dir"/*.yml; do
        [[ -f "$template" ]] || continue
        local base; base=$(basename "$template")
        local target="${target_dir}/${base}"

        if [[ ! -f "$target" ]]; then
            echo "  MISSING: $base"
            (( drifted++ ))
            continue
        fi

        # Compare after rendering
        local rendered; rendered=$(mktemp)
        envsubst < "$template" > "$rendered"

        if ! diff -q "$rendered" "$target" &>/dev/null; then
            echo "  DRIFTED: $base"
            (( drifted++ ))
        fi

        rm -f "$rendered"
    done

    echo ""
    echo "Total drifted: $drifted"
    (( drifted == 0 ))
}

apply_all_configs() {
    local template_dir=${1:-$CONFIG_TEMPLATE_DIR}
    local output_dir=${2:-$CONFIG_OUTPUT_DIR}

    for template in "$template_dir"/*.conf "$template_dir"/*.yaml; do
        [[ -f "$template" ]] || continue
        local output="${output_dir}/$(basename "$template")"
        config_apply_template "$template" "$output"
    done
}
```

---

## 69.5 Cloud-init and User Data

```bash
#!/bin/bash
# cloud_init.sh - Generate cloud-init user data scripts

generate_cloud_init() {
    local hostname=$1 packages=${2:-}
    local ssh_authorized_key=${3:-} deploy_key=${4:-}

    cat <<CLOUDINIT
#!/bin/bash
set -euo pipefail

# Set hostname
hostnamectl set-hostname '$hostname' 2>/dev/null || hostname '$hostname'
echo '127.0.1.1 $hostname' >> /etc/hosts

# Update system
export DEBIAN_FRONTEND=noninteractive
apt-get update -qq
apt-get upgrade -y -qq

# Install packages
$(if [[ -n "$packages" ]]; then
    echo "apt-get install -y -qq $packages"
fi)

# Add SSH key
$(if [[ -n "$ssh_authorized_key" ]]; then
    cat <<SSHEOF
mkdir -p /root/.ssh
chmod 700 /root/.ssh
echo '$ssh_authorized_key' >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
SSHEOF
fi)

# Configure sysctl
cat >> /etc/sysctl.d/99-app.conf <<SYSCTL
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.ip_local_port_range = 1024 65535
vm.swappiness = 10
SYSCTL
sysctl -p /etc/sysctl.d/99-app.conf

# Signal completion
touch /tmp/cloud-init-complete
echo "Cloud-init complete: $hostname at \$(date)" >> /var/log/cloud-init-custom.log
CLOUDINIT
}

generate_ansible_inventory() {
    local resource_db=$1 env=${2:-all}

    echo "[${env}]"
    jq -r --arg env "$env" \
        '.[] | select(.env == $env or $env == "all") | .ip + " ansible_host=" + .ip + " ansible_user=" + (.user // "ubuntu") + " hostname=" + .name' \
        "$resource_db" 2>/dev/null

    echo ""
    echo "[${env}:vars]"
    echo "ansible_ssh_private_key_file=~/.ssh/deploy_key"
    echo "ansible_python_interpreter=/usr/bin/python3"
}
```

---

## 69.6 SSH and Bastion Management

```bash
#!/bin/bash
# ssh_mgmt.sh - SSH connectivity and tunneling

SSH_OPTS=(
    -o StrictHostKeyChecking=no
    -o ConnectTimeout=10
    -o BatchMode=yes
    -o ServerAliveInterval=60
)

ssh_run() {
    local host=$1 user=${2:-ubuntu} key=${3:-~/.ssh/id_rsa}
    shift 3
    local cmd=("$@")

    ssh "${SSH_OPTS[@]}" -i "$key" "${user}@${host}" "${cmd[@]}"
}

ssh_copy() {
    local src=$1 dest_host=$2 dest_path=$3
    local user=${4:-ubuntu} key=${5:-~/.ssh/id_rsa}

    scp -i "$key" "${SSH_OPTS[@]}" "$src" "${user}@${dest_host}:${dest_path}"
}

bastion_tunnel() {
    local bastion_host=$1 target_host=$2 target_port=$3
    local local_port=$4 bastion_user=${5:-ubuntu}
    local pid_file="/tmp/tunnel_${local_port}.pid"

    ssh "${SSH_OPTS[@]}" \
        -N -L "${local_port}:${target_host}:${target_port}" \
        "${bastion_user}@${bastion_host}" &

    echo $! > "$pid_file"
    iac_log "Tunnel: localhost:${local_port} -> ${target_host}:${target_port} via $bastion_host (pid: $!)"
    sleep 2
}

bastion_close_tunnel() {
    local local_port=$1
    local pid_file="/tmp/tunnel_${local_port}.pid"

    if [[ -f "$pid_file" ]]; then
        kill "$(cat "$pid_file")" 2>/dev/null || true
        rm -f "$pid_file"
        iac_log "Closed tunnel on port $local_port"
    fi
}

ssh_run_parallel() {
    local hosts_file=$1 user=${2:-ubuntu} key=${3:-~/.ssh/id_rsa}
    shift 3
    local cmd=("$@")

    local -a pids=()

    while IFS= read -r host; do
        [[ -z "$host" || "$host" =~ ^# ]] && continue
        (
            echo "=== $host ==="
            ssh_run "$host" "$user" "$key" "${cmd[@]}" 2>&1 || echo "  [FAILED: $host]"
        ) &
        pids+=($!)
    done < "$hosts_file"

    for pid in "${pids[@]}"; do wait "$pid" 2>/dev/null || true; done
}
```

---

## 69.7 Multi-environment Configuration

```bash
#!/bin/bash
# env_config.sh - Multi-environment configuration with layering

CONFIG_BASE_DIR="${CONFIG_BASE_DIR:-./config}"

load_config() {
    local environment=${1:-development}
    local config={}

    # Layer: base defaults
    local base_file="${CONFIG_BASE_DIR}/base.json"
    [[ -f "$base_file" ]] && config=$(jq -s '.[0] * .[1]' <(echo "$config") "$base_file")

    # Layer: environment overrides
    local env_file="${CONFIG_BASE_DIR}/${environment}.json"
    [[ -f "$env_file" ]] && config=$(jq -s '.[0] * .[1]' <(echo "$config") "$env_file")

    # Layer: local overrides (never committed)
    local local_file="${CONFIG_BASE_DIR}/local.json"
    [[ -f "$local_file" ]] && config=$(jq -s '.[0] * .[1]' <(echo "$config") "$local_file")

    echo "$config"
}

config_get() {
    local environment=$1 key=$2
    load_config "$environment" | jq -r ".$key // empty"
}

config_diff_envs() {
    local env1=$1 env2=$2

    diff \
        <(load_config "$env1" | jq -S .) \
        <(load_config "$env2" | jq -S .) \
        || echo "Environments differ"
}

export_env_vars() {
    local environment=$1 prefix=${2:-APP_}

    load_config "$environment" | jq -r 'to_entries[] | .key + "=" + (.value | tostring)' | \
    while IFS='=' read -r key value; do
        export "${prefix}${key^^}=${value}"
        echo "${prefix}${key^^}=${value}"
    done
}
```

---

## 69.8 Exercises

### Exercise 1: Cloud Provisioner
สร้าง provisioner ที่:
- EC2 + RDS + ElastiCache stack
- Automatic security group rules
- DNS record creation
- Slack notification on complete

### Exercise 2: Config Drift Monitor
สร้าง monitor ที่:
- Compare running config vs desired
- Auto-remediate simple drifts
- Alert on complex drifts
- Config history tracking

### Exercise 3: Self-healing Infrastructure
สร้าง system ที่:
- Health check all resources
- Auto-restart failed services
- Replace terminated instances
- Scale based on metrics

---

## สรุป Part 69

✅ Idempotent resource management: JSON state file, plan/apply/diff pattern
✅ Terraform wrapper: init/validate/plan/apply/destroy with env layering
✅ AWS EC2: create instance, wait-running, wait-SSH, name-based lookup
✅ AWS VPC: create VPC, IGW, subnet, public access block
┅ AWS S3: create bucket, versioning, encryption, public access block
✅ Configuration rendering: envsubst templates, drift detection, backup on apply
✅ Cloud-init user-data generation with hostname/packages/SSH keys/sysctl
✅ Ansible inventory generation from JSON resource database
✅ SSH helpers: run/copy/parallel; bastion tunnel open/close
✅ Multi-environment config layering: base + env + local override, env var export

---

**→ Part 70: Advanced Shell Scripting Patterns**
