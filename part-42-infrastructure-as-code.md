# Part 42: Infrastructure as Code with Bash
## หลักสูตร Bash/Shell Script ระดับ Advanced

---

## 42.1 Terraform Automation

```bash
#!/bin/bash
# terraform_automation.sh

set -euo pipefail

TF_DIR="${TF_DIR:-./terraform}"
TF_WORKSPACE="${TF_WORKSPACE:-default}"
TF_BACKEND_BUCKET="${TF_BACKEND_BUCKET:-}"

tf_init() {
    local dir=${1:-$TF_DIR}
    echo "=== Terraform Init: $dir ==="

    local args=(-input=false -no-color)

    [[ -n "$TF_BACKEND_BUCKET" ]] && args+=(
        -backend-config="bucket=${TF_BACKEND_BUCKET}"
        -backend-config="key=${TF_WORKSPACE}/terraform.tfstate"
        -backend-config="region=${AWS_REGION:-us-east-1}"
    )

    terraform -chdir="$dir" init "${args[@]}"
    terraform -chdir="$dir" workspace select "$TF_WORKSPACE" 2>/dev/null || \
        terraform -chdir="$dir" workspace new "$TF_WORKSPACE"
    echo "Workspace: $TF_WORKSPACE"
}

tf_plan() {
    local dir=${1:-$TF_DIR}
    local plan_file="${dir}/tfplan"
    local vars_file="${dir}/vars/${TF_WORKSPACE}.tfvars"

    echo "=== Terraform Plan ==="

    local args=(-input=false -no-color -out="$plan_file")
    [[ -f "$vars_file" ]] && args+=(-var-file="$vars_file")

    terraform -chdir="$dir" plan "${args[@]}" | tee /tmp/tf_plan.log

    local changes destroys
    changes=$(grep -oP '\d+ to add' /tmp/tf_plan.log || echo "0 to add")
    destroys=$(grep -oP '\d+ to destroy' /tmp/tf_plan.log || echo "0 to destroy")

    echo ""
    echo "Plan: $changes, $destroys"

    if echo "$destroys" | grep -qv "^0"; then
        echo "WARNING: Resources will be destroyed!"
        return 2
    fi
}

tf_apply() {
    local dir=${1:-$TF_DIR}
    local plan_file="${dir}/tfplan"
    local auto_approve=${2:-false}

    echo "=== Terraform Apply ==="

    if [[ ! -f "$plan_file" ]]; then
        echo "No plan file found. Run tf_plan first." >&2
        return 1
    fi

    local args=(-no-color "$plan_file")
    $auto_approve && args=(-no-color -auto-approve "$plan_file")

    terraform -chdir="$dir" apply "${args[@]}"
    echo "Apply complete"
}

tf_destroy() {
    local dir=${1:-$TF_DIR}
    local vars_file="${dir}/vars/${TF_WORKSPACE}.tfvars"

    echo "=== Terraform Destroy: $TF_WORKSPACE ==="
    echo "WARNING: This will destroy all resources!"
    read -rp "Type the workspace name to confirm: " confirm

    if [[ "$confirm" != "$TF_WORKSPACE" ]]; then
        echo "Cancelled"
        return 1
    fi

    local args=(-no-color -auto-approve)
    [[ -f "$vars_file" ]] && args+=(-var-file="$vars_file")

    terraform -chdir="$dir" destroy "${args[@]}"
}

tf_output() {
    local dir=${1:-$TF_DIR}
    local key=${2:-}

    if [[ -n "$key" ]]; then
        terraform -chdir="$dir" output -raw "$key"
    else
        terraform -chdir="$dir" output -json
    fi
}

tf_state_list() {
    local dir=${1:-$TF_DIR}
    terraform -chdir="$dir" state list
}

tf_import() {
    local dir=${1:-$TF_DIR}
    local resource_addr=$2
    local resource_id=$3

    echo "Importing: $resource_addr = $resource_id"
    terraform -chdir="$dir" import "$resource_addr" "$resource_id"
}
```

---

## 42.2 AWS Infrastructure Automation

