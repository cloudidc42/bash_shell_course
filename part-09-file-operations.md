# Part 09: File Operations
## หลักสูตร Bash/Shell Script ระดับมืออาชีพ

---

## 9.1 File Creation & Basic Operations

```bash
#!/bin/bash

# Create empty file
touch newfile.txt
touch -t 202401151200 file.txt   # set timestamp

# Create with content
echo "content" > file.txt
cat > file.txt << 'EOF'
Line 1
Line 2
Line 3
EOF

printf "Hello\nWorld\n" > file.txt

# Copy files
cp source.txt dest.txt
cp -r srcdir/ destdir/       # recursive
cp -p file1 file2            # preserve permissions, timestamps
cp -a src/ dst/              # archive (preserve everything)
cp -i file1 file2            # interactive (prompt overwrite)
cp -u file1 file2            # only if source newer
cp -v *.txt /backup/         # verbose

# Move/Rename
mv old.txt new.txt
mv -i src dst                # interactive
mv -n src dst                # no overwrite
mv *.log /var/log/archive/

# Delete
rm file.txt
rm -f file.txt               # force (no error if not exist)
rm -r directory/             # recursive
rm -rf directory/            # ⚠️ force recursive (dangerous!)
rm -i *.txt                  # interactive

# Safe delete alternatives
trash-put file.txt           # send to trash (trash-cli)
mv file.txt ~/.Trash/        # manual trash
```

---

## 9.2 Directory Operations

```bash
# Create directories
mkdir mydir
mkdir -p path/to/nested/dir  # create parents
mkdir -m 755 secured_dir     # with permissions
mkdir dir1 dir2 dir3         # multiple

# Create with brace expansion
mkdir -p project/{src,tests,docs,bin}
mkdir -p app/{frontend/{components,pages},backend/{api,models}}

# Remove directories
rmdir emptydir               # only empty directories
rm -rf nonempty_dir/         # ⚠️ force remove

# Navigate
cd /path/to/dir
cd -                         # go back to previous
pushd /new/path              # save current and go to new
popd                         # return to saved path
dirs -v                      # show directory stack

# List directory contents
ls -la                       # all files, long format
ls -lhS                      # sorted by size, human-readable
ls -lt                       # sorted by time
ls -R                        # recursive
ls -d */                     # directories only
ls *.{txt,sh}                # multiple patterns

# Find files
find . -name "*.txt"
find /etc -type f -name "*.conf"
find . -newer file.txt
find . -mtime -7             # modified in last 7 days
find . -size +1M             # larger than 1MB
find . -empty                # empty files/dirs
find . -perm 644             # specific permissions
find . -user nobody          # owned by user

# File information
stat file.txt                # detailed info
file file.txt                # file type
wc -l file.txt               # line count
wc -c file.txt               # byte count
wc -w file.txt               # word count
du -sh directory/            # disk usage
du -sh *                     # all items

# Disk space
df -h                        # filesystem usage
df -h /home                  # specific path
lsblk                        # block devices
```

---

## 9.3 File Permissions

```bash
# Permission notation
# rw-r--r-- = 644
# rwxr-xr-x = 755
# rwxrwxrwx = 777
# r-------- = 400

# chmod - change permissions
chmod 755 script.sh          # numeric notation
chmod u+x script.sh          # symbolic: add execute for user
chmod go-w file.txt          # remove write for group,other
chmod a+r file.txt           # add read for all
chmod u=rw,g=r,o= file.txt   # set exact

# Ownership
chown user file.txt          # change owner
chown user:group file.txt    # change owner and group
chown -R user:group dir/     # recursive
chgrp group file.txt         # change group only

# Special permissions
chmod 4755 script            # setuid (s in user execute)
chmod 2755 directory         # setgid (s in group execute)
chmod 1777 /tmp              # sticky bit (t in other execute)

# umask - default permissions
umask                        # show current umask
umask 022                    # set umask (files=644, dirs=755)
umask 027                    # stricter (files=640, dirs=750)

# File attributes (lsattr/chattr)
lsattr file.txt
chattr +i file.txt           # immutable (can't modify/delete)
chattr -i file.txt           # remove immutable
chattr +a file.txt           # append only
chattr +e file.txt           # extent format

# ACL (Access Control Lists)
getfacl file.txt
setfacl -m u:username:rw file.txt    # add user ACL
setfacl -m g:groupname:r file.txt   # add group ACL
setfacl -x u:username file.txt      # remove user ACL
setfacl -b file.txt                 # remove all ACLs
```

---

