# Part 15: Package Management
## หลักสูตร Bash/Shell Script ระดับมืออาชีพ

---

## 15.1 APT (Debian/Ubuntu)

```bash
# ─── Repository Management ────────────────────────────────────
apt update                      # update package lists
apt upgrade                     # upgrade installed packages
apt full-upgrade                # upgrade + handle dependencies
apt dist-upgrade                # distribution upgrade

# Install/Remove
apt install package             # install
apt install package1 package2   # multiple packages
apt install package=1.2.3       # specific version
apt install -y package          # no confirmation
apt remove package              # remove (keep config)
apt purge package               # remove + config files
apt autoremove                  # remove orphaned packages
apt autoclean                   # clean old package cache

# Search & Info
apt search keyword              # search packages
apt show package                # package details
apt list --installed            # list installed
apt list --upgradable           # list upgradable

# ─── dpkg (low-level) ─────────────────────────────────────────
dpkg -l                         # list installed
dpkg -l | grep package          # check if installed
dpkg -s package                 # package status
dpkg -L package                 # list files in package
dpkg -S /path/to/file           # which package owns file
dpkg -i package.deb             # install .deb file
dpkg -r package                 # remove
dpkg -P package                 # purge

# ─── apt-cache ────────────────────────────────────────────────
apt-cache search keyword
apt-cache show package
apt-cache policy package        # available versions
apt-cache depends package       # dependencies
apt-cache rdepends package      # reverse dependencies

# ─── Repositories ─────────────────────────────────────────────
# List sources
cat /etc/apt/sources.list
ls /etc/apt/sources.list.d/

# Add repository
add-apt-repository ppa:user/ppa
add-apt-repository "deb http://repo.example.com/apt stable main"

# Add key
curl -fsSL https://example.com/key.gpg | sudo apt-key add -
# Modern way (apt-key deprecated):
curl -fsSL https://example.com/key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/myapp.gpg
echo "deb [signed-by=/usr/share/keyrings/myapp.gpg] https://example.com/apt stable main" | \
    sudo tee /etc/apt/sources.list.d/myapp.list

apt update
apt install myapp
```

---

## 15.2 YUM/DNF (Red Hat/CentOS/Fedora)

```bash
# ─── DNF (Fedora/RHEL 8+) ────────────────────────────────────
dnf install package
dnf remove package
dnf update                      # update all
dnf update package              # update specific
dnf search keyword
dnf info package
dnf list installed
dnf list available | grep keyword
dnf autoremove
dnf clean all

# Groups
dnf group list
dnf group install "Development Tools"
dnf group remove "Development Tools"

# History
dnf history
dnf history info 5
dnf history undo 5              # undo transaction 5

# ─── YUM (RHEL 7/CentOS 7) ────────────────────────────────────
yum install package
yum remove package
yum update
yum search keyword
yum list installed
yum provides /path/to/file      # find which package provides file
yum info package

# ─── RPM (low-level) ──────────────────────────────────────────
rpm -qa                         # list all installed
rpm -qi package                 # package info
rpm -ql package                 # list files
rpm -qf /path/to/file           # which package owns file
rpm -i package.rpm              # install
rpm -e package                  # erase/remove
rpm -V package                  # verify package
rpm -K package.rpm              # verify signature

# ─── Repositories ─────────────────────────────────────────────
dnf repolist                    # list repos
ls /etc/yum.repos.d/

# Add EPEL
dnf install epel-release

# Add custom repo
cat > /etc/yum.repos.d/myapp.repo << 'EOF'
[myapp]
name=My Application Repository
baseurl=https://repo.example.com/rhel/$releasever/$basearch/
enabled=1
gpgcheck=1
gpgkey=https://repo.example.com/gpg.key
EOF
```

---

## 15.3 Snap, Flatpak & AppImage

```bash
# ─── Snap ─────────────────────────────────────────────────────
snap find package               # search
snap install package            # install
snap install package --classic  # classic confinement
snap remove package             # remove
snap list                       # list installed
snap info package               # package info
snap refresh                    # update all snaps
snap refresh package            # update specific
snap revert package             # revert to previous version

# Snap channels
snap install package --channel=edge
snap install package --channel=beta
snap install package --channel=stable

# ─── Flatpak ──────────────────────────────────────────────────
# Add Flathub repository
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

flatpak search package
flatpak install flathub com.app.Name
flatpak remove com.app.Name
flatpak list
flatpak update
flatpak info com.app.Name

# Run
flatpak run com.app.Name

# ─── AppImage ─────────────────────────────────────────────────
# Self-contained executables, no installation needed
chmod +x app.AppImage
./app.AppImage

# Extract AppImage
./app.AppImage --appimage-extract

# AppImageLauncher for integration
apt install appimagelauncher
```

