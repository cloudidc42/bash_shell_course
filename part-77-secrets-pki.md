# Part 77: Secrets, Certificates, and PKI Automation
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 77.1 GPG Operations

```bash
#!/bin/bash
# gpg_ops.sh - GPG key and encryption operations

set -euo pipefail

GPG_HOME="${GNUPGHOME:-${HOME}/.gnupg}"

gpg_cmd() {
    gpg --batch --no-tty --homedir "$GPG_HOME" "$@"
}

gpg_generate_key() {
    local name=$1 email=$2 passphrase=${3:-}
    local key_type=${4:-RSA} key_length=${5:-4096}

    local config; config=$(mktemp)
    cat > "$config" << EOF
%no-protection
Key-Type: $key_type
Key-Length: $key_length
Subkey-Type: $key_type
Subkey-Length: $key_length
Name-Real: $name
Name-Email: $email
Expire-Date: 2y
%commit
EOF
    [[ -n "$passphrase" ]] && sed -i "s|%no-protection|Passphrase: $passphrase|" "$config"

    gpg_cmd --gen-key "$config"
    rm -f "$config"
    echo "GPG key generated: ${name} <${email}>"
}

gpg_export_public() {
    local email=$1 output_file=${2:-public.gpg}
    gpg_cmd --armor --export "$email" > "$output_file"
    echo "Public key exported: $output_file"
}

gpg_export_private() {
    local email=$1 output_file=${2:-private.gpg}
    gpg_cmd --armor --export-secret-keys "$email" > "$output_file"
    chmod 600 "$output_file"
    echo "Private key exported: $output_file"
}

gpg_import_key() {
    local key_file=$1
    gpg_cmd --import "$key_file"
}

gpg_encrypt_file() {
    local input_file=$1 recipient=$2 output_file=${3:-${input_file}.gpg}
    gpg_cmd --encrypt --recipient "$recipient" --output "$output_file" "$input_file"
    echo "Encrypted: $output_file"
}

gpg_decrypt_file() {
    local input_file=$1 output_file=${2:-${input_file%.gpg}}
    local passphrase=${3:-}

    if [[ -n "$passphrase" ]]; then
        echo "$passphrase" | gpg_cmd --passphrase-fd 0 --decrypt --output "$output_file" "$input_file"
    else
        gpg_cmd --decrypt --output "$output_file" "$input_file"
    fi
    echo "Decrypted: $output_file"
}

gpg_sign_file() {
    local file=$1 signer=$2
    gpg_cmd --detach-sign --armor --local-user "$signer" "$file"
    echo "Signature: ${file}.asc"
}

gpg_verify_signature() {
    local file=$1 sig_file=${2:-${file}.asc}
    gpg_cmd --verify "$sig_file" "$file" 2>&1
}

gpg_encrypt_string() {
    local plaintext=$1 recipient=$2
    echo "$plaintext" | gpg_cmd --encrypt --armor --recipient "$recipient"
}

gpg_decrypt_string() {
    local ciphertext=$1
    echo "$ciphertext" | gpg_cmd --decrypt 2>/dev/null
}
```

---

## 77.2 OpenSSL CA and Certificate Management