## 9.4 File Content Viewing

```bash
file="/var/log/syslog"

# Basic viewing
cat file                     # all content
cat -n file                  # with line numbers
cat -A file                  # show special chars (tabs, newlines)

# Pager
less file                    # scroll up/down (q=quit, /=search)
more file                    # older pager
most file                    # enhanced less

# Head/Tail
head file                    # first 10 lines
head -n 20 file              # first 20 lines
head -c 100 file             # first 100 bytes
tail file                    # last 10 lines
tail -n 50 file              # last 50 lines
tail -f file                 # follow (real-time)
tail -F file                 # follow + reopen if rotated

# Watch file changes
watch -n 1 "tail -n 20 file"    # refresh every second
inotifywait -m -e modify file   # event-based watching

# Hex view
xxd file.txt                 # hex dump
xxd -l 64 file               # first 64 bytes
od -c file.txt               # octal dump with chars
hexdump -C file              # canonical hex+ASCII

# Differences
diff file1 file2             # line differences
diff -u file1 file2          # unified format
diff -r dir1/ dir2/          # recursive directory diff
colordiff file1 file2        # colored diff (if installed)
vimdiff file1 file2          # side-by-side in vim

# Count
wc -l file                   # lines
wc -w file                   # words
wc -c file                   # bytes
wc -m file                   # chars (respects locale)
```

---

## 9.5 File Searching

```bash
# find - comprehensive file search
find [path] [options] [expression]

# By name
find . -name "*.py"          # case sensitive
find . -iname "*.PY"         # case insensitive
find . -name "test_*"        # wildcard

# By type
find . -type f               # files
find . -type d               # directories
find . -type l               # symbolic links
find . -type p               # named pipes
find . -type s               # sockets

# By time
find . -mtime -7             # modified < 7 days ago
find . -mtime +30            # modified > 30 days ago
find . -mtime 0              # modified today
find . -newer reference.txt  # newer than reference
find . -atime -1             # accessed < 1 day ago
find . -ctime -1             # status changed < 1 day

# By size
find . -size +1M             # > 1 MB
find . -size -1k             # < 1 KB
find . -size 100c            # exactly 100 bytes
find . -empty                # empty files

# By permissions
find . -perm 644             # exactly 644
find . -perm /u+w            # user has write
find . -perm -g+x            # group has execute (all)

# By ownership
find . -user username
find . -group groupname
find . -uid 1000
find . -nouser               # no owner (deleted users)

# Execute actions
find . -name "*.log" -delete                    # delete matches
find . -name "*.txt" -exec cat {} \;            # run for each
find . -name "*.txt" -exec cat {} +             # combine args
find . -name "*.sh" -exec chmod +x {} \;        # make executable
find . -name "*.log" -print0 | xargs -0 rm      # safe with spaces

# Combine conditions
find . -name "*.log" -mtime +7 -delete
find . -type f \( -name "*.tmp" -o -name "*.bak" \)
find . -not -name "*.py"
find . -type f -size +1M -newer /tmp/timestamp

# locate (faster, uses database)
locate file.txt
locate -i "*.pdf"            # case insensitive
updatedb                     # update database (root)

# which/where
which bash                   # find in PATH
which -a python              # all occurrences
whereis ls                   # find binary, source, man
type ls                      # shell's understanding of command
```

---

## 9.6 File Processing Pipeline

```bash
# grep - search content
grep "pattern" file.txt
grep -i "pattern" file.txt   # case insensitive
grep -r "pattern" dir/       # recursive
grep -n "pattern" file.txt   # with line numbers
grep -c "pattern" file.txt   # count matches
grep -l "pattern" *.txt      # files with matches
grep -v "pattern" file.txt   # invert (not matching)
grep -o "pattern" file.txt   # only matching part
grep -A 3 "pattern" file.txt # 3 lines after match
grep -B 3 "pattern" file.txt # 3 lines before match
grep -C 3 "pattern" file.txt # 3 lines context

# Extended grep (egrep / grep -E)
grep -E "pattern1|pattern2" file.txt
grep -E "^[0-9]+" file.txt   # starts with digits
grep -P "(?<=:)\d+" file.txt  # Perl regex (lookahead/behind)

# sort
sort file.txt                # alphabetical
sort -r file.txt             # reverse
sort -n file.txt             # numeric
sort -k2 file.txt            # by field 2
sort -t: -k3 -n file.txt    # by field 3 (colon-delimited, numeric)
sort -u file.txt             # unique
sort -R file.txt             # random

# uniq
uniq file.txt                # remove consecutive duplicates
sort file.txt | uniq         # all duplicates (sort first)
sort file.txt | uniq -c     # count occurrences
sort file.txt | uniq -d     # only duplicates
sort file.txt | uniq -u     # only unique lines

# cut - extract fields
cut -d: -f1 /etc/passwd      # field 1 (username)
cut -d: -f1,3 /etc/passwd    # fields 1 and 3
cut -c1-10 file.txt          # chars 1-10
cut -c5- file.txt            # from char 5 to end

# tr - translate/delete
tr 'a-z' 'A-Z'               # uppercase
tr -d '\n'                   # remove newlines
tr -s ' '                    # squeeze spaces
tr -cd '[:print:]'           # keep only printable chars

# join - merge files on common field
join file1.txt file2.txt     # join on first field
join -1 2 -2 1 file1 file2  # join on specific fields

# paste - merge lines
paste file1.txt file2.txt    # side by side
paste -d',' file1 file2      # with delimiter

# comm - compare sorted files
comm file1.txt file2.txt     # 3 columns
comm -12 file1 file2         # only common lines
comm -23 file1 file2         # only in file1
```