```bash
#!/bin/bash
# aws_infra.sh

AWS_REGION="${AWS_REGION:-us-east-1}"
AWS_PROFILE="${AWS_PROFILE:-default}"

aws_cmd() {
    aws --region "$AWS_REGION" --profile "$AWS_PROFILE" "$@"
}

# ─── EC2 Management ────────────────────────────────────────────
create_ec2_instance() {
    local name=$1
    local instance_type=${2:-t3.micro}
    local ami_id=${3:-}
    local subnet_id=${4:-}
    local sg_ids=${5:-}
    local key_name=${6:-}

    [[ -z "$ami_id" ]] && ami_id=$(get_latest_amazon_linux_ami)

    echo "Creating EC2: $name ($instance_type)"

    local instance_id
    instance_id=$(aws_cmd ec2 run-instances \
        --image-id "$ami_id" \
        --instance-type "$instance_type" \
        --key-name "$key_name" \
        --subnet-id "$subnet_id" \
        --security-group-ids $sg_ids \
        --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$name}]" \
        --query 'Instances[0].InstanceId' \
        --output text)

    echo "Instance ID: $instance_id"
    aws_cmd ec2 wait instance-running --instance-ids "$instance_id"

    local public_ip
    public_ip=$(aws_cmd ec2 describe-instances \
        --instance-ids "$instance_id" \
        --query 'Reservations[0].Instances[0].PublicIpAddress' \
        --output text)

    echo "Public IP: $public_ip"
    echo "$instance_id"
}

get_latest_amazon_linux_ami() {
    aws_cmd ec2 describe-images \
        --owners amazon \
        --filters \
            "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
            "Name=state,Values=available" \
        --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
        --output text
}

list_instances() {
    local state=${1:-running}
    aws_cmd ec2 describe-instances \
        --filters "Name=instance-state-name,Values=$state" \
        --query 'Reservations[].Instances[].[InstanceId,InstanceType,PublicIpAddress,Tags[?Key==`Name`].Value|[0],State.Name]' \
        --output table
}

stop_instances_by_tag() {
    local tag_key=$1
    local tag_value=$2

    local instance_ids
    instance_ids=$(aws_cmd ec2 describe-instances \
        --filters "Name=tag:$tag_key,Values=$tag_value" "Name=instance-state-name,Values=running" \
        --query 'Reservations[].Instances[].InstanceId' \
        --output text)

    [[ -z "$instance_ids" ]] && echo "No running instances found" && return 0

    echo "Stopping: $instance_ids"
    aws_cmd ec2 stop-instances --instance-ids $instance_ids
    aws_cmd ec2 wait instance-stopped --instance-ids $instance_ids
    echo "Stopped"
}

# ─── S3 Bucket Management ──────────────────────────────────────
create_s3_bucket() {
    local bucket_name=$1
    local versioning=${2:-true}
    local encryption=${3:-true}
    local public_access_block=${4:-true}

    echo "Creating S3 bucket: $bucket_name"

    if [[ "$AWS_REGION" == "us-east-1" ]]; then
        aws_cmd s3api create-bucket --bucket "$bucket_name"
    else
        aws_cmd s3api create-bucket \
            --bucket "$bucket_name" \
            --create-bucket-configuration "LocationConstraint=$AWS_REGION"
    fi

    $versioning && aws_cmd s3api put-bucket-versioning \
        --bucket "$bucket_name" \
        --versioning-configuration Status=Enabled

    $encryption && aws_cmd s3api put-bucket-encryption \
        --bucket "$bucket_name" \
        --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

    $public_access_block && aws_cmd s3api put-public-access-block \
        --bucket "$bucket_name" \
        --public-access-block-configuration \
            "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

    echo "Bucket created: s3://$bucket_name"
}

sync_to_s3() {
    local local_path=$1
    local bucket=$2
    local prefix=${3:-}
    local delete=${4:-false}

    local dest="s3://${bucket}/${prefix}"
    local args=(--region "$AWS_REGION")
    $delete && args+=(--delete)

    echo "Syncing: $local_path → $dest"
    aws s3 sync "$local_path" "$dest" "${args[@]}"
    echo "Sync complete"
}

# ─── RDS Snapshots ─────────────────────────────────────────────
create_rds_snapshot() {
    local db_identifier=$1
    local snapshot_id="${db_identifier}-$(date +%Y%m%d%H%M%S)"

    echo "Creating RDS snapshot: $snapshot_id"
    aws_cmd rds create-db-snapshot \
        --db-instance-identifier "$db_identifier" \
        --db-snapshot-identifier "$snapshot_id"

    aws_cmd rds wait db-snapshot-completed \
        --db-snapshot-identifier "$snapshot_id"

    echo "Snapshot created: $snapshot_id"
}
```

