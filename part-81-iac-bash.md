# Part 81: Infrastructure as Code with Bash
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 81.1 Terraform Wrapper

```bash
#!/bin/bash
# terraform.sh - Terraform automation wrapper

set -euo pipefail

TF_DIR="${TF_DIR:-.}"
TF_WORKSPACE="${TF_WORKSPACE:-default}"
TF_VARS_FILE="${TF_VARS_FILE:-}"
TF_LOG_FILE="${TF_LOG_FILE:-/tmp/terraform.log}"

_tf() {
    local -a cmd=(terraform)
    cmd+=(-chdir="$TF_DIR")
    echo "[TF] ${cmd[*]} $*" >> "$TF_LOG_FILE"
    "${cmd[@]}" "$@"
}

tf_init() {
    local upgrade=${1:-false}
    local -a args=(-input=false)
    ${upgrade} && args+=(-upgrade)
    _tf init "${args[@]}"
}

tf_workspace_select() {
    local ws=$1
    _tf workspace select "$ws" 2>/dev/null || \
        _tf workspace new "$ws"
    TF_WORKSPACE=$ws
    echo "Workspace: $ws"
}

tf_plan() {
    local plan_file=${1:-/tmp/tfplan} extra_vars=${2:-}
    local -a args=(-out="$plan_file" -input=false)
    [[ -n "$TF_VARS_FILE" ]] && args+=(-var-file="$TF_VARS_FILE")
    [[ -n "$extra_vars" ]]   && args+=(-var "$extra_vars")
    _tf plan "${args[@]}"
    echo "Plan saved: $plan_file"
    echo "$plan_file"
}

tf_apply() {
    local plan_file=${1:-/tmp/tfplan}
    if [[ -f "$plan_file" ]]; then
        _tf apply -input=false -auto-approve "$plan_file"
    else
        local -a args=(-input=false -auto-approve)
        [[ -n "$TF_VARS_FILE" ]] && args+=(-var-file="$TF_VARS_FILE")
        _tf apply "${args[@]}"
    fi
    echo "Apply complete"
}

tf_destroy() {
    local target=${1:-}
    local -a args=(-input=false -auto-approve)
    [[ -n "$TF_VARS_FILE" ]] && args+=(-var-file="$TF_VARS_FILE")
    [[ -n "$target" ]] && args+=(-target="$target")
    _tf destroy "${args[@]}"
    echo "Destroy complete"
}

tf_output() {
    local key=${1:-}
    if [[ -n "$key" ]]; then
        _tf output -raw "$key" 2>/dev/null
    else
        _tf output -json 2>/dev/null
    fi
}

tf_state_list() {
    _tf state list 2>/dev/null
}

tf_state_show() {
    local resource=$1
    _tf state show "$resource" 2>/dev/null
}

tf_taint() {
    local resource=$1
    _tf state taint "$resource"
    echo "Tainted: $resource"
}

tf_import() {
    local resource=$1 cloud_id=$2
    _tf import "$resource" "$cloud_id"
    echo "Imported: $resource <- $cloud_id"
}

tf_validate() {
    _tf validate -json 2>/dev/null | jq '{valid, error_count, warning_count}'
}

tf_format_check() {
    if _tf fmt -check -recursive "$TF_DIR" 2>/dev/null; then
        echo "Format: OK"
        return 0
    else
        echo "Format: needs formatting"
        return 1
    fi
}

tf_plan_diff() {
    local plan_file=${1:-/tmp/tfplan}
    _tf show -json "$plan_file" 2>/dev/null | \
        jq '.resource_changes[] | {resource:.address, action:.change.actions[]}' 2>/dev/null
}
```

---

## 81.2 Ansible Wrapper