---

## 9.7 File Encoding & Compression

```bash
# File encoding detection
file file.txt                # detect encoding
chardet file.txt             # (chardet tool)

# Convert encoding
iconv -f UTF-8 -t ISO-8859-1 input.txt > output.txt
iconv -f UTF-8 -t UTF-16 input.txt > output.utf16

# Line ending conversion
dos2unix file.txt            # CRLF → LF
unix2dos file.txt            # LF → CRLF
sed -i 's/\r$//' file.txt    # manual CRLF → LF

# Compression
# gzip
gzip file.txt                # compress (creates file.txt.gz, removes original)
gzip -k file.txt             # keep original
gzip -d file.txt.gz          # decompress
gunzip file.txt.gz           # decompress
gzip -l file.txt.gz          # list info
gzip -1 file.txt             # fast compression
gzip -9 file.txt             # best compression

# bzip2 (better compression, slower)
bzip2 file.txt
bunzip2 file.txt.bz2

# xz (best compression)
xz file.txt
unxz file.txt.xz

# zstd (fast + good compression)
zstd file.txt
unzstd file.txt.zst

# tar archives
tar -czf archive.tar.gz dir/         # create gzip archive
tar -cjf archive.tar.bz2 dir/        # create bzip2 archive
tar -cJf archive.tar.xz dir/         # create xz archive
tar -czf archive.tar.gz -C /base dir/ # relative paths

tar -tzf archive.tar.gz              # list contents
tar -xzf archive.tar.gz              # extract all
tar -xzf archive.tar.gz file.txt     # extract specific file
tar -xzf archive.tar.gz -C /dest/    # extract to destination
tar -xzf archive.tar.gz --strip-components=1  # remove top dir

# zip/unzip
zip archive.zip file1 file2
zip -r archive.zip directory/
unzip archive.zip
unzip archive.zip -d /dest/
unzip -l archive.zip         # list
unzip -p archive.zip file    # extract to stdout

# 7zip
7z a archive.7z files/       # create
7z x archive.7z              # extract
7z l archive.7z              # list
```

---

## 9.8 File Integrity

```bash
# Checksums
md5sum file.txt              # MD5 (fast, not secure)
sha1sum file.txt             # SHA1
sha256sum file.txt           # SHA256 (recommended)
sha512sum file.txt           # SHA512

# Verify
echo "expected_hash  file.txt" | sha256sum -c
sha256sum -c checksums.txt   # check from file

# Generate and verify
sha256sum important/*.txt > checksums.sha256
sha256sum -c checksums.sha256

# Compare files
cmp file1 file2              # binary compare
diff file1 file2             # text compare
md5sum file1 file2           # compare checksums

# File signature
gpg --sign file.txt          # sign
gpg --verify file.txt.sig file.txt  # verify

# Integrity monitoring
# Save state
find /etc -type f -exec sha256sum {} \; > /var/lib/checksums/etc.sha256

# Check later
sha256sum -c /var/lib/checksums/etc.sha256 2>&1 | grep FAILED
```

---

## 9.9 Symbolic Links & Hard Links