---

## 42.3 Configuration Management

```bash
#!/bin/bash
# config_management.sh

declare -A TASK_RESULTS=()
declare -a TASK_NAMES=()

task() {
    local name=$1
    shift
    TASK_NAMES+=("$name")
    echo "TASK [$name]"
    if "$@"; then
        TASK_RESULTS["$name"]="ok"
        echo "  ok"
    else
        TASK_RESULTS["$name"]="failed"
        echo "  FAILED"
        return 1
    fi
}

task_lineinfile() {
    local name=$1
    local file=$2
    local line=$3
    local state=${4:-present}

    TASK_NAMES+=("$name")
    echo "TASK [$name] lineinfile: $file"

    if [[ "$state" == "present" ]]; then
        if grep -qF "$line" "$file" 2>/dev/null; then
            TASK_RESULTS["$name"]="ok (no change)"
            echo "  ok (already present)"
        else
            echo "$line" >> "$file"
            TASK_RESULTS["$name"]="changed"
            echo "  changed (added)"
        fi
    elif [[ "$state" == "absent" ]]; then
        if grep -qF "$line" "$file" 2>/dev/null; then
            grep -vF "$line" "$file" > "${file}.tmp" && mv "${file}.tmp" "$file"
            TASK_RESULTS["$name"]="changed"
            echo "  changed (removed)"
        else
            TASK_RESULTS["$name"]="ok (no change)"
            echo "  ok (not present)"
        fi
    fi
}

task_package() {
    local name=$1
    local package=$2
    local state=${3:-installed}

    TASK_NAMES+=("$name")
    echo "TASK [$name] package: $package"

    local pkg_cmd
    command -v apt-get &>/dev/null && pkg_cmd="apt-get"
    command -v yum &>/dev/null && pkg_cmd="yum"
    command -v dnf &>/dev/null && pkg_cmd="dnf"

    if [[ "$state" == "installed" ]]; then
        if dpkg -l "$package" &>/dev/null 2>&1 || rpm -q "$package" &>/dev/null 2>&1; then
            TASK_RESULTS["$name"]="ok (already installed)"
            echo "  ok (already installed)"
        else
            $pkg_cmd install -y "$package" -q
            TASK_RESULTS["$name"]="changed (installed)"
            echo "  changed (installed)"
        fi
    fi
}

task_service() {
    local name=$1
    local service=$2
    local state=${3:-started}
    local enabled=${4:-true}

    TASK_NAMES+=("$name")
    echo "TASK [$name] service: $service ($state)"

    case "$state" in
        started)
            if ! systemctl is-active --quiet "$service"; then
                systemctl start "$service"
                TASK_RESULTS["$name"]="changed (started)"
                echo "  changed (started)"
            else
                TASK_RESULTS["$name"]="ok (running)"
                echo "  ok (already running)"
            fi
            ;;
        stopped)
            if systemctl is-active --quiet "$service"; then
                systemctl stop "$service"
                TASK_RESULTS["$name"]="changed (stopped)"
                echo "  changed (stopped)"
            else
                TASK_RESULTS["$name"]="ok (already stopped)"
                echo "  ok (already stopped)"
            fi
            ;;
        restarted)
            systemctl restart "$service"
            TASK_RESULTS["$name"]="changed (restarted)"
            echo "  changed (restarted)"
            ;;
    esac

    $enabled && systemctl enable "$service" 2>/dev/null
}

playbook_summary() {
    echo ""
    echo "=== Playbook Summary ==="
    local ok=0 changed=0 failed=0

    for t in "${TASK_NAMES[@]}"; do
        local result="${TASK_RESULTS[$t]:-unknown}"
        case "$result" in
            ok*)      (( ok++ )) ;;
            changed*) (( changed++ )) ;;
            failed*)  (( failed++ )) ;;
        esac
    done

    echo "ok=$ok  changed=$changed  failed=$failed"
}
```