```bash
#!/bin/bash
# ansible.sh - Ansible automation wrapper

ANSIBLE_DIR="${ANSIBLE_DIR:-.}"
ANSIBLE_INVENTORY="${ANSIBLE_INVENTORY:-inventory}"
ANSIBLE_VAULT_PASSWORD_FILE="${ANSIBLE_VAULT_PASSWORD_FILE:-}"
ANSIBLE_LOG="${ANSIBLE_LOG:-/tmp/ansible.log}"

_ansible_common_args() {
    local -a args=(-i "${ANSIBLE_DIR}/${ANSIBLE_INVENTORY}")
    [[ -n "$ANSIBLE_VAULT_PASSWORD_FILE" ]] && \
        args+=(--vault-password-file "$ANSIBLE_VAULT_PASSWORD_FILE")
    printf '%s\n' "${args[@]}"
}

ansible_ping() {
    local pattern=${1:-all}
    mapfile -t common < <(_ansible_common_args)
    ansible "${common[@]}" "$pattern" -m ping
}

ansible_run_playbook() {
    local playbook=$1
    shift
    local extra_vars=("$@")

    mapfile -t common < <(_ansible_common_args)
    local -a cmd=(ansible-playbook "${common[@]}" \
        "${ANSIBLE_DIR}/${playbook}")

    for var in "${extra_vars[@]}"; do
        cmd+=(-e "$var")
    done

    echo "Running: ${playbook}"
    "${cmd[@]}" 2>&1 | tee -a "$ANSIBLE_LOG"
}

ansible_check_playbook() {
    local playbook=$1
    mapfile -t common < <(_ansible_common_args)
    ansible-playbook "${common[@]}" \
        "${ANSIBLE_DIR}/${playbook}" --check --diff
}

ansible_run_role() {
    local hosts=$1 role=$2
    shift 2
    local extra_vars=("$@")

    local tmp_playbook; tmp_playbook=$(mktemp --suffix=.yml)
    cat > "$tmp_playbook" << YAML
---
- hosts: $hosts
  roles:
    - $role
YAML

    ANSIBLE_DIR="$(dirname "$tmp_playbook")" \
        ansible_run_playbook "$(basename "$tmp_playbook")" "${extra_vars[@]}"
    rm -f "$tmp_playbook"
}

ansible_vault_encrypt() {
    local file=$1
    ansible-vault encrypt "$file" \
        ${ANSIBLE_VAULT_PASSWORD_FILE:+--vault-password-file "$ANSIBLE_VAULT_PASSWORD_FILE"}
}

ansible_vault_decrypt() {
    local file=$1
    ansible-vault decrypt "$file" \
        ${ANSIBLE_VAULT_PASSWORD_FILE:+--vault-password-file "$ANSIBLE_VAULT_PASSWORD_FILE"}
}

ansible_gather_facts() {
    local host=$1
    mapfile -t common < <(_ansible_common_args)
    ansible "${common[@]}" "$host" -m setup 2>/dev/null
}
```

---

## 81.3 AWS Infrastructure Helpers

```bash
#!/bin/bash
# aws_infra.sh - AWS infrastructure automation

AWS_REGION="${AWS_DEFAULT_REGION:-us-east-1}"

ec2_create_instance() {
    local name=$1 ami=$2 instance_type=${3:-t3.micro}
    local key_name=${4:-} security_group=${5:-} subnet=${6:-}

    local -a args=(
        --image-id "$ami"
        --instance-type "$instance_type"
        --region "$AWS_REGION"
        --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$name}]"
    )
    [[ -n "$key_name" ]]        && args+=(--key-name "$key_name")
    [[ -n "$security_group" ]]  && args+=(--security-group-ids "$security_group")
    [[ -n "$subnet" ]]          && args+=(--subnet-id "$subnet")

    local instance_id
    instance_id=$(aws ec2 run-instances "${args[@]}" \
        --query 'Instances[0].InstanceId' --output text 2>/dev/null)

    echo "$instance_id"
}

ec2_wait_running() {
    local instance_id=$1
    aws ec2 wait instance-running \
        --instance-ids "$instance_id" \
        --region "$AWS_REGION"
    echo "Running: $instance_id"
}

ec2_get_public_ip() {
    local instance_id=$1
    aws ec2 describe-instances \
        --instance-ids "$instance_id" \
        --region "$AWS_REGION" \
        --query 'Reservations[0].Instances[0].PublicIpAddress' \
        --output text 2>/dev/null
}

ec2_terminate() {
    local instance_id=$1
    aws ec2 terminate-instances \
        --instance-ids "$instance_id" \
        --region "$AWS_REGION" > /dev/null
    echo "Terminating: $instance_id"
}

s3_ensure_bucket() {
    local bucket=$1 region=${2:-$AWS_REGION}

    if ! aws s3api head-bucket --bucket "$bucket" --region "$region" 2>/dev/null; then
        if [[ "$region" == "us-east-1" ]]; then
            aws s3api create-bucket \
                --bucket "$bucket" \
                --region "$region" > /dev/null
        else
            aws s3api create-bucket \
                --bucket "$bucket" \
                --region "$region" \
                --create-bucket-configuration LocationConstraint="$region" > /dev/null
        fi
        echo "Created bucket: $bucket"
    fi

    # Enable versioning
    aws s3api put-bucket-versioning \
        --bucket "$bucket" \
        --versioning-configuration Status=Enabled \
        --region "$region" 2>/dev/null

    echo "Bucket ready: $bucket"
}

s3_sync_dir() {
    local local_dir=$1 s3_path=$2 extra=${3:-}
    aws s3 sync "$local_dir" "$s3_path" \
        --delete \
        ${extra:+$extra}
}

vpc_get_default_id() {
    aws ec2 describe-vpcs \
        --filters 'Name=isDefault,Values=true' \
        --query 'Vpcs[0].VpcId' \
        --output text \
        --region "$AWS_REGION" 2>/dev/null
}

sg_create() {
    local name=$1 description=$2 vpc_id=${3:-}
    [[ -z "$vpc_id" ]] && vpc_id=$(vpc_get_default_id)

    aws ec2 create-security-group \
        --group-name "$name" \
        --description "$description" \
        --vpc-id "$vpc_id" \
        --region "$AWS_REGION" \
        --query 'GroupId' --output text 2>/dev/null
}

sg_allow_port() {
    local sg_id=$1 port=$2 cidr=${3:-0.0.0.0/0} proto=${4:-tcp}
    aws ec2 authorize-security-group-ingress \
        --group-id "$sg_id" \
        --protocol "$proto" \
        --port "$port" \
        --cidr "$cidr" \
        --region "$AWS_REGION" 2>/dev/null || true
    echo "Allowed port $port on $sg_id"
}
```

