# Part 76: Package Management and Dependency Automation
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 76.1 Cross-Distro Package Manager Abstraction

```bash
#!/bin/bash
# pkg.sh - Universal package manager wrapper

set -euo pipefail

PKG_LOG="${PKG_LOG:-/tmp/pkg_install.log}"

pkg_log() {
    printf '%s [PKG] %s\n' "$(date '+%Y-%m-%dT%H:%M:%S')" "$1" | tee -a "$PKG_LOG"
}

# ─── Detect package manager ──────────────────────────────────────────────────
detect_pkg_manager() {
    if command -v apt-get &>/dev/null; then echo apt
    elif command -v dnf     &>/dev/null; then echo dnf
    elif command -v yum     &>/dev/null; then echo yum
    elif command -v pacman  &>/dev/null; then echo pacman
    elif command -v apk     &>/dev/null; then echo apk
    elif command -v brew    &>/dev/null; then echo brew
    elif command -v zypper  &>/dev/null; then echo zypper
    else echo "unknown"; return 1
    fi
}

PKG_MANAGER="${PKG_MANAGER:-$(detect_pkg_manager)}"

pkg_update_cache() {
    pkg_log "Updating package cache ($PKG_MANAGER)"
    case "$PKG_MANAGER" in
        apt)    DEBIAN_FRONTEND=noninteractive apt-get update -qq ;;
        dnf)    dnf makecache -q ;;
        yum)    yum makecache -q ;;
        pacman) pacman -Sy --noconfirm &>/dev/null ;;
        apk)    apk update -q ;;
        brew)   brew update &>/dev/null ;;
        zypper) zypper refresh -q ;;
    esac
}

pkg_install() {
    local -a packages=("$@")
    pkg_log "Installing: ${packages[*]}"

    case "$PKG_MANAGER" in
        apt)    DEBIAN_FRONTEND=noninteractive apt-get install -y -qq "${packages[@]}" ;;
        dnf)    dnf install -y -q "${packages[@]}" ;;
        yum)    yum install -y -q "${packages[@]}" ;;
        pacman) pacman -S --noconfirm --needed "${packages[@]}" &>/dev/null ;;
        apk)    apk add -q "${packages[@]}" ;;
        brew)   brew install "${packages[@]}" &>/dev/null ;;
        zypper) zypper install -y -q "${packages[@]}" ;;
        *)      pkg_log "Unknown package manager: $PKG_MANAGER"; return 1 ;;
    esac
}

pkg_remove() {
    local -a packages=("$@")
    pkg_log "Removing: ${packages[*]}"

    case "$PKG_MANAGER" in
        apt)    DEBIAN_FRONTEND=noninteractive apt-get remove -y -qq "${packages[@]}" ;;
        dnf)    dnf remove -y -q "${packages[@]}" ;;
        yum)    yum remove -y -q "${packages[@]}" ;;
        pacman) pacman -R --noconfirm "${packages[@]}" &>/dev/null ;;
        apk)    apk del -q "${packages[@]}" ;;
        brew)   brew uninstall "${packages[@]}" &>/dev/null ;;
        zypper) zypper remove -y -q "${packages[@]}" ;;
    esac
}

pkg_is_installed() {
    local pkg=$1

    case "$PKG_MANAGER" in
        apt)    dpkg -l "$pkg" 2>/dev/null | grep -q '^ii' ;;
        dnf|yum) rpm -q "$pkg" &>/dev/null ;;
        pacman) pacman -Qi "$pkg" &>/dev/null ;;
        apk)    apk info -e "$pkg" &>/dev/null ;;
        brew)   brew list "$pkg" &>/dev/null ;;
        *) command -v "$pkg" &>/dev/null ;;
    esac
}

pkg_installed_version() {
    local pkg=$1

    case "$PKG_MANAGER" in
        apt)    dpkg-query -W -f='${Version}' "$pkg" 2>/dev/null ;;
        dnf|yum) rpm -q --queryformat '%{VERSION}-%{RELEASE}' "$pkg" 2>/dev/null ;;
        pacman) pacman -Q "$pkg" 2>/dev/null | awk '{print $2}' ;;
        apk)    apk info "$pkg" 2>/dev/null | grep -oP '\S+-\d[\d.]+' | head -1 ;;
        brew)   brew info "$pkg" 2>/dev/null | head -1 | awk '{print $3}' ;;
    esac
}

pkg_ensure() {
    local pkg=$1 min_version=${2:-}

    if pkg_is_installed "$pkg"; then
        pkg_log "Already installed: $pkg"
        return 0
    fi

    pkg_install "$pkg"
}

pkg_ensure_list() {
    local packages_file=$1
    local installed=0 skipped=0 failed=0

    while IFS= read -r line; do
        [[ "$line" =~ ^# || -z "$line" ]] && continue
        local pkg="${line%%=*}"
        if pkg_ensure "$pkg"; then
            (( installed++ ))
        else
            (( failed++ ))
        fi
    done < "$packages_file"

    pkg_log "Ensure complete: installed=$installed failed=$failed"
    (( failed == 0 ))
}
```