```bash
#!/bin/bash
# pki.sh - OpenSSL CA and certificate operations

CA_DIR="${CA_DIR:-/tmp/pki}"
CA_BITS="${CA_BITS:-4096}"
CA_DAYS="${CA_DAYS:-3650}"
CERT_DAYS="${CERT_DAYS:-365}"

pki_init_ca() {
    local ca_name=${1:-MyCA}
    local ca_subj=${2:-"/C=US/ST=State/O=Org/CN=${ca_name}"}

    mkdir -p "${CA_DIR}/certs" "${CA_DIR}/private" "${CA_DIR}/crl" "${CA_DIR}/newcerts"
    chmod 700 "${CA_DIR}/private"
    echo "01" > "${CA_DIR}/serial"
    touch "${CA_DIR}/index.txt"

    # Generate CA key and self-signed cert
    openssl genrsa -out "${CA_DIR}/private/ca.key" "$CA_BITS" 2>/dev/null
    chmod 600 "${CA_DIR}/private/ca.key"

    openssl req -new -x509 -days "$CA_DAYS" \
        -key "${CA_DIR}/private/ca.key" \
        -out "${CA_DIR}/certs/ca.crt" \
        -subj "$ca_subj" 2>/dev/null

    echo "CA initialized: ${CA_DIR}/certs/ca.crt"
}

pki_generate_server_cert() {
    local cn=$1 san=${2:-}
    local key_file="${CA_DIR}/private/${cn}.key"
    local csr_file="${CA_DIR}/${cn}.csr"
    local cert_file="${CA_DIR}/certs/${cn}.crt"

    # Generate key
    openssl genrsa -out "$key_file" 2048 2>/dev/null
    chmod 600 "$key_file"

    # Generate CSR
    local subj="/CN=${cn}"
    openssl req -new -key "$key_file" -out "$csr_file" -subj "$subj" 2>/dev/null

    # Extensions for SAN
    local ext_file; ext_file=$(mktemp)
    cat > "$ext_file" << EOF
[server_ext]
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = ${san:-DNS:${cn}}
EOF

    # Sign with CA
    openssl x509 -req -days "$CERT_DAYS" \
        -in "$csr_file" \
        -CA "${CA_DIR}/certs/ca.crt" \
        -CAkey "${CA_DIR}/private/ca.key" \
        -CAserial "${CA_DIR}/serial" \
        -extfile "$ext_file" -extensions server_ext \
        -out "$cert_file" 2>/dev/null

    rm -f "$ext_file" "$csr_file"
    echo "Server cert: $cert_file"
    echo "Key:         $key_file"
}

pki_generate_client_cert() {
    local cn=$1
    local key_file="${CA_DIR}/private/${cn}-client.key"
    local cert_file="${CA_DIR}/certs/${cn}-client.crt"
    local csr_file="${CA_DIR}/${cn}-client.csr"

    openssl genrsa -out "$key_file" 2048 2>/dev/null
    chmod 600 "$key_file"
    openssl req -new -key "$key_file" -out "$csr_file" -subj "/CN=${cn}" 2>/dev/null
    openssl x509 -req -days "$CERT_DAYS" \
        -in "$csr_file" \
        -CA "${CA_DIR}/certs/ca.crt" \
        -CAkey "${CA_DIR}/private/ca.key" \
        -CAserial "${CA_DIR}/serial" \
        -out "$cert_file" 2>/dev/null
    rm -f "$csr_file"
    echo "Client cert: $cert_file"
}

pki_create_pkcs12() {
    local cn=$1 password=${2:-changeme}
    local p12_file="${CA_DIR}/certs/${cn}.p12"

    openssl pkcs12 -export \
        -in "${CA_DIR}/certs/${cn}.crt" \
        -inkey "${CA_DIR}/private/${cn}.key" \
        -certfile "${CA_DIR}/certs/ca.crt" \
        -out "$p12_file" \
        -passout "pass:$password" 2>/dev/null
    echo "PKCS12 bundle: $p12_file"
}

pki_cert_info() {
    local cert_file=$1
    openssl x509 -in "$cert_file" -noout \
        -subject -issuer -dates -fingerprint 2>/dev/null
}

pki_cert_expiry_days() {
    local cert_file=$1
    local expiry_date; expiry_date=$(openssl x509 -in "$cert_file" -noout -enddate 2>/dev/null | cut -d= -f2)
    local expiry_epoch; expiry_epoch=$(date -d "$expiry_date" +%s 2>/dev/null || date -j -f "%b %d %T %Y %Z" "$expiry_date" +%s 2>/dev/null)
    echo $(( (expiry_epoch - $(date +%s)) / 86400 ))
}

pki_verify_cert_chain() {
    local cert_file=$1 ca_file=${2:-${CA_DIR}/certs/ca.crt}
    openssl verify -CAfile "$ca_file" "$cert_file" 2>&1
}
```