---

## 15.4 Language-specific Package Managers

```bash
# ─── Python (pip/pipx) ────────────────────────────────────────
pip install package
pip install package==1.2.3
pip install -r requirements.txt
pip install --user package       # user install
pip list
pip list --outdated
pip upgrade pip
pip freeze > requirements.txt
pip uninstall package

# pip virtual environments
python3 -m venv myenv
source myenv/bin/activate
pip install package
deactivate

# pipx (isolated CLI tools)
pipx install tool
pipx upgrade tool
pipx list
pipx run tool  # run without installing

# ─── Node.js (npm/yarn/pnpm) ──────────────────────────────────
npm install package
npm install -g package          # global
npm install --save-dev package  # dev dependency
npm update
npm list -g --depth=0           # list global
npm uninstall package
npm run script_name

# yarn
yarn add package
yarn add -D package             # dev
yarn global add package
yarn upgrade
yarn remove package

# pnpm
pnpm add package
pnpm add -D package
pnpm add -g package
pnpm update
pnpm remove package

# ─── Ruby (gem/bundler) ───────────────────────────────────────
gem install package
gem update
gem list
gem uninstall package
bundle install                  # from Gemfile
bundle update

# ─── Rust (cargo) ─────────────────────────────────────────────
cargo install tool
cargo install --list
cargo uninstall tool
rustup update                   # update Rust toolchain

# ─── Go ───────────────────────────────────────────────────────
go install tool@latest
go install tool@v1.2.3
# Installed to $GOPATH/bin or $HOME/go/bin
```

---

## 15.5 Universal Package Manager Script

```bash
#!/bin/bash
# pkg.sh - Universal package manager wrapper

set -euo pipefail

detect_pm() {
    if command -v apt &>/dev/null; then
        echo "apt"
    elif command -v dnf &>/dev/null; then
        echo "dnf"
    elif command -v yum &>/dev/null; then
        echo "yum"
    elif command -v pacman &>/dev/null; then
        echo "pacman"
    elif command -v zypper &>/dev/null; then
        echo "zypper"
    elif command -v brew &>/dev/null; then
        echo "brew"
    else
        echo "unknown"
    fi
}

PM=$(detect_pm)

pkg_install() {
    local pkg=$1
    echo "Installing $pkg with $PM..."
    case $PM in
        apt)    sudo apt-get install -y "$pkg" ;;
        dnf)    sudo dnf install -y "$pkg" ;;
        yum)    sudo yum install -y "$pkg" ;;
        pacman) sudo pacman -S --noconfirm "$pkg" ;;
        zypper) sudo zypper install -y "$pkg" ;;
        brew)   brew install "$pkg" ;;
        *)      echo "Unknown package manager" ; return 1 ;;
    esac
}

pkg_remove() {
    local pkg=$1
    case $PM in
        apt)    sudo apt-get remove -y "$pkg" ;;
        dnf)    sudo dnf remove -y "$pkg" ;;
        yum)    sudo yum remove -y "$pkg" ;;
        pacman) sudo pacman -R --noconfirm "$pkg" ;;
        zypper) sudo zypper remove -y "$pkg" ;;
        brew)   brew uninstall "$pkg" ;;
        *)      echo "Unknown package manager" ; return 1 ;;
    esac
}

pkg_update() {
    case $PM in
        apt)    sudo apt-get update && sudo apt-get upgrade -y ;;
        dnf)    sudo dnf update -y ;;
        yum)    sudo yum update -y ;;
        pacman) sudo pacman -Syu --noconfirm ;;
        zypper) sudo zypper update -y ;;
        brew)   brew update && brew upgrade ;;
        *)      echo "Unknown package manager" ; return 1 ;;
    esac
}

pkg_search() {
    local keyword=$1
    case $PM in
        apt)    apt-cache search "$keyword" ;;
        dnf)    dnf search "$keyword" ;;
        yum)    yum search "$keyword" ;;
        pacman) pacman -Ss "$keyword" ;;
        zypper) zypper search "$keyword" ;;
        brew)   brew search "$keyword" ;;
        *)      echo "Unknown package manager" ; return 1 ;;
    esac
}

pkg_installed() {
    local pkg=$1
    case $PM in
        apt)    dpkg -l "$pkg" &>/dev/null ;;
        dnf|yum) rpm -q "$pkg" &>/dev/null ;;
        pacman) pacman -Qi "$pkg" &>/dev/null ;;
        brew)   brew list "$pkg" &>/dev/null ;;
        *)      return 1 ;;
    esac
}

# Install multiple with check
ensure_packages() {
    local packages=("$@")
    local to_install=()
    
    for pkg in "${packages[@]}"; do
        if pkg_installed "$pkg"; then
            echo "✓ $pkg already installed"
        else
            to_install+=("$pkg")
        fi
    done
    
    if [[ ${#to_install[@]} -gt 0 ]]; then
        echo "Installing: ${to_install[*]}"
        for pkg in "${to_install[@]}"; do
            pkg_install "$pkg"
        done
    fi
}

# Main
case "${1:-help}" in
    install|i)   pkg_install "$2" ;;
    remove|r)    pkg_remove "$2" ;;
    update|u)    pkg_update ;;
    search|s)    pkg_search "$2" ;;
    ensure)      shift; ensure_packages "$@" ;;
    detect)      echo "Package manager: $PM" ;;
    help)
        echo "Usage: pkg [install|remove|update|search|ensure|detect] [package]"
        ;;
esac
```