---

## 76.2 Version Pinning

```bash
#!/bin/bash
# version_pin.sh - Pinned package installations

pkg_install_pinned_apt() {
    local pkg=$1 version=$2

    if pkg_is_installed "$pkg"; then
        local installed; installed=$(pkg_installed_version "$pkg")
        if [[ "$installed" == *"$version"* ]]; then
            pkg_log "Already at correct version: $pkg=$installed"
            return 0
        fi
    fi

    pkg_log "Installing pinned: ${pkg}=${version}"
    DEBIAN_FRONTEND=noninteractive apt-get install -y -qq "${pkg}=${version}" 2>/dev/null || \
    apt-get install -y -qq "${pkg}" 2>/dev/null

    # Pin to prevent unwanted upgrades
    cat > "/etc/apt/preferences.d/${pkg}" << EOF
Package: $pkg
Pin: version $version
Pin-Priority: 1001
EOF
}

apt_hold() {
    local pkg=$1
    apt-mark hold "$pkg" 2>/dev/null
    pkg_log "Held: $pkg"
}

apt_unhold() {
    local pkg=$1
    apt-mark unhold "$pkg" 2>/dev/null
    pkg_log "Unheld: $pkg"
}

apt_list_held() {
    apt-mark showhold 2>/dev/null
}

pkg_snapshot_versions() {
    local output_file=${1:-packages.lock}

    case "$PKG_MANAGER" in
        apt)
            dpkg-query -W -f='${Package}=${Version}\n' | sort > "$output_file"
            ;;
        dnf|yum)
            rpm -qa --queryformat '%{NAME}=%{VERSION}-%{RELEASE}\n' | sort > "$output_file"
            ;;
        pacman)
            pacman -Q | tr ' ' '=' | sort > "$output_file"
            ;;
    esac
    pkg_log "Snapshot saved: $output_file ($(wc -l < "$output_file") packages)"
}

pkg_diff_from_snapshot() {
    local snapshot_file=$1
    local current_file; current_file=$(mktemp)
    pkg_snapshot_versions "$current_file" 2>/dev/null
    diff "$snapshot_file" "$current_file" | grep '^[<>]' || echo "No changes"
    rm -f "$current_file"
}
```

---

## 76.3 Language Package Manager Wrappers