---

## 77.3 Certificate Expiry Monitor

```bash
#!/bin/bash
# cert_monitor.sh - Certificate expiry monitoring

CERT_WARN_DAYS="${CERT_WARN_DAYS:-30}"
CERT_CRITICAL_DAYS="${CERT_CRITICAL_DAYS:-7}"

check_remote_cert() {
    local host=$1 port=${2:-443}

    local cert_info; cert_info=$(echo | openssl s_client \
        -servername "$host" \
        -connect "${host}:${port}" 2>/dev/null | \
        openssl x509 -noout -dates -subject 2>/dev/null)

    [[ -z "$cert_info" ]] && { echo "UNKNOWN $host: Cannot connect"; return 2; }

    local expiry_date; expiry_date=$(echo "$cert_info" | grep 'notAfter' | cut -d= -f2)
    local expiry_epoch; expiry_epoch=$(date -d "$expiry_date" +%s 2>/dev/null)
    local now; now=$(date +%s)
    local days_left=$(( (expiry_epoch - now) / 86400 ))

    local status
    if   (( days_left <= CERT_CRITICAL_DAYS )); then status="CRITICAL"
    elif (( days_left <= CERT_WARN_DAYS ));     then status="WARNING"
    else                                             status="OK"
    fi

    printf '%-10s %-30s %3d days  %s\n' "$status" "${host}:${port}" "$days_left" "$expiry_date"
    [[ "$status" == "OK" ]]
}

check_cert_files() {
    local cert_dir=${1:-.}
    local warn=0 critical=0 ok=0

    for cert_file in "$cert_dir"/*.crt "$cert_dir"/*.pem 2>/dev/null; do
        [[ -f "$cert_file" ]] || continue

        local days; days=$(pki_cert_expiry_days "$cert_file" 2>/dev/null || echo 9999)
        local status
        if   (( days <= CERT_CRITICAL_DAYS )); then status="CRITICAL"; (( critical++ ))
        elif (( days <= CERT_WARN_DAYS ));     then status="WARNING";  (( warn++ ))
        else                                        status="OK";        (( ok++ ))
        fi

        printf '%-10s %-40s %3d days\n' "$status" "$(basename "$cert_file")" "$days"
    done

    echo ""
    echo "OK=$ok WARNING=$warn CRITICAL=$critical"
    (( critical == 0 ))
}

cert_monitor_hosts() {
    local hosts_file=${1:-cert_hosts.txt}
    local failed=0

    echo "=== Certificate Monitor ==="
    while IFS=: read -r host port; do
        [[ "$host" =~ ^# || -z "$host" ]] && continue
        check_remote_cert "$host" "${port:-443}" || (( failed++ ))
    done < "$hosts_file"

    echo ""
    echo "Alerts: $failed"
    (( failed == 0 ))
}
```

---

## 77.4 HashiCorp Vault

