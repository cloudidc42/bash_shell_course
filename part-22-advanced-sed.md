# Part 22: Advanced sed & Stream Processing
## หลักสูตร Bash/Shell Script ระดับ Intermediate

---

## 22.1 sed Architecture

```bash
# sed = Stream EDitor
# Read line → Apply scripts → Print
# Scripts: [address][command]

# Address types:
# N        - line number
# $        - last line
# /regex/  - matching regex
# /r1/,/r2/ - range between two regex
# N,M      - line range
# ~step    - every Nth line

# Commands:
# p  - print
# d  - delete
# s  - substitute
# a  - append after
# i  - insert before
# c  - change line
# y  - transliterate
# q  - quit
# r  - read file
# w  - write to file
# =  - print line number
# n  - next line
# N  - append next to pattern space
# D  - delete first line of multi-line
# P  - print first line of multi-line
# H  - append to hold space
# h  - copy to hold space
# G  - append from hold space
# g  - copy from hold space
# x  - exchange pattern/hold space
# l  - print unambiguously (show escapes)
# b  - branch (goto label)
# t  - branch if substitution succeeded
# T  - branch if substitution failed
# :  - define label
```

---

## 22.2 Substitution Mastery

```bash
# ─── Basic substitution ───────────────────────────────────────
sed 's/old/new/'            # first occurrence
sed 's/old/new/g'           # global (all occurrences)
sed 's/old/new/2'           # second occurrence only
sed 's/old/new/gi'          # global, case-insensitive
sed 's/old/new/p'           # print if substitution made
sed 's/old/new/w out.txt'   # write matches to file

# Case-insensitive (GNU sed)
sed 's/error/ERROR/I'
sed 's/[Ee][Rr][Rr][Oo][Rr]/ERROR/'   # portable

# ─── Backreferences ───────────────────────────────────────────
# Capture groups with \( \) and reference with \1, \2
sed 's/\(foo\)bar/\1baz/'   # foobar → foobaz
sed 's/\([0-9]\+\) \([a-z]\+\)/\2 \1/'  # swap "123 abc" → "abc 123"

# ERE mode (-E or -r): use () without backslash
sed -E 's/(foo)bar/\1baz/'
sed -E 's/([0-9]+) ([a-z]+)/\2 \1/'

# Reformat date: YYYY-MM-DD → DD/MM/YYYY
echo "2024-01-15" | sed -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\3\/\2\/\1/'

# Reformat IP: remove leading zeros
echo "192.168.001.010" | sed -E 's/0*([0-9]+)/\1/g'

# ─── Special replacements ─────────────────────────────────────
sed 's/old/new/e'           # execute replacement as shell command (GNU)
sed 's/.*/echo "&" | wc -c/e'   # replace each line with its length

# & = matched text
sed 's/[0-9]\+/[&]/'       # wrap numbers in brackets: 42 → [42]
sed 's/.*/"\0"/'            # quote entire line
sed 's/^/>> /'             # prefix each line

# ─── Multi-line substitution ──────────────────────────────────
# Join continuation lines (lines ending with \)
sed -e ':a' -e '/\\$/{N;s/\\\n//;ba}'  file

# Replace newline within pattern
echo -e "hello\nworld" | sed -N '/hello/{N;s/\n/ /p}'
```

---

## 22.3 Addresses & Ranges

```bash
# ─── Line addresses ───────────────────────────────────────────
sed '5d'                    # delete line 5
sed '1,5d'                  # delete lines 1-5
sed '5,$d'                  # delete line 5 to end
sed '1~2d'                  # delete every other line (odd)
sed '0~2d'                  # delete every other line (even)
sed -n '5,10p'             # print only lines 5-10

# ─── Regex addresses ──────────────────────────────────────────
sed '/pattern/d'            # delete matching lines
sed '/pattern/!d'           # delete NON-matching lines
sed -n '/pattern/p'         # print only matching lines (= grep)
sed '/^#/d'                 # remove comment lines
sed '/^$/d'                 # remove empty lines
sed '/^[[:space:]]*$/d'     # remove blank/whitespace-only lines

# ─── Range addresses ──────────────────────────────────────────
sed '/START/,/END/d'        # delete between START and END
sed '/START/,/END/s/old/new/g'  # substitute in range
sed -n '/START/,/END/p'    # print between START and END

# Exclusive range (don't include the markers)
sed '/START/,/END/{/START/d;/END/d;p}' -n

# ─── Negation ─────────────────────────────────────────────────
sed '/pattern/!{command}'   # command on lines NOT matching
sed '1,5!d'                 # keep only lines 1-5 (delete rest)
```

---

## 22.4 Hold Space & Multi-line

```bash
# Pattern space = current processing buffer
# Hold space = secondary buffer (persistent across lines)

# ─── Reverse file (tac equivalent) ───────────────────────────
sed -n '1!G;h;$p' file

# Explanation:
# 1!G  - if not first line, append hold space to pattern space
# h    - copy pattern space to hold space
# $p   - if last line, print pattern space

# ─── Remove duplicate adjacent lines (uniq equivalent) ────────
sed '$!N;/^\(.*\)\n\1$/!P;D'

# ─── Join every two lines ─────────────────────────────────────
sed 'N;s/\n/ /'

# ─── Print paragraph (double-spaced blocks) ───────────────────
sed -n '/./,/^$/p'

# ─── Reverse words on a line ──────────────────────────────────
echo "one two three" | sed -E ':a;s/(\S+)(.*) (\S+)/\3\2 \1/;ta'

# ─── Number lines (like cat -n) ───────────────────────────────
sed '=' file | sed 'N;s/\n/\t/'

# ─── Print context around pattern ─────────────────────────────
# Print 2 lines before and after pattern
sed -n '/ERROR/{
    x;p;x
    N;N
    p
}' file

# ─── Insert text at specific position ─────────────────────────
# Insert after line 5
sed '5a\New line after 5' file

# Insert before line 5
sed '5i\New line before 5' file

# Append after matching line
sed '/pattern/a\Line after matching' file

# ─── Transform multi-line blocks ──────────────────────────────
# Remove blank lines at start/end of file
sed '/./,/^$/!d' file   # remove leading blank lines
sed -n '/^$/!{h;p;d};x;/./p' file  # remove trailing blank lines
```