---

## 81.4 Configuration Drift Detection

```bash
#!/bin/bash
# drift_detect.sh - Infrastructure configuration drift detection

DRIFT_DB="${DRIFT_DB:-/tmp/drift.db}"

drift_init() {
    sqlite3 "$DRIFT_DB" <<'SQL'
CREATE TABLE IF NOT EXISTS snapshots (
    id        INTEGER PRIMARY KEY AUTOINCREMENT,
    host      TEXT NOT NULL,
    item_type TEXT NOT NULL,
    item_key  TEXT NOT NULL,
    value     TEXT,
    captured_at DATETIME NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX IF NOT EXISTS idx_snap ON snapshots(host, item_type, item_key, captured_at DESC);
SQL
}

drift_snapshot_files() {
    local host=${1:-$(hostname)} paths=${2:-"/etc/passwd /etc/hosts"}

    for path in $paths; do
        [[ -f "$path" ]] || continue
        local hash; hash=$(sha256sum "$path" | cut -d' ' -f1)
        sqlite3 "$DRIFT_DB" \
            "INSERT INTO snapshots (host, item_type, item_key, value) \
             VALUES ('$host', 'file_hash', '$path', '$hash');" 2>/dev/null
    done
    echo "Snapshot taken: $host"
}

drift_snapshot_packages() {
    local host=${1:-$(hostname)}

    local pkg_list
    if command -v dpkg &>/dev/null; then
        pkg_list=$(dpkg-query -W -f='${Package}=${Version}\n' 2>/dev/null)
    elif command -v rpm &>/dev/null; then
        pkg_list=$(rpm -qa --qf '%{NAME}=%{VERSION}\n' 2>/dev/null)
    else
        return 1
    fi

    while IFS='=' read -r pkg ver; do
        sqlite3 "$DRIFT_DB" \
            "INSERT INTO snapshots (host, item_type, item_key, value) \
             VALUES ('$host', 'package', '$pkg', '$ver');" 2>/dev/null
    done <<< "$pkg_list"
    echo "Package snapshot taken: $host"
}

drift_snapshot_services() {
    local host=${1:-$(hostname)}

    systemctl list-units --type=service --state=running \
        --no-legend --no-pager 2>/dev/null | \
    awk '{print $1}' | while read -r svc; do
        sqlite3 "$DRIFT_DB" \
            "INSERT INTO snapshots (host, item_type, item_key, value) \
             VALUES ('$host', 'service_running', '$svc', 'true');" 2>/dev/null
    done
    echo "Service snapshot taken: $host"
}

drift_compare() {
    local host=${1:-$(hostname)} item_type=${2:-file_hash}

    sqlite3 "$DRIFT_DB" <<SQL
.mode column
.headers on
SELECT
    a.item_key,
    a.value AS baseline,
    b.value AS current,
    CASE WHEN a.value != b.value THEN 'CHANGED'
         WHEN b.value IS NULL THEN 'REMOVED'
         ELSE 'OK' END AS status
FROM (
    SELECT item_key, value FROM snapshots
    WHERE host='$host' AND item_type='$item_type'
    GROUP BY item_key HAVING MIN(captured_at)
) a
LEFT JOIN (
    SELECT item_key, value FROM snapshots
    WHERE host='$host' AND item_type='$item_type'
    GROUP BY item_key HAVING MAX(captured_at)
) b USING (item_key)
WHERE a.value != COALESCE(b.value, '') OR b.value IS NULL;
SQL
}
```

---

## 81.5 Exercises

### Exercise 1: Terraform Workflow
สร้าง full workflow:
- init → workspace select → plan → review diff → apply
- Output instance IDs/IPs
- Post-apply drift snapshot

### Exercise 2: Idempotent Server Bootstrap
สร้าง bootstrap.sh ที่:
- ติดตั้ง packages อีกครั้งไม่ติดตั้งซ้ำ
- Configure firewall
- Create users
- Deploy app config
- Record step completion

### Exercise 3: Drift Alert
สร้าง cron job ที่:
- snapshot รายวัน
- เปรียบกับเมื่อวาน
- Alert ถ้าพบ drift
- Auto-remediate ไฟล์ปรับแต่ง

---

## สรุป Part 81

✅ Terraform: init/workspace/plan/apply/destroy with auto-approve and var-file
┅ Terraform: output, state list/show/taint/import, validate JSON, fmt check
┅ tf_plan_diff: JSON plan → resource changes summary
┅ Ansible: ping, run_playbook (tee log), check+diff, run_role (temp playbook)
┅ Ansible vault: encrypt/decrypt files
┅ AWS: EC2 create/wait/IP/terminate, S3 ensure bucket + versioning + sync
┅ AWS: VPC default ID, security group create + allow port
┅ Drift detection: SQLite snapshot of file hashes, packages, running services
┅ drift_compare: baseline vs latest snapshot, CHANGED/REMOVED/OK per item

---

**→ Part 82: Log Management and Analysis Pipelines**