---

## 15.6 Dependency Checker Script

```bash
#!/bin/bash
# check_deps.sh - Check and install dependencies

REQUIRED_COMMANDS=(
    curl
    wget
    jq
    git
    python3
    docker
    kubectl
)

REQUIRED_PACKAGES=(
    "curl:curl"
    "wget:wget"
    "jq:jq"
    "git:git"
    "python3:python3"
)

check_command() {
    local cmd=$1
    if command -v "$cmd" &>/dev/null; then
        local version
        version=$("$cmd" --version 2>&1 | head -1)
        echo -e "\033[32m✓\033[0m $cmd: $version"
        return 0
    else
        echo -e "\033[31m✗\033[0m $cmd: NOT FOUND"
        return 1
    fi
}

check_version() {
    local cmd=$1
    local min_version=$2
    local actual
    actual=$("$cmd" --version 2>&1 | grep -oP '\d+\.\d+' | head -1)
    
    if [[ "$(printf '%s\n' "$min_version" "$actual" | sort -V | head -1)" == "$min_version" ]]; then
        echo -e "\033[32m✓\033[0m $cmd $actual >= $min_version"
        return 0
    else
        echo -e "\033[31m✗\033[0m $cmd $actual < $min_version required"
        return 1
    fi
}

echo "=== Dependency Check ==="
missing=()

for cmd in "${REQUIRED_COMMANDS[@]}"; do
    check_command "$cmd" || missing+=("$cmd")
done

if [[ ${#missing[@]} -gt 0 ]]; then
    echo ""
    echo "Missing: ${missing[*]}"
    echo "Run with --install to attempt installation"
fi

if [[ "${1:-}" == "--install" ]]; then
    for entry in "${REQUIRED_PACKAGES[@]}"; do
        cmd="${entry%%:*}"
        pkg="${entry##*:}"
        if ! command -v "$cmd" &>/dev/null; then
            echo "Installing $pkg..."
            sudo apt-get install -y "$pkg" 2>/dev/null || \
            sudo yum install -y "$pkg" 2>/dev/null || \
            brew install "$pkg" 2>/dev/null || \
            echo "Failed to install $pkg"
        fi
    done
fi
```

---

## 15.7 Exercises

### Exercise 1: Software Inventory
สร้าง script ที่:
- List ทุก installed package
- Check update availability
- Export เป็น CSV/JSON
- Compare สอง servers

### Exercise 2: Automated Setup Script
สร้าง script ที่ setup development environment:
- Install required tools
- Configure settings
- Clone repositories
- Works on Ubuntu/CentOS/macOS

### Exercise 3: Package Version Manager
สร้าง script ที่:
- Pin versions ของ packages
- Alert เมื่อ updates มี security fix
- Rollback ไปยัง previous version

---

## สรุป Part 15

✅ APT (Debian/Ubuntu) commands  
✅ YUM/DNF (Red Hat) commands  
✅ Snap, Flatpak, AppImage  
✅ Language package managers (pip, npm, cargo, gem)  
✅ Universal package manager wrapper  
✅ Dependency checker script  

---

**→ Part 16: Networking Basics**