```bash
#!/bin/bash
# vault.sh - HashiCorp Vault operations

VAULT_ADDR="${VAULT_ADDR:-http://127.0.0.1:8200}"

vault_api() {
    local method=$1 path=$2
    shift 2
    local body=${1:-}

    local -a curl_args=(
        --silent --fail
        --header "X-Vault-Token: ${VAULT_TOKEN:-}"
        --header 'Content-Type: application/json'
        --request "$method"
        "${VAULT_ADDR}/v1/${path}"
    )
    [[ -n "$body" ]] && curl_args+=(--data "$body")

    curl "${curl_args[@]}" 2>/dev/null
}

vault_login_token() {
    local token=$1
    export VAULT_TOKEN="$token"
}

vault_login_approle() {
    local role_id=$1 secret_id=$2
    local response; response=$(vault_api POST auth/approle/login \
        "{\"role_id\":\"$role_id\",\"secret_id\":\"$secret_id\"}" 2>/dev/null)
    VAULT_TOKEN=$(echo "$response" | jq -r '.auth.client_token')
    export VAULT_TOKEN
    echo "Logged in via AppRole"
}

vault_kv_get() {
    local path=$1 field=${2:-}
    local response; response=$(vault_api GET "secret/data/$path" 2>/dev/null)

    if [[ -n "$field" ]]; then
        echo "$response" | jq -r ".data.data.${field} // empty"
    else
        echo "$response" | jq '.data.data'
    fi
}

vault_kv_set() {
    local path=$1
    shift
    local -a kv_pairs=("$@")

    local data="{}"
    for pair in "${kv_pairs[@]}"; do
        local key="${pair%%=*}" value="${pair#*=}"
        data=$(echo "$data" | jq --arg k "$key" --arg v "$value" '. + {($k): $v}')
    done

    vault_api POST "secret/data/$path" "{\"data\":${data}}" > /dev/null
    echo "Set: $path"
}

vault_kv_delete() {
    local path=$1
    vault_api DELETE "secret/data/$path" > /dev/null
    echo "Deleted: $path"
}

vault_kv_list() {
    local path=${1:-}
    vault_api LIST "secret/metadata/$path" 2>/dev/null | jq -r '.data.keys[]'
}

vault_token_renew() {
    vault_api POST auth/token/renew-self > /dev/null
    echo "Token renewed"
}

vault_dynamic_db_creds() {
    local role=$1
    vault_api GET "database/creds/$role" 2>/dev/null | \
        jq '{username: .data.username, password: .data.password, lease_duration: .lease_duration}'
}

vault_inject_env() {
    local path=$1
    local secrets; secrets=$(vault_kv_get "$path" 2>/dev/null)

    if [[ -z "$secrets" || "$secrets" == "null" ]]; then
        echo "No secrets found at: $path" >&2
        return 1
    fi

    while IFS='=' read -r key value; do
        export "$key"="$value"
    done < <(echo "$secrets" | jq -r 'to_entries[] | .key + "=" + .value')

    echo "Injected secrets from: $path"
}
```

---

## 77.5 AWS Secrets Manager

```bash
#!/bin/bash
# aws_secrets.sh - AWS Secrets Manager and SSM operations

aws_sm_get_secret() {
    local secret_name=$1 field=${2:-}
    local region=${3:-${AWS_DEFAULT_REGION:-us-east-1}}

    local response; response=$(aws secretsmanager get-secret-value \
        --secret-id "$secret_name" \
        --region "$region" \
        --query SecretString \
        --output text 2>/dev/null)

    if [[ -z "$response" ]]; then
        echo "Secret not found: $secret_name" >&2
        return 1
    fi

    if [[ -n "$field" ]]; then
        echo "$response" | jq -r ".${field} // empty" 2>/dev/null || echo "$response"
    else
        echo "$response"
    fi
}

aws_sm_set_secret() {
    local secret_name=$1 value=$2 description=${3:-}
    local region=${AWS_DEFAULT_REGION:-us-east-1}

    # Try update first, create if not exists
    if ! aws secretsmanager describe-secret --secret-id "$secret_name" \
        --region "$region" &>/dev/null; then
        aws secretsmanager create-secret \
            --name "$secret_name" \
            --description "${description:-Created by script}" \
            --secret-string "$value" \
            --region "$region" > /dev/null
        echo "Created secret: $secret_name"
    else
        aws secretsmanager put-secret-value \
            --secret-id "$secret_name" \
            --secret-string "$value" \
            --region "$region" > /dev/null
        echo "Updated secret: $secret_name"
    fi
}

aws_ssm_get_param() {
    local param_name=$1 decrypt=${2:-true}
    local region=${AWS_DEFAULT_REGION:-us-east-1}

    aws ssm get-parameter \
        --name "$param_name" \
        --region "$region" \
        $(${decrypt} && echo '--with-decryption') \
        --query Parameter.Value \
        --output text 2>/dev/null
}

aws_ssm_put_param() {
    local param_name=$1 value=$2 param_type=${3:-SecureString}
    local region=${AWS_DEFAULT_REGION:-us-east-1}

    aws ssm put-parameter \
        --name "$param_name" \
        --value "$value" \
        --type "$param_type" \
        --overwrite \
        --region "$region" > /dev/null
    echo "SSM parameter set: $param_name"
}

aws_ssm_inject_env() {
    local prefix=$1
    local region=${AWS_DEFAULT_REGION:-us-east-1}

    aws ssm get-parameters-by-path \
        --path "$prefix" \
        --with-decryption \
        --region "$region" \
        --query 'Parameters[].{Name:Name,Value:Value}' \
        --output json 2>/dev/null | \
    jq -r '.[] | .Name + "=" + .Value' | \
    while IFS='=' read -r key value; do
        local env_key="${key##*/}" # strip prefix, keep last segment
        export "$env_key"="$value"
        echo "Injected: $env_key"
    done
}
```