```bash
#!/bin/bash
# lang_pkg.sh - Language-specific package managers

# ─── Python / pip ───────────────────────────────────────────────────────────
pip_install() {
    local -a packages=("$@")
    local pip_cmd; pip_cmd=$(command -v pip3 || command -v pip || echo pip)

    "$pip_cmd" install --quiet "${packages[@]}" 2>/dev/null
}

pip_install_requirements() {
    local req_file=${1:-requirements.txt}
    local pip_cmd; pip_cmd=$(command -v pip3 || command -v pip || echo pip)

    "$pip_cmd" install --quiet -r "$req_file" 2>/dev/null
    pkg_log "pip requirements installed from: $req_file"
}

pip_freeze_sorted() {
    local output=${1:-requirements.txt}
    pip3 freeze 2>/dev/null | sort > "$output"
    echo "Frozen: $output"
}

create_venv() {
    local venv_dir=${1:-.venv}
    python3 -m venv "$venv_dir"
    source "${venv_dir}/bin/activate"
    pkg_log "Virtualenv: $venv_dir"
}

# ─── Node / npm ─────────────────────────────────────────────────────────────
npm_install() {
    local -a flags=(--silent)
    npm install "${flags[@]}" "$@" 2>/dev/null
}

npm_install_ci() {
    npm ci --silent 2>/dev/null
    pkg_log "npm ci complete"
}

npm_outdated_json() {
    npm outdated --json 2>/dev/null || echo '{}'
}

npm_audit_fix() {
    npm audit fix --silent 2>/dev/null
    pkg_log "npm audit fix applied"
}

# ─── Go modules ──────────────────────────────────────────────────────────────
go_mod_tidy() {
    go mod tidy 2>/dev/null
    pkg_log "go mod tidy complete"
}

go_mod_vendor() {
    go mod vendor 2>/dev/null
    pkg_log "go mod vendor complete"
}

go_mod_verify() {
    go mod verify 2>/dev/null
}

go_install_tool() {
    local tool_url=$1 version=${2:-latest}
    go install "${tool_url}@${version}" 2>/dev/null
    pkg_log "Installed go tool: $tool_url@$version"
}

# ─── Ruby / gem ─────────────────────────────────────────────────────────────
bundle_install() {
    bundle install --quiet 2>/dev/null
    pkg_log "bundle install complete"
}

bundle_update() {
    local gem=${1:-}
    if [[ -n "$gem" ]]; then
        bundle update "$gem" 2>/dev/null
    else
        bundle update 2>/dev/null
    fi
    pkg_log "bundle update complete"
}
```

---

## 76.4 Dependency Requirements File

```bash
#!/bin/bash
# deps.sh - Dependency requirements management

DEPS_FILE="${DEPS_FILE:-deps.txt}"

# deps.txt format:
# os:curl
# os:jq
# python:requests>=2.25
# node:axios@^1.0
# go:github.com/spf13/cobra@v1.7

parse_deps_file() {
    local file=${1:-$DEPS_FILE}

    while IFS=: read -r manager package; do
        [[ "$manager" =~ ^# || -z "$manager" ]] && continue
        echo "$manager" "$package"
    done < "$file"
}

install_deps_from_file() {
    local file=${1:-$DEPS_FILE}
    local ok=0 failed=0

    pkg_log "Installing deps from: $file"

    while IFS=' ' read -r manager package; do
        local exit_code=0
        case "$manager" in
            os)     pkg_ensure "$package" || exit_code=$? ;;
            python) pip_install "$package" || exit_code=$? ;;
            node)   npm_install "$package" || exit_code=$? ;;
            go)     go_install_tool "$package" || exit_code=$? ;;
            *) pkg_log "Unknown manager: $manager"; exit_code=1 ;;
        esac

        if (( exit_code == 0 )); then
            pkg_log "  OK: ${manager}:${package}"
            (( ok++ ))
        else
            pkg_log "  FAIL: ${manager}:${package}"
            (( failed++ ))
        fi
    done < <(parse_deps_file "$file")

    pkg_log "Deps installed: ok=$ok failed=$failed"
    (( failed == 0 ))
}

check_deps_file() {
    local file=${1:-$DEPS_FILE}
    local missing=()

    while IFS=' ' read -r manager package; do
        local pkg_name="${package%%[>=<@]*}"
        case "$manager" in
            os) pkg_is_installed "$pkg_name" || missing+=("os:$pkg_name") ;;
            python) python3 -c "import ${pkg_name//-/_}" 2>/dev/null || missing+=("python:$pkg_name") ;;
            node) [[ -d "node_modules/${pkg_name}" ]] || missing+=("node:$pkg_name") ;;
        esac
    done < <(parse_deps_file "$file")

    if (( ${#missing[@]} > 0 )); then
        echo "Missing dependencies:"
        printf '  %s\n' "${missing[@]}"
        return 1
    fi

    echo "All dependencies satisfied"
}
```

---

## 76.5 Vendor / Air-Gap Bundling