```bash
# Symbolic links (symlink/softlink)
ln -s /path/to/target link_name      # create symlink
ln -sf /new/target existing_link     # force update
ls -la                               # shows: link -> target
readlink link_name                   # show target
readlink -f link_name                # absolute path

# Hard links
ln source.txt hardlink.txt           # create hard link
ls -i source.txt hardlink.txt        # same inode number

# Check for links
find . -type l                       # find symlinks
find . -type l -! -e                 # broken symlinks
find . -maxdepth 1 -type l -exec ls -la {} \;

# Remove symlinks
rm link_name                         # remove symlink (not target)
unlink link_name                     # same

# Common use cases
ln -s /usr/bin/python3 /usr/local/bin/python    # python alias
ln -s /var/app/config/nginx.conf /etc/nginx/nginx.conf
ln -s /media/data /home/user/data               # mount point alias

# Resolve symlink chain
realpath link_name                   # final target
python3 -c "import os; print(os.path.realpath('link_name'))"
```

---

## 9.10 File Monitoring & Watching

```bash
# inotifywait (inotify-tools package)
inotifywait -m /path/to/watch        # monitor all events
inotifywait -m -e modify,create,delete /path
inotifywait -m -r /path              # recursive

# Watch specific events
inotifywait -m -e modify /etc/hosts  # file changed

# Build auto-reload on change
watch_and_reload() {
    local dir=$1
    local cmd=$2
    
    while true; do
        inotifywait -r -e modify,create,delete "$dir" 2>/dev/null
        echo "Change detected, running: $cmd"
        eval "$cmd"
    done
}

# Simple polling watcher (no inotify needed)
watch_file() {
    local file=$1
    local interval=${2:-2}
    local last_mtime=0
    
    while true; do
        if [[ -f "$file" ]]; then
            current_mtime=$(stat -c%Y "$file")
            if [[ $current_mtime -ne $last_mtime ]]; then
                echo "File changed: $file"
                last_mtime=$current_mtime
                # Do something
            fi
        fi
        sleep "$interval"
    done
}

# Monitor directory for new files
watch_directory() {
    local dir=$1
    local -A seen_files
    
    # Initialize with current files
    for f in "$dir"/*; do seen_files[$f]=1; done
    
    while true; do
        for f in "$dir"/*; do
            if [[ ! -v seen_files[$f] ]]; then
                echo "New file: $f"
                seen_files[$f]=1
            fi
        done
        sleep 2
    done
}
```

---

## 9.11 Temporary Files & Locking

```bash
# Create temp files safely
tmpfile=$(mktemp)            # create temp file
tmpdir=$(mktemp -d)         # create temp directory
tmpfile=$(mktemp /tmp/myapp.XXXXXX)  # custom pattern

# Auto-cleanup on exit
cleanup() {
    rm -f "$tmpfile"
    rm -rf "$tmpdir"
}
trap cleanup EXIT INT TERM

# File locking (flock)
# Exclusive lock on a file
(
    flock -x 200          # acquire exclusive lock on fd 200
    echo "Critical section - only one process here"
    sleep 5
    echo "Releasing lock"
) 200>/var/lock/myapp.lock

# With timeout
(
    flock -x -w 10 200 || { echo "Cannot acquire lock"; exit 1; }
    echo "Got lock"
) 200>/var/lock/myapp.lock

# Script-level locking
LOCK_FILE="/var/run/myapp.pid"

acquire_lock() {
    exec 9>"$LOCK_FILE"
    flock -n 9 || { echo "Another instance running"; exit 1; }
    echo $$ > "$LOCK_FILE"
}

release_lock() {
    rm -f "$LOCK_FILE"
    exec 9>&-
}

trap release_lock EXIT
acquire_lock

echo "Script running with PID $$"
# ... do work ...

# Atomic file operations
atomic_write() {
    local target=$1
    local tmpfile
    tmpfile=$(mktemp "${target}.XXXXXX")
    
    # Write to temp file
    echo "content" > "$tmpfile"
    
    # Atomic rename (on same filesystem)
    mv -f "$tmpfile" "$target"
}
```

---

## 9.12 Complete File Manager Script