---

## 77.6 SSH Key Management

```bash
#!/bin/bash
# ssh_keys.sh - SSH key generation and management

SSH_KEY_DIR="${SSH_KEY_DIR:-${HOME}/.ssh}"

ssh_generate_key() {
    local key_name=${1:-id_rsa} key_type=${2:-ed25519} comment=${3:-"$(whoami)@$(hostname)"}
    local key_file="${SSH_KEY_DIR}/${key_name}"
    local passphrase=${4:-}

    mkdir -p "$SSH_KEY_DIR"
    chmod 700 "$SSH_KEY_DIR"

    ssh-keygen \
        -t "$key_type" \
        -C "$comment" \
        -f "$key_file" \
        -N "$passphrase" \
        -q 2>/dev/null

    chmod 600 "$key_file"
    chmod 644 "${key_file}.pub"
    echo "Generated: $key_file ($key_type)"
    echo "Public key:"
    cat "${key_file}.pub"
}

ssh_add_authorized_key() {
    local user=$1 public_key=$2
    local auth_keys="/home/${user}/.ssh/authorized_keys"

    if [[ "$user" == "root" ]]; then
        auth_keys="/root/.ssh/authorized_keys"
    fi

    mkdir -p "$(dirname "$auth_keys")"
    chmod 700 "$(dirname "$auth_keys")"

    if grep -qF "$public_key" "$auth_keys" 2>/dev/null; then
        echo "Key already authorized for $user"
        return 0
    fi

    echo "$public_key" >> "$auth_keys"
    chmod 600 "$auth_keys"
    echo "Authorized key added for: $user"
}

ssh_remove_authorized_key() {
    local user=$1 key_comment=$2
    local auth_keys="/home/${user}/.ssh/authorized_keys"

    [[ -f "$auth_keys" ]] || return 0
    grep -v "$key_comment" "$auth_keys" > "${auth_keys}.tmp"
    mv "${auth_keys}.tmp" "$auth_keys"
    echo "Removed key for: $key_comment"
}

ssh_rotate_key() {
    local user=$1 key_name=${2:-id_ed25519}
    local key_file="/home/${user}/.ssh/${key_name}"
    local old_pub; old_pub=$(cat "${key_file}.pub" 2>/dev/null)

    # Generate new key
    ssh_generate_key "${key_name}_new" ed25519 "${user}@$(hostname)-$(date +%Y%m%d)"
    local new_pub; new_pub=$(cat "${SSH_KEY_DIR}/${key_name}_new.pub")

    # Add new, remove old
    ssh_add_authorized_key "$user" "$new_pub"
    [[ -n "$old_pub" ]] && ssh_remove_authorized_key "$user" "${old_pub##* }"

    mv "${SSH_KEY_DIR}/${key_name}_new" "${key_file}"
    mv "${SSH_KEY_DIR}/${key_name}_new.pub" "${key_file}.pub"
    echo "SSH key rotated for: $user"
}

ssh_fingerprint() {
    local key_file=$1
    ssh-keygen -lf "$key_file" 2>/dev/null
}

ssh_list_authorized_keys() {
    local user=$1
    local auth_keys="/home/${user}/.ssh/authorized_keys"
    [[ "$user" == "root" ]] && auth_keys="/root/.ssh/authorized_keys"

    [[ -f "$auth_keys" ]] || { echo "No authorized_keys for $user"; return; }

    echo "Authorized keys for $user:"
    while IFS= read -r key; do
        [[ "$key" =~ ^# || -z "$key" ]] && continue
        printf '  %s\n' "$(echo "$key" | awk '{print $3, substr($1,1,10)"..."}')" 
    done < "$auth_keys"
}
```