```bash
#!/bin/bash
# vendor.sh - Offline/air-gap package bundling

VENDOR_DIR="${VENDOR_DIR:-./vendor}"

mkdir -p "$VENDOR_DIR"

apt_download_packages() {
    local -a packages=("$@")
    local deb_dir="${VENDOR_DIR}/debs"
    mkdir -p "$deb_dir"

    pkg_log "Downloading packages (with deps): ${packages[*]}"
    apt-get download $(apt-get install --print-uris -qq "${packages[@]}" 2>/dev/null | \
        grep -oP "'\K[^']+(?=')" | tr '\n' ' ') 2>/dev/null
    mv ./*.deb "$deb_dir/" 2>/dev/null || true

    pkg_log "Downloaded to: $deb_dir"
}

apt_install_from_vendor() {
    local deb_dir="${VENDOR_DIR}/debs"
    dpkg -i "${deb_dir}"/*.deb 2>/dev/null || \
        apt-get install -f -y -qq 2>/dev/null
    pkg_log "Installed from vendor: $deb_dir"
}

pip_download_packages() {
    local -a packages=("$@")
    local wheel_dir="${VENDOR_DIR}/wheels"
    mkdir -p "$wheel_dir"

    pip3 download --quiet --dest "$wheel_dir" "${packages[@]}" 2>/dev/null
    pkg_log "Downloaded to: $wheel_dir"
}

pip_install_from_vendor() {
    local wheel_dir="${VENDOR_DIR}/wheels"
    pip3 install --quiet --no-index --find-links "$wheel_dir" -r requirements.txt 2>/dev/null
    pkg_log "Installed from vendor: $wheel_dir"
}

npm_pack_offline() {
    local -a packages=("$@")
    local tgz_dir="${VENDOR_DIR}/npm"
    mkdir -p "$tgz_dir"

    for pkg in "${packages[@]}"; do
        npm pack "$pkg" --pack-destination "$tgz_dir" --quiet 2>/dev/null
    done
    pkg_log "npm packs in: $tgz_dir"
}

create_vendor_bundle() {
    local bundle_name="${1:-vendor_bundle}"
    local archive="${bundle_name}_$(date +%Y%m%d).tar.gz"

    tar -czf "$archive" -C "$(dirname "$VENDOR_DIR")" "$(basename "$VENDOR_DIR")"
    local size; size=$(du -h "$archive" | cut -f1)
    pkg_log "Vendor bundle: $archive ($size)"
}
```

---

## 76.6 Package Audit

```bash
#!/bin/bash
# pkg_audit.sh - Package security and compliance audit

apt_outdated_count() {
    apt list --upgradable 2>/dev/null | grep -c 'upgradable' || echo 0
}

apt_security_updates() {
    apt-get -s upgrade 2>/dev/null | grep -i 'security' | \
        awk '{print $2}' | sort -u
}

pip_outdated() {
    pip3 list --outdated --format=columns 2>/dev/null | tail -n +3
}

npm_vulnerabilities() {
    npm audit --json 2>/dev/null | \
        jq -r '.vulnerabilities | to_entries[] | "\(.key): \(.value.severity)"' 2>/dev/null || \
        echo "npm audit not available"
}

pkg_license_check() {
    local dir=${1:-.}
    local allowed_licenses=("MIT" "Apache-2.0" "BSD-2-Clause" "BSD-3-Clause" "ISC")

    if [[ -f "${dir}/package.json" ]] && command -v license-checker &>/dev/null; then
        license-checker --start "$dir" --production --csv 2>/dev/null | \
        while IFS=',' read -r pkg version license _; do
            local found=false
            for allowed in "${allowed_licenses[@]}"; do
                [[ "$license" == *"$allowed"* ]] && found=true && break
            done
            $found || echo "NON-APPROVED LICENSE: $pkg $version: $license"
        done
    else
        echo "License check requires license-checker (npm install -g license-checker)"
    fi
}

pkg_size_analysis() {
    case "$PKG_MANAGER" in
        apt)
            dpkg-query -W -f='${Installed-Size}\t${Package}\n' 2>/dev/null | \
                sort -rn | head -20 | \
                awk '{printf "%8.1f MB\t%s\n", $1/1024, $2}'
            ;;
        brew)
            brew list --formula 2>/dev/null | while read -r pkg; do
                local size; size=$(brew info --json "$pkg" 2>/dev/null | \
                    jq -r '.[0].installed[0].installed_on_request // false')
                echo "$size $pkg"
            done | sort -rn | head -20
            ;;
    esac
}

pkg_auto_update() {
    local dry_run=${1:-false}
    pkg_log "Auto-update (dry_run=$dry_run)"

    case "$PKG_MANAGER" in
        apt)
            pkg_update_cache
            if $dry_run; then
                apt-get -s upgrade 2>/dev/null | grep '^Inst'
            else
                DEBIAN_FRONTEND=noninteractive apt-get upgrade -y -qq 2>/dev/null
            fi
            ;;
        dnf)
            if $dry_run; then
                dnf check-update 2>/dev/null || true
            else
                dnf upgrade -y -q 2>/dev/null
            fi
            ;;
        brew)
            brew upgrade 2>/dev/null
            ;;
    esac
}
```