---

## 42.4 Server Provisioning

```bash
#!/bin/bash
# server_provision.sh

provision_web_server() {
    local domain=$1
    local app_user=${2:-webapp}
    local app_dir=${3:-/var/www/$domain}

    echo "=== Provisioning Web Server: $domain ==="

    task "Update packages"   apt-get update -q
    task "Install Nginx"     apt-get install -y nginx
    task "Install Certbot"   apt-get install -y certbot python3-certbot-nginx

    task "Create app user" bash -c "
        id '$app_user' &>/dev/null || useradd -m -s /bin/bash '$app_user'
    "

    task "Create app directory" bash -c "
        mkdir -p '$app_dir'
        chown '$app_user:$app_user' '$app_dir'
        chmod 755 '$app_dir'
    "

    cat > "/tmp/nginx_${domain}.conf" << EOF
server {
    listen 80;
    server_name $domain www.$domain;
    root $app_dir;
    index index.html;

    location / {
        try_files \$uri \$uri/ =404;
    }

    location ~* \.(jpg|jpeg|png|gif|ico|css|js)\$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    access_log /var/log/nginx/$domain.access.log;
    error_log  /var/log/nginx/$domain.error.log;
}
EOF

    task "Install Nginx vhost" \
        cp "/tmp/nginx_${domain}.conf" "/etc/nginx/sites-available/$domain"

    task "Enable vhost" bash -c "
        ln -sf '/etc/nginx/sites-available/$domain' '/etc/nginx/sites-enabled/$domain'
        nginx -t
    "

    task_service "Start Nginx" nginx started true
    playbook_summary
}

provision_database_server() {
    local db_type=${1:-postgresql}
    local db_version=${2:-14}
    local db_name=${3:-appdb}
    local db_user=${4:-appuser}
    local db_password=${5:-}

    echo "=== Provisioning DB Server: $db_type $db_version ==="

    case "$db_type" in
        postgresql)
            task "Install PostgreSQL" apt-get install -y "postgresql-$db_version"
            task_service "Start PostgreSQL" "postgresql" started true
            task "Create database" sudo -u postgres psql << SQL
CREATE USER ${db_user} WITH PASSWORD '${db_password}';
CREATE DATABASE ${db_name} OWNER ${db_user};
GRANT ALL PRIVILEGES ON DATABASE ${db_name} TO ${db_user};
SQL
            ;;
        mysql)
            task "Install MySQL" apt-get install -y mysql-server
            task_service "Start MySQL" mysql started true
            ;;
    esac

    playbook_summary
}
```

---

## 42.5 Infrastructure State Management

```bash
#!/bin/bash
# infra_state.sh

STATE_DIR="${STATE_DIR:-/var/lib/infra}"
STATE_FILE="${STATE_DIR}/state.json"

state_init() {
    mkdir -p "$STATE_DIR"
    [[ ! -f "$STATE_FILE" ]] && echo '{"resources":{},"version":1}' > "$STATE_FILE"
}

state_set() {
    local resource_id=$1
    local resource_type=$2
    local status=$3
    local metadata=${4:-{}}

    state_init

    local updated
    updated=$(date -u +%Y-%m-%dT%H:%M:%SZ)

    jq \
        --arg id "$resource_id" \
        --arg type "$resource_type" \
        --arg status "$status" \
        --argjson meta "$metadata" \
        --arg ts "$updated" \
        '.resources[$id] = {type: $type, status: $status, metadata: $meta, updated: $ts}' \
        "$STATE_FILE" > "${STATE_FILE}.tmp" && mv "${STATE_FILE}.tmp" "$STATE_FILE"
}

state_get() {
    local resource_id=$1
    state_init
    jq -r ".resources[\"$resource_id\"] // null" "$STATE_FILE"
}

state_list() {
    state_init
    jq -r '.resources | to_entries[] |
        [.key, .value.type, .value.status, .value.updated] | @tsv' "$STATE_FILE" | \
        column -t -s $'\t'
}

state_delete() {
    local resource_id=$1
    state_init
    jq --arg id "$resource_id" 'del(.resources[$id])' "$STATE_FILE" > "${STATE_FILE}.tmp"
    mv "${STATE_FILE}.tmp" "$STATE_FILE"
    echo "Removed: $resource_id"
}

infra_drift_detect() {
    echo "=== Infrastructure Drift Detection ==="
    local drifted=0

    while IFS=$'\t' read -r resource_id type status _; do
        local actual_status=""

        case "$type" in
            systemd_service)
                systemctl is-active --quiet "$resource_id" 2>/dev/null && \
                    actual_status="running" || actual_status="stopped"
                ;;
            *) continue ;;
        esac

        if [[ "$actual_status" != "$status" ]]; then
            echo "DRIFT: $resource_id | expected=$status actual=$actual_status"
            (( drifted++ ))
        fi
    done < <(state_list)

    (( drifted == 0 )) && echo "No drift detected" || { echo "$drifted resource(s) drifted"; return 1; }
}
```