---

## 77.7 Secret Injection and Masking

```bash
#!/bin/bash
# secret_inject.sh - Secure secret injection into scripts

declare -a SECRET_NAMES=()

register_secret() {
    local var_name=$1
    SECRET_NAMES+=("$var_name")
}

mask_secrets_in_output() {
    local text=$1
    for name in "${SECRET_NAMES[@]}"; do
        local value="${!name:-}"
        [[ -n "$value" && ${#value} -gt 3 ]] && \
            text="${text//$value/[MASKED]}"
    done
    echo "$text"
}

load_secrets_from_file() {
    local secrets_file=$1
    local -a loaded=()

    if [[ ! -f "$secrets_file" ]]; then
        echo "Secrets file not found: $secrets_file" >&2
        return 1
    fi

    local perms; perms=$(stat -c '%a' "$secrets_file" 2>/dev/null || stat -f '%A' "$secrets_file" 2>/dev/null)
    if [[ "$perms" != "600" && "$perms" != "400" ]]; then
        echo "WARNING: Secrets file has permissive permissions: $perms" >&2
    fi

    while IFS='=' read -r key value; do
        [[ "$key" =~ ^# || -z "$key" ]] && continue
        export "$key"="$value"
        register_secret "$key"
        loaded+=("$key")
    done < "$secrets_file"

    echo "Loaded ${#loaded[@]} secrets"
}

secure_exec() {
    local cmd=("$@")
    local output exit_code=0

    output=$("${cmd[@]}" 2>&1) || exit_code=$?
    mask_secrets_in_output "$output"
    return $exit_code
}

wipe_secrets() {
    for name in "${SECRET_NAMES[@]}"; do
        printf -v "$name" '%*s' "${#!name}" '' 2>/dev/null || true
        unset "$name"
    done
    SECRET_NAMES=()
    echo "Secrets wiped from memory"
}

trap wipe_secrets EXIT
```

---

## 77.8 Exercises

### Exercise 1: Certificate Lifecycle Manager
สร้าง tool ที่:
- Issue certs from internal CA
- Track expiry in SQLite
- Auto-renew 30 days before
- Distribute via SCP to services

### Exercise 2: Secrets Rotation
สร้าง rotation pipeline ที่:
- Read from Vault
- Rotate DB password
- Update Vault
- Notify services (SIGHUP or API)
- Verify new creds work

### Exercise 3: SSH Key Audit
สร้าง audit ที่:
- List all authorized_keys on all servers
- Find orphan/expired keys
- Report by user/team
- Auto-revoke stale keys

---

## สรุป Part 77

✅ GPG: key generation, export/import, encrypt/decrypt file+string, sign/verify
┅ OpenSSL CA: init CA, issue server/client certs with SAN, PKCS12 bundle
┅ pki_cert_expiry_days, pki_verify_cert_chain, pki_cert_info
┅ Certificate monitor: remote host check, file check, host list file
┅ Vault: AppRole login, KV get/set/delete/list, token renew, dynamic DB creds
┅ vault_inject_env: export vault secrets to environment
┅ AWS Secrets Manager: get/set, SSM get/put/inject (path-based env injection)
┅ SSH: generate (ed25519), authorized_keys add/remove, key rotation, fingerprint
┅ Secret injection: register, mask in output, load from 600-perms file, wipe on EXIT

---

**→ Part 78: Automation Framework and Task Orchestration**