---

## 22.5 Practical sed Scripts

```bash
# ─── File editing (in-place) ──────────────────────────────────
sed -i 's/old/new/g' file              # in-place edit
sed -i.bak 's/old/new/g' file         # with backup
sed -i '' 's/old/new/g' file          # macOS (no backup)

# Multiple expressions
sed -i -e 's/foo/bar/' -e 's/baz/qux/' file

# Edit specific line in config
sed -i 's/^PORT=.*/PORT=8080/' /etc/myapp.conf
sed -i "s|^ServerName .*|ServerName $HOSTNAME|" /etc/apache2/sites.conf

# ─── Config file manipulation ─────────────────────────────────
# Set a value (add if missing)
set_config() {
    local file=$1
    local key=$2
    local value=$3
    
    if grep -q "^${key}=" "$file"; then
        sed -i "s|^${key}=.*|${key}=${value}|" "$file"
    else
        echo "${key}=${value}" >> "$file"
    fi
}

# Comment/uncomment lines
comment_line()   { sed -i "s|^$1|#$1|" "$2"; }
uncomment_line() { sed -i "s|^#$1|$1|" "$2"; }

# ─── Text transformation ──────────────────────────────────────
# Convert Windows line endings to Unix
sed -i 's/\r//' file
dos2unix file               # or use this

# Remove trailing whitespace
sed -i 's/[[:space:]]*$//' file

# Remove HTML tags
sed 's/<[^>]*>//g'

# Extract text between tags
sed -n 's/.*<title>\(.*\)<\/title>.*/\1/p'

# Remove comments
sed '/^#/d; /^$/d'          # remove comment lines and blank lines
sed 's/[[:space:]]*#.*//'   # remove inline comments

# ─── Generate output ──────────────────────────────────────────
# Replace placeholders in template
render_template() {
    local template=$1
    local -n vars=$2   # nameref to associative array
    
    local result
    result=$(cat "$template")
    
    for key in "${!vars[@]}"; do
        result=$(echo "$result" | sed "s|{{${key}}}|${vars[$key]}|g")
    done
    
    echo "$result"
}

# Usage
declare -A data=(
    [NAME]="Alice"
    [DATE]="2024-01-15"
    [VERSION]="1.0.0"
)
render_template template.txt data > output.txt

# ─── Stream transformations ───────────────────────────────────
# Extract CSV column
awk -F, '{print $3}'  # better, but:
sed 's/[^,]*,\([^,]*\).*/\1/'  # second column (first match)

# Convert spaces to tabs
sed 's/  */\t/g'

# Number lines with specific format
sed = file | sed 'N;s/\n/. /'

# ─── Complex multi-command ────────────────────────────────────
# Process nginx config: remove comments, blank lines, standardize
sed -n \
    -e '/^[[:space:]]*#/d' \
    -e '/^[[:space:]]*$/d' \
    -e 's/[[:space:]]*#.*//' \
    -e 's/[[:space:]]\+/ /g' \
    -e '/./p' \
    nginx.conf
```

---

## 22.6 sed Script Files

```bash
# Can write sed commands to a file
cat > transform.sed << 'EOF'
# Remove comments
/^[[:space:]]*#/d
/^[[:space:]]*$/d

# Normalize spaces
s/[[:space:]]\+/ /g

# Uppercase section headers
s/^\[.*\]/\U&\E/

# Fix indentation (replace tabs with 4 spaces)
s/^\t/    /g
EOF

sed -f transform.sed input.conf

# ─── Sed + Shell Integration ──────────────────────────────────
# Use shell variables in sed
name="Alice"
sed "s/USER/${name}/" template.txt

# Safe substitution (avoid / in values)
dir="/usr/local/bin"
sed "s|INSTALL_DIR|${dir}|g" install.sh

# Multiple file processing
for f in *.conf; do
    sed -i \
        -e 's/localhost/db.prod.internal/g' \
        -e 's/debug: true/debug: false/' \
        "$f"
done
```

---

## 22.7 Exercises

### Exercise 1: Log Sanitizer
สร้าง script ที่:
- Remove personal information (emails, IPs, usernames)
- Standardize timestamps
- Remove debug messages
- Process multiple files

### Exercise 2: Config Manager
สร้าง wrapper ที่:
- Read/write config values
- Support nested sections
- Handle comments
- Diff two config files

### Exercise 3: Code Formatter
สร้าง simple code formatter:
- Fix indentation
- Standardize line endings
- Remove trailing whitespace
- Normalize quotes (single to double)

---

## สรุป Part 22

✅ sed architecture (pattern space, hold space)  
✅ Substitution with backreferences  
✅ Line and regex addressing  
✅ Range operations  
✅ Multi-line processing (N, D, P, H, G, h, g)  
✅ In-place file editing  
✅ Config file manipulation  
✅ Template rendering  
✅ sed script files  

---

**→ Part 23: JSON & Data Formats**