---

## 42.6 Multi-Environment Management

```bash
#!/bin/bash
# env_manager.sh

declare -A ENV_CONFIGS=(
    [dev_region]="us-west-2"
    [dev_account]="111111111111"
    [staging_region]="us-east-1"
    [staging_account]="222222222222"
    [prod_region]="us-east-1"
    [prod_account]="333333333333"
)

switch_env() {
    local env=$1
    local region="${ENV_CONFIGS[${env}_region]:-}"

    if [[ -z "$region" ]]; then
        echo "Unknown environment: $env" >&2
        return 1
    fi

    export AWS_REGION="$region"
    export ENV_NAME="$env"
    echo "Switched to: $env (region=$region)"
}

with_env() {
    local env=$1
    shift

    local orig_region="${AWS_REGION:-}"
    local orig_env="${ENV_NAME:-}"

    switch_env "$env"

    if [[ "${ENV_NAME}" == "prod" ]]; then
        echo "PRODUCTION - Extra confirmation required"
        read -rp "Type 'I understand this is PRODUCTION' to proceed: " confirm
        if [[ "$confirm" != "I understand this is PRODUCTION" ]]; then
            echo "Cancelled"
            AWS_REGION="$orig_region"; ENV_NAME="$orig_env"
            return 1
        fi
    fi

    "$@"
    local exit_code=$?

    AWS_REGION="$orig_region"
    ENV_NAME="$orig_env"
    return $exit_code
}

promote_artifact() {
    local artifact=$1
    local from_env=$2
    local to_env=$3

    echo "=== Promoting: $artifact ($from_env → $to_env) ==="

    local from_bucket="${from_env}-artifacts"
    local to_bucket="${to_env}-artifacts"

    with_env "$to_env" aws s3 cp \
        "s3://${from_bucket}/${artifact}" \
        "s3://${to_bucket}/${artifact}"

    echo "Promoted to: s3://${to_bucket}/${artifact}"
}
```

---

## 42.7 Exercises

### Exercise 1: Auto-Scaling Group Manager
สร้าง tool จัดการ ASG:
- Scale up/down based on metrics
- Drain instances gracefully
- Update launch template
- Rolling AMI update

### Exercise 2: Cost Optimizer
สร้าง cost optimization tool:
- Find unused resources
- Identify over-provisioned instances
- Generate savings report
- Auto-stop dev resources at night

### Exercise 3: Disaster Recovery
สร้าง DR automation:
- Backup to secondary region
- Failover procedure
- RTO/RPO measurement
- Recovery testing

---

## สรุป Part 42

✅ Terraform automation (init/plan/apply/destroy)  
✅ AWS EC2/S3/RDS management  
✅ Configuration management (task/playbook pattern)  
✅ Server provisioning (web + database)  
✅ Infrastructure state tracking  
✅ Drift detection  
✅ Multi-environment management  
✅ Production safeguards  

---

**→ Part 43: Advanced Shell Scripting Patterns**