```bash
#!/usr/bin/env bash
# file_manager.sh - Interactive file manager

set -euo pipefail

# Colors
RED='\033[0;31m'; GREEN='\033[0;32m'
YELLOW='\033[1;33m'; BLUE='\033[0;34m'
CYAN='\033[0;36m'; NC='\033[0m'

CURRENT_DIR="$PWD"

# Display directory listing
show_listing() {
    echo -e "\n${BLUE}📁 Directory: ${CYAN}$CURRENT_DIR${NC}\n"
    
    local count=0
    printf "${YELLOW}%-5s %-40s %-10s %-20s${NC}\n" "Perms" "Name" "Size" "Modified"
    printf "%s\n" "$(printf '─%.0s' {1..75})"
    
    for item in "$CURRENT_DIR"/{.,..} "$CURRENT_DIR"/*; do
        [[ -e "$item" || -L "$item" ]] || continue
        
        local name="${item##*/}"
        local perms size mtime
        
        perms=$(stat -c "%A" "$item" 2>/dev/null || echo "?????????")
        size=$(stat -c "%s" "$item" 2>/dev/null || echo 0)
        mtime=$(stat -c "%y" "$item" 2>/dev/null | cut -d. -f1)
        
        # Human size
        if (( size < 1024 )); then
            size="${size}B"
        elif (( size < 1048576 )); then
            size="$(( size / 1024 ))K"
        else
            size="$(( size / 1048576 ))M"
        fi
        
        local color=""
        if [[ -d "$item" ]]; then color="$BLUE📁 "
        elif [[ -x "$item" ]]; then color="$GREEN⚙  "
        elif [[ -l "$item" ]]; then color="$CYAN🔗 "
        else color="  "; fi
        
        printf "%-5s ${color}%-38s${NC} %-10s %s\n" \
            "$perms" "$name" "$size" "$mtime"
        ((count++))
    done
    
    echo -e "\n${count} items"
}

# Main menu
while true; do
    show_listing
    
    echo -e "\n${GREEN}Commands:${NC}"
    echo "  cd <dir>    - Change directory"
    echo "  cat <file>  - View file"
    echo "  cp <src> <dst> - Copy"
    echo "  mv <src> <dst> - Move/Rename"
    echo "  rm <file>   - Delete (with confirm)"
    echo "  mkdir <dir> - Create directory"
    echo "  find <name> - Search files"
    echo "  q           - Quit"
    
    read -rp $'\n> ' cmd args_str
    read -ra args <<< "$args_str"
    
    case $cmd in
        cd)
            target="${args[0]:-$HOME}"
            if cd "$CURRENT_DIR/$target" 2>/dev/null || cd "$target" 2>/dev/null; then
                CURRENT_DIR="$PWD"
            else
                echo -e "${RED}Cannot cd to: $target${NC}"
            fi
            ;;
        cat|less|view)
            target="${args[0]:-}"
            [[ -f "$CURRENT_DIR/$target" ]] && less "$CURRENT_DIR/$target" || echo "File not found"
            ;;
        rm|delete)
            target="${args[0]:-}"
            if [[ -e "$CURRENT_DIR/$target" ]]; then
                read -rp "Delete '$target'? [y/N] " confirm
                [[ "$confirm" =~ ^[yY]$ ]] && rm -rf "$CURRENT_DIR/$target" && echo "Deleted"
            else
                echo "Not found: $target"
            fi
            ;;
        mkdir)
            mkdir -p "$CURRENT_DIR/${args[0]}" && echo "Created"
            ;;
        find)
            pattern="${args[0]:-*}"
            find "$CURRENT_DIR" -name "$pattern" 2>/dev/null
            ;;
        q|quit|exit)
            echo "Bye!"
            exit 0
            ;;
        "")
            ;;
        *)
            echo -e "${YELLOW}Unknown command: $cmd${NC}"
            ;;
    esac
done
```

---

## 9.13 Exercises

### Exercise 1: Bulk File Renamer
สร้าง script ที่:
- รับ pattern เก่าและใหม่
- Rename ไฟล์ทั้งหมดใน directory
- Preview ก่อน confirm
- Log การ rename

### Exercise 2: File Organizer
จัดระเบียบไฟล์ตาม extension:
- Images → images/
- Documents → docs/
- Videos → videos/
- Archives → archives/

### Exercise 3: Duplicate Finder
ค้นหาไฟล์ซ้ำโดยเปรียบเทียบ:
- ขนาด
- Checksum (MD5/SHA256)
- แสดงกลุ่มไฟล์ซ้ำ

### Exercise 4: Backup System
สร้าง incremental backup:
- Backup เฉพาะไฟล์ที่เปลี่ยนแปลง
- Compress และ timestamp
- Verify checksum
- Rotation policy

---

## สรุป Part 09

✅ File creation, copy, move, delete  
✅ Directory operations  
✅ Permissions (chmod, chown, ACL, chattr)  
✅ File content viewing  
✅ Search (find, locate, grep)  
✅ Encoding & compression  
✅ File integrity (checksums)  
✅ Symlinks & hard links  
✅ File monitoring (inotify, polling)  
✅ Temp files & locking  

---

**→ Part 10: Text Processing (grep, sed, awk)**