---

## 76.7 Installation Bootstrap Script

```bash
#!/bin/bash
# bootstrap.sh - System bootstrap from scratch

BOOTSTRAP_LOG="/tmp/bootstrap_$(date +%Y%m%d_%H%M%S).log"

bootstrap_log() {
    printf '%s %s\n' "$(date '+%H:%M:%S')" "$1" | tee -a "$BOOTSTRAP_LOG"
}

bootstrap_step() {
    local name=$1
    shift
    local cmd=("$@")

    bootstrap_log "STEP: $name"
    if "${cmd[@]}" >> "$BOOTSTRAP_LOG" 2>&1; then
        bootstrap_log "  OK: $name"
        return 0
    else
        bootstrap_log "  FAIL: $name"
        return 1
    fi
}

bootstrap_require_root() {
    (( EUID == 0 )) || { echo "This script requires root"; exit 1; }
}

bootstrap_check_os() {
    if [[ -f /etc/os-release ]]; then
        source /etc/os-release
        bootstrap_log "OS: $PRETTY_NAME"
        bootstrap_log "Arch: $(uname -m)"
    fi
}

bootstrap_install_base_tools() {
    local base_tools=(curl wget git jq unzip tar gzip bzip2 openssl ca-certificates)

    pkg_update_cache
    for tool in "${base_tools[@]}"; do
        pkg_ensure "$tool" || true
    done
    bootstrap_log "Base tools installed"
}

bootstrap_setup_user() {
    local username=$1 uid=${2:-1000}
    local groups=(${3:-sudo docker})

    if ! id "$username" &>/dev/null; then
        useradd --create-home --uid "$uid" --shell /bin/bash "$username"
        bootstrap_log "Created user: $username"
    fi

    for grp in "${groups[@]}"; do
        getent group "$grp" &>/dev/null && usermod -aG "$grp" "$username" || true
    done
}

run_bootstrap() {
    bootstrap_check_os
    bootstrap_install_base_tools
    bootstrap_log "Bootstrap complete! Log: $BOOTSTRAP_LOG"
}
```

---

## 76.8 Exercises

### Exercise 1: Multi-Lang Dependency Installer
สร้าง installer ที่:
- อ่าน deps.txt สำหรับ OS + Python + Node + Go
- Parallel install per language
- Report summary
- Fail fast on critical deps

### Exercise 2: Reproducible Environment
สร้าง tool ที่:
- Snapshot current versions
- Create lock file
- Restore exact versions
- Diff between two environments

### Exercise 3: Air-Gap Deployment Kit
สร้าง kit ที่:
- Download all deps with deps
- Bundle into tar.gz
- Install script for air-gapped target
- Checksum verification

---

## สรุป Part 76

✅ detect_pkg_manager(): apt/dnf/yum/pacman/apk/brew/zypper
┅ pkg_install/remove/is_installed/installed_version: cross-distro API
┅ pkg_ensure/ensure_list: idempotent install from file
┅ Version pinning: apt preferences, hold/unhold, snapshot+diff
┅ Python: pip install/requirements/freeze, venv
┅ Node: npm install/ci/outdated/audit-fix
┅ Go: mod tidy/vendor/verify, install tool
┅ Ruby: bundle install/update
┅ Deps file: multi-manager format, parse, install, check
┅ Vendor/air-gap: apt download, pip download, npm pack, tar bundle
┅ Audit: outdated, security updates, npm vulns, license check, size analysis
┅ Bootstrap: step-logger, base tools, user setup

---

**→ Part 77: Secrets, Certificates, and PKI Automation**
