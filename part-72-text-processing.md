# Part 72: Text Processing and Natural Language Tools
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 72.1 sed/awk Transformation Pipelines

```bash
#!/bin/bash
# text_transform.sh - Text transformation utilities

set -euo pipefail

# ─── sed Patterns ────────────────────────────────────────────────────────────
normalize_whitespace() {
    sed 's/[[:space:]]\+/ /g; s/^[[:space:]]//; s/[[:space:]]$//'
}

strip_comments() {
    local comment_char=${1:-#}
    sed "/^[[:space:]]*${comment_char}/d; s/[[:space:]]*${comment_char}[^\"]*$//"
}

strip_blank_lines() {
    sed '/^[[:space:]]*$/d'
}

replace_in_file() {
    local file=$1 search=$2 replace=$3
    sed -i "s|${search}|${replace}|g" "$file"
}

extract_between_tags() {
    local open_tag=$1 close_tag=$2
    sed -n "/<${open_tag}>/,/<\/${close_tag}>/p" | \
        sed "1d; \$d"
}

insert_after_match() {
    local pattern=$1 insert_line=$2
    sed "/^${pattern}/a ${insert_line}"
}

delete_lines_matching() {
    local pattern=$1
    sed "/^${pattern}/d"
}

# ─── awk Patterns ────────────────────────────────────────────────────────────
extract_fields() {
    local delimiter=${1:-,} fields=${2:-1}
    awk -F"$delimiter" -v cols="$fields" '
    BEGIN { n=split(cols,c,",") }
    { for(i=1;i<=n;i++) printf "%s%s", $c[i], (i<n?OFS:"\n") }'
}

format_columns() {
    local delimiter=${1:-\t}
    awk -F"$delimiter" '{
        for(i=1;i<=NF;i++) if(length($i)>w[i]) w[i]=length($i)
        rows[NR]=$0; nf[NR]=NF
    }
    END {
        for(r=1;r<=NR;r++) {
            n=split(rows[r],f,FS)
            for(i=1;i<=n;i++) printf "%-"w[i]"s  ",f[i]
            print ""
        }
    }'
}

sum_column() {
    local col=${1:-1} delimiter=${2:-,}
    awk -F"$delimiter" -v c="$col" '{sum+=$c} END{print sum}'
}

awk_pivot() {
    local file=$1 row_col=$2 col_col=$3 val_col=$4
    awk -F',' -v rc="$row_col" -v cc="$col_col" -v vc="$val_col" '
    NR==1 { next }
    {
        r=$rc; c=$cc; v=$vc
        gsub(/"/,"",r); gsub(/"/,"",c); gsub(/"/,"",v)
        rows[r][c]=v
        if(!(c in cols)) { cols_arr[++nc]=c; cols[c]=1 }
        if(!(r in seen)) { rows_arr[++nr]=r; seen[r]=1 }
    }
    END {
        printf "row"
        for(i=1;i<=nc;i++) printf ",%s",cols_arr[i]
        print ""
        for(i=1;i<=nr;i++) {
            printf "%s",rows_arr[i]
            for(j=1;j<=nc;j++) printf ",%s",(rows_arr[i] in rows && cols_arr[j] in rows[rows_arr[i]] ? rows[rows_arr[i]][cols_arr[j]] : "")
            print ""
        }
    }' "$file"
}
```

---

## 72.2 grep Pattern Library

```bash
#!/bin/bash
# grep_patterns.sh - Useful grep patterns

# ─── Log Parsing ───────────────────────────────────────────────────────────────
grep_errors() {
    grep -iE '\b(error|err|failed|failure|fatal|critical|exception)\b'
}

grep_warnings() {
    grep -iE '\b(warn|warning|caution|deprecated)\b'
}

grep_ip_addresses() {
    grep -oP '\b(?:[0-9]{1,3}\.){3}[0-9]{1,3}\b'
}

grep_emails() {
    grep -oP '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}'
}

grep_urls() {
    grep -oP 'https?://[^\s"'\''>]+'
}

grep_timestamps() {
    grep -oP '\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}'
}

grep_json_field() {
    local field=$1
    grep -oP "\"${field}\":\s*\"?[^,}\"]+\"?"
}

# ─── Context Extraction ─────────────────────────────────────────────────────────
grep_context_window() {
    local pattern=$1 before=${2:-2} after=${3:-2}
    grep -A "$after" -B "$before" "$pattern"
}

grep_between_patterns() {
    local start=$1 end=$2
    awk "/^${start}/{found=1} found{print} /^${end}/{found=0}"
}

grep_count_by_pattern() {
    local file=$1
    shift
    local patterns=("$@")

    for pattern in "${patterns[@]}"; do
        local count; count=$(grep -c "$pattern" "$file" 2>/dev/null || echo 0)
        printf "%-40s %d\n" "$pattern" "$count"
    done
}

# ─── Log Summarizer ──────────────────────────────────────────────────────────
log_summary() {
    local log_file=$1

    echo "=== Log Summary: $log_file ==="
    echo "Total lines:    $(wc -l < "$log_file")"
    echo "Errors:         $(grep -ciE '\berror\b' "$log_file" 2>/dev/null || echo 0)"
    echo "Warnings:       $(grep -ciE '\bwarn\b' "$log_file" 2>/dev/null || echo 0)"
    echo ""
    echo "Top 5 Error Messages:"
    grep -iE '\berror\b' "$log_file" 2>/dev/null | \
        sed 's/^[^ ]* [^ ]* //; s/[0-9]\+/N/g' | \
        sort | uniq -c | sort -rn | head -5 | \
        awk '{count=$1; $1=""; printf "  %5d %s\n",count,substr($0,2)}'

    echo ""
    echo "Error Timeline (by hour):"
    grep -iE '\berror\b' "$log_file" 2>/dev/null | \
        grep -oP '\d{4}-\d{2}-\d{2} \d{2}' | \
        sort | uniq -c | \
        awk '{printf "  %s:00  %d\n", $2, $1}'
}
```

---

## 72.3 String Templating Engine

```bash
#!/bin/bash
# template.sh - String templating with variables, conditionals, loops

template_render() {
    local template_file=$1
    shift
    local -A vars=()

    # Parse key=value pairs
    while [[ $# -gt 0 ]]; do
        local key="${1%%=*}" value="${1#*=}"
        vars["$key"]="$value"
        shift
    done

    local content
    IFS= read -r -d '' content < "$template_file" || true

    # Replace {{var}} with values
    for key in "${!vars[@]}"; do
        content="${content//\{\{${key}\}\}/${vars[$key]}}"
    done

    # Replace {{env.VAR}} with environment variables
    while [[ "$content" =~ \{\{env\.([A-Z_]+)\}\} ]]; do
        local env_name="${BASH_REMATCH[1]}"
        local env_val="${!env_name:-}"
        content="${content//\{\{env.${env_name}\}\}/${env_val}}"
    done

    echo "$content"
}

template_render_string() {
    local template=$1
    shift

    # Simple variable substitution: ${VAR} from environment
    local result; result=$(eval "echo \"$template\"" 2>/dev/null || echo "$template")
    echo "$result"
}

template_if() {
    local condition=$1 true_text=$2 false_text=${3:-}
    eval "$condition" 2>/dev/null && echo "$true_text" || echo "$false_text"
}

template_for_each() {
    local template=$1
    shift
    local items=("$@")

    for item in "${items[@]}"; do
        echo "${template//\{\{item\}\}/$item}"
    done
}

template_repeat() {
    local template=$1 n=$2
    local i
    for (( i=0; i<n; i++ )); do
        echo "${template//\{\{i\}\}/$i}"
    done
}

generate_from_template() {
    local template_file=$1 data_file=$2 output_file=$3

    local template
    IFS= read -r -d '' template < "$template_file" || true

    > "$output_file"

    while IFS=',' read -r -a fields; do
        local rendered="$template"
        local i
        for (( i=0; i<${#fields[@]}; i++ )); do
            rendered="${rendered//\{\{col$((i+1))\}\}/${fields[$i]}}"
        done
        echo "$rendered" >> "$output_file"
    done < "$data_file"

    echo "Generated: $output_file"
}
```

---

## 72.4 Word Frequency and Text Analysis

```bash
#!/bin/bash
# text_analysis.sh - NLP-style text analysis

word_frequency() {
    local file=${1:-/dev/stdin} top_n=${2:-20}

    tr -cs '[:alpha:]' '\n' < "$file" | \
        tr '[:upper:]' '[:lower:]' | \
        grep -v '^$' | \
        sort | uniq -c | sort -rn | \
        head -"$top_n" | \
        awk '{printf "%5d  %s\n", $1, $2}'
}

char_frequency() {
    local file=${1:-/dev/stdin}

    fold -w1 < "$file" | \
        grep -v '^$' | \
        sort | uniq -c | sort -rn
}

sentence_count() {
    grep -oP '[^.!?]+[.!?]+' | wc -l
}

word_count_stats() {
    local file=$1

    local total_words; total_words=$(wc -w < "$file")
    local total_lines; total_lines=$(wc -l < "$file")
    local total_chars; total_chars=$(wc -c < "$file")
    local unique_words; unique_words=$(tr -cs '[:alpha:]' '\n' < "$file" | tr '[:upper:]' '[:lower:]' | sort -u | wc -l)
    local avg_word_len; avg_word_len=$(tr -cs '[:alpha:]' '\n' < "$file" | awk '{sum+=length($0)} END{printf "%d", sum/NR}' 2>/dev/null || echo 0)

    echo "Words:        $total_words"
    echo "Lines:        $total_lines"
    echo "Characters:   $total_chars"
    echo "Unique words: $unique_words"
    echo "Avg word len: $avg_word_len"
    echo "Lexical diversity: $(awk "BEGIN{printf \"%.2f\", $unique_words / ($total_words > 0 ? $total_words : 1)}")"
}

find_keywords() {
    local file=$1
    shift
    local keywords=("$@")

    for kw in "${keywords[@]}"; do
        local count; count=$(grep -oci "\b${kw}\b" "$file" 2>/dev/null || echo 0)
        local lines; lines=$(grep -in "\b${kw}\b" "$file" 2>/dev/null | cut -d: -f1 | tr '\n' ',' | sed 's/,$//')
        printf "%-20s count=%d  lines=%s\n" "$kw" "$count" "$lines"
    done
}

concordance() {
    local file=$1 word=$2 context=${3:-5}

    grep -oin "\b${word}\b" "$file" | while IFS=: read -r lineno _; do
        local start=$(( lineno - context ))
        (( start < 1 )) && start=1
        local end=$(( lineno + context ))

        awk -v s="$start" -v e="$end" -v w="$word" -v l="$lineno" '
        NR>=s && NR<=e {
            marker = (NR==l) ? ">>" : "  "
            printf "%s %4d: %s\n", marker, NR, $0
        }' "$file"
        echo "---"
    done
}

tf_idf_approx() {
    local dir=$1 word=$2

    local file_count; file_count=$(ls "$dir"/*.txt 2>/dev/null | wc -l)
    local docs_with_word=0

    echo "=== TF-IDF Approximation for: $word ==="
    echo ""

    for f in "$dir"/*.txt 2>/dev/null; do
        [[ -f "$f" ]] || continue
        local total_words; total_words=$(wc -w < "$f")
        local term_count; term_count=$(grep -oci "\b${word}\b" "$f" 2>/dev/null || echo 0)
        (( term_count > 0 )) && (( docs_with_word++ ))
        local tf; tf=$(awk "BEGIN{printf \"%.4f\", ${term_count} / (${total_words} > 0 ? ${total_words} : 1)}")
        printf "%-30s TF=%-8s count=%d\n" "$(basename "$f")" "$tf" "$term_count"
    done

    echo ""
    local idf; idf=$(awk "BEGIN{if($docs_with_word>0) printf \"%.4f\", log($file_count/$docs_with_word)/log(10); else print \"0\"}")
    echo "IDF: $idf  (docs=$file_count, docs_with_word=$docs_with_word)"
}
```

---

## 72.5 Markdown and Format Converters

```bash
#!/bin/bash
# format_convert.sh - Document format converters

markdown_to_plain() {
    sed \
        's/^#\+ //g' \
        -e 's/\*\*\(.*\)\*\*/\1/g' \
        -e 's/__\(.*\)__/\1/g' \
        -e 's/\*\(.*\)\*/\1/g' \
        -e 's/_\(.*\)_/\1/g' \
        -e 's/`\([^`]*\)`/\1/g' \
        -e 's/^[[:space:]]*[*+-] /  - /g' \
        -e 's/\[\([^]]*\)\]([^)]*)/\1/g' \
        -e 's/!\[\([^]]*\)\]([^)]*)/[Image: \1]/g' \
        -e 's/^>/  |/g' \
        -e '/^```/d'
}

markdown_to_html() {
    awk '
    /^```/ {
        if (!in_code) { print "<pre><code>"; in_code=1 }
        else { print "</code></pre>"; in_code=0 }
        next
    }
    in_code { print; next }
    /^######/ { sub(/^######[[:space:]]*/, ""); print "<h6>"$0"</h6>"; next }
    /^#####/  { sub(/^#####[[:space:]]*/,  ""); print "<h5>"$0"</h5>"; next }
    /^####/   { sub(/^####[[:space:]]*/,   ""); print "<h4>"$0"</h4>"; next }
    /^###/    { sub(/^###[[:space:]]*/,    ""); print "<h3>"$0"</h3>"; next }
    /^##/     { sub(/^##[[:space:]]*/,     ""); print "<h2>"$0"</h2>"; next }
    /^#/      { sub(/^#[[:space:]]*/,      ""); print "<h1>"$0"</h1>"; next }
    /^---+$/  { print "<hr>"; next }
    /^[[:space:]]*[*+-] / {
        if (!in_list) { print "<ul>"; in_list=1 }
        sub(/^[[:space:]]*[*+-][[:space:]]*/,"")
        print "<li>"$0"</li>"
        next
    }
    in_list { print "</ul>"; in_list=0 }
    /^$/ { print "<br>"; next }
    { print "<p>"$0"</p>" }
    END { if (in_list) print "</ul>" }
    '
}

csv_to_markdown_table() {
    awk -F',' '
    NR==1 {
        n=NF
        header=$0
        gsub(","," | ",header)
        print "| "header" |"
        sep="|"
        for(i=1;i<=n;i++) sep=sep"---|";
        print sep
        next
    }
    {
        line=$0
        gsub(","," | ",line)
        print "| "line" |"
    }'
}

ini_to_json() {
    awk '
    /^\[/ {
        if (section != "") printf "},\n"
        section=substr($0,2,length($0)-2)
        printf "\""section"\": {\n"
        first=1
        next
    }
    /=/ && section != "" {
        key=substr($0,1,index($0,"=")-1)
        val=substr($0,index($0,"=")+1)
        gsub(/^[[:space:]]+|[[:space:]]+$/, "", key)
        gsub(/^[[:space:]]+|[[:space:]]+$/, "", val)
        if (!first) printf ",\n"
        printf "  \"%s\": \"%s\"", key, val
        first=0
    }
    END { if (section != "") printf "\n}\n" }
    ' | awk 'BEGIN{print "{"} {print} END{print "}"}'
}

env_to_json() {
    local env_file=$1

    echo '{'
    local first=true
    while IFS='=' read -r key value; do
        [[ "$key" =~ ^#|^$ ]] && continue
        $first || echo ','
        printf '  "%s": "%s"' "$key" "${value//"/\\"}"
        first=false
    done < "$env_file"
    echo ''
    echo '}'
}
```

---

## 72.6 diff and Patch Tools

```bash
#!/bin/bash
# diff_patch.sh - Text diff and patching

text_diff() {
    local file1=$1 file2=$2 format=${3:-unified}

    case "$format" in
        unified) diff -u "$file1" "$file2" ;;
        context) diff -c "$file1" "$file2" ;;
        side)    diff -y --width=160 "$file1" "$file2" ;;
        brief)   diff -q "$file1" "$file2" ;;
    esac
}

generate_patch() {
    local orig=$1 modified=$2 patch_file=${3:-changes.patch}

    diff -u "$orig" "$modified" > "$patch_file" || true
    local added; added=$(grep -c '^+' "$patch_file" 2>/dev/null || echo 0)
    local removed; removed=$(grep -c '^-' "$patch_file" 2>/dev/null || echo 0)
    echo "Patch: $patch_file  (+$added -$removed lines)"
}

apply_patch() {
    local patch_file=$1 target_dir=${2:-.}
    local dry_run=${3:-false}

    local patch_args=(--strip=1 --directory="$target_dir")
    $dry_run && patch_args+=(--dry-run)

    patch "${patch_args[@]}" < "$patch_file"
}

diff_dirs() {
    local dir1=$1 dir2=$2

    diff -rq \
        --exclude='*.pyc' \
        --exclude='*.log' \
        --exclude='.git' \
        "$dir1" "$dir2"
}

interactive_merge() {
    local base=$1 ours=$2 theirs=$3 output=${4:-merged.txt}

    if command -v diff3 &>/dev/null; then
        diff3 -m "$ours" "$base" "$theirs" > "$output" 2>/dev/null
        local conflicts; conflicts=$(grep -c '^<<<<<<' "$output" 2>/dev/null || echo 0)
        echo "Merged: $output  (conflicts: $conflicts)"
    else
        echo "diff3 not available"
        return 1
    fi
}
```

---

## 72.7 Log Analyzer

```bash
#!/bin/bash
# log_analyzer.sh - Advanced log analysis

LOG_DB="${LOG_DB:-/tmp/log_analysis.db}"

log_analyze_init() {
    sqlite3 "$LOG_DB" "
        CREATE TABLE IF NOT EXISTS log_entries (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            ts TEXT,
            level TEXT,
            source TEXT,
            message TEXT,
            raw TEXT
        );
        CREATE INDEX IF NOT EXISTS idx_ts ON log_entries(ts);
        CREATE INDEX IF NOT EXISTS idx_level ON log_entries(level);
    " 2>/dev/null
}

log_analyze_import() {
    local log_file=$1

    log_analyze_init

    local count=0
    while IFS= read -r line; do
        local ts level source message

        # Parse common log formats
        if [[ "$line" =~ ^([0-9]{4}-[0-9]{2}-[0-9]{2}[T ][0-9]{2}:[0-9]{2}:[0-9]{2})[^[:alpha:]]*(ERROR|WARN|INFO|DEBUG)[[:space:]]*([^[:space:]]*)[[:space:]]*(.*) ]]; then
            ts="${BASH_REMATCH[1]}"
            level="${BASH_REMATCH[2]}"
            source="${BASH_REMATCH[3]}"
            message="${BASH_REMATCH[4]}"
        else
            ts="" level="UNKNOWN" source="" message="$line"
        fi

        # Escape for SQLite
        message="${message//\'/\'\'}" 
        local raw="${line//\'/\'\'}" 

        sqlite3 "$LOG_DB" \
            "INSERT INTO log_entries (ts,level,source,message,raw) VALUES ('$ts','$level','$source','$message','$raw')" 2>/dev/null
        (( count++ ))
    done < "$log_file"

    echo "Imported $count log entries"
}

log_error_clusters() {
    local top_n=${1:-10}

    sqlite3 -column -header "$LOG_DB" "
        SELECT
            SUBSTR(message,1,80) as pattern,
            COUNT(*) as count
        FROM log_entries
        WHERE level IN ('ERROR','FATAL','CRITICAL')
        GROUP BY SUBSTR(message,1,60)
        ORDER BY count DESC
        LIMIT $top_n;
    " 2>/dev/null
}

log_timeline() {
    local from_ts=${1:-} to_ts=${2:-} bucket=${3:-hour}

    local trunc_expr
    case "$bucket" in
        minute) trunc_expr="SUBSTR(ts,1,16)" ;;
        hour)   trunc_expr="SUBSTR(ts,1,13)" ;;
        day)    trunc_expr="SUBSTR(ts,1,10)" ;;
        *)      trunc_expr="SUBSTR(ts,1,13)" ;;
    esac

    local where="WHERE ts != ''"
    [[ -n "$from_ts" ]] && where+="AND ts >= '$from_ts'"
    [[ -n "$to_ts" ]]   && where+="AND ts <= '$to_ts'"

    sqlite3 "$LOG_DB" "
        SELECT ${trunc_expr} as bucket, level, COUNT(*) as count
        FROM log_entries
        $where
        GROUP BY bucket, level
        ORDER BY bucket, level;
    " 2>/dev/null
}

log_anomaly_detect() {
    local window_minutes=${1:-60} threshold_multiplier=${2:-3}

    sqlite3 "$LOG_DB" "
        WITH hourly AS (
            SELECT SUBSTR(ts,1,13) as hour, COUNT(*) as cnt
            FROM log_entries
            WHERE level='ERROR' AND ts != ''
            GROUP BY hour
        ),
        stats AS (
            SELECT AVG(cnt) as avg_cnt, AVG(cnt*cnt) - AVG(cnt)*AVG(cnt) as var_cnt FROM hourly
        )
        SELECT h.hour, h.cnt,
               ROUND(s.avg_cnt,1) as avg,
               ROUND(h.cnt - s.avg_cnt,1) as deviation
        FROM hourly h, stats s
        WHERE h.cnt > s.avg_cnt + $threshold_multiplier * MAX(1, SQRT(ABS(s.var_cnt)))
        ORDER BY deviation DESC;
    " 2>/dev/null
}
```

---

## 72.8 Text Encoding Utilities

```bash
#!/bin/bash
# encoding.sh - Text encoding detection and conversion

detect_encoding() {
    local file=$1

    if command -v file &>/dev/null; then
        file -i "$file" | grep -oP 'charset=\K\S+'
    elif command -v chardet &>/dev/null; then
        chardet "$file" | grep -oP ":\s+\K\S+"
    else
        echo "unknown"
    fi
}

convert_encoding() {
    local input_file=$1 from_enc=$2 to_enc=$3
    local output_file="${4:-${input_file%.txt}_${to_enc}.txt}"

    if command -v iconv &>/dev/null; then
        iconv -f "$from_enc" -t "$to_enc" "$input_file" > "$output_file"
        echo "Converted: $input_file ($from_enc) -> $output_file ($to_enc)"
    else
        echo "iconv not available"
        return 1
    fi
}

remove_bom() {
    local file=$1
    sed -i '1s/^\xef\xbb\xbf//' "$file"
}

normalize_newlines() {
    local file=$1 target=${2:-unix}

    case "$target" in
        unix)    sed -i 's/\r//' "$file" ;;
        windows) sed -i 's/$/\r/' "$file" ;;
        mac)     tr '\n' '\r' < "$file" | sponge "$file" 2>/dev/null || \
                 { tmp=$(mktemp); tr '\n' '\r' < "$file" > "$tmp"; mv "$tmp" "$file"; } ;;
    esac
}

url_encode() {
    local string=$1
    python3 -c "import urllib.parse; print(urllib.parse.quote('$string'))" 2>/dev/null || \
    printf '%s' "$string" | xxd -plain | sed 's/\(..\)/%\1/g'
}

url_decode() {
    local string=$1
    python3 -c "import urllib.parse; print(urllib.parse.unquote('$string'))" 2>/dev/null || \
    echo -e "${string//%/\\x}"
}

base64_encode() { base64 -w0 <<< "$1"; }
base64_decode() { base64 -d <<< "$1"; }

hex_encode() { xxd -p <<< "$1" | tr -d '\n'; }
hex_decode() { xxd -r -p <<< "$1"; }
```

---

## 72.9 Column and Pretty-Print Utilities

```bash
#!/bin/bash
# pretty_print.sh - Output formatting utilities

table_print() {
    local -a headers=("$@")

    column -t -s $'\t'
}

align_columns() {
    local delimiter=${1:-,}
    column -t -s "$delimiter"
}

wrap_text() {
    local width=${1:-80}
    fold -s -w "$width"
}

indent_text() {
    local indent=${1:-4}
    local spaces; spaces=$(printf '%*s' "$indent" '')
    sed "s/^/${spaces}/"
}

truncate_lines() {
    local max_len=${1:-100} suffix=${2:-...}
    local suffix_len=${#suffix}
    local trim_len=$(( max_len - suffix_len ))

    awk -v max="$max_len" -v trim="$trim_len" -v suf="$suffix" '
    {
        if (length($0) > max) print substr($0,1,trim) suf
        else print
    }'
}

color_output() {
    local color=${1:-reset}
    local text=${2:-}

    local -A colors=(
        [red]='\033[0;31m'
        [green]='\033[0;32m'
        [yellow]='\033[0;33m'
        [blue]='\033[0;34m'
        [magenta]='\033[0;35m'
        [cyan]='\033[0;36m'
        [white]='\033[0;37m'
        [bold]='\033[1m'
        [reset]='\033[0m'
    )

    local code="${colors[$color]:-${colors[reset]}}"
    printf '%b%s%b\n' "$code" "$text" "${colors[reset]}"
}

progress_bar() {
    local current=$1 total=$2 width=${3:-40} label=${4:-Progress}
    local percent=$(( current * 100 / (total > 0 ? total : 1) ))
    local filled=$(( current * width / (total > 0 ? total : 1) ))
    local empty=$(( width - filled ))

    local bar; bar=$(printf '%0.s#' $(seq 1 $filled 2>/dev/null))
    local space; space=$(printf '%0.s.' $(seq 1 $empty 2>/dev/null))

    printf '\r%s [%s%s] %d%%' "$label" "$bar" "$space" "$percent"
    (( current >= total )) && echo ''
}

spinner() {
    local pid=$1 label=${2:-Working}
    local -a frames=('/' '-' '\\' '|')
    local i=0

    while kill -0 "$pid" 2>/dev/null; do
        printf '\r%s %s' "${frames[$i]}" "$label"
        i=$(( (i+1) % 4 ))
        sleep 0.1
    done
    printf '\r%-*s\r' $(( ${#label} + 2 )) ''
}

print_box() {
    local title=$1
    shift
    local lines=("$@")

    local max_len=${#title}
    for line in "${lines[@]}"; do
        (( ${#line} > max_len )) && max_len=${#line}
    done

    local border; border=$(printf '%0.s-' $(seq 1 $(( max_len + 4 )) 2>/dev/null))
    echo "+${border}+"
    printf '|  %-*s  |\n' "$max_len" "$title"
    echo "+${border}+"
    for line in "${lines[@]}"; do
        printf '|  %-*s  |\n' "$max_len" "$line"
    done
    echo "+${border}+"
}
```

---

## 72.10 Exercises

### Exercise 1: Log Intelligence Platform
สร้าง platform ที่:
- Multi-format log parser (nginx, apache, syslog, JSON)
- Error clustering with edit distance
- Anomaly scoring
- Email digest generation

### Exercise 2: Document Processor
สร้าง processor ที่:
- Batch markdown → HTML conversion
- Table of contents generation
- Cross-reference linking
- Word count and reading time

### Exercise 3: Text Diff Tool
สร้าง diff tool ที่:
- Side-by-side colored diff
- Semantic diff (ignore whitespace/comments)
- Directory diff summary
- Auto-apply patch with conflict detection

---

## สรุป Part 72

✅ sed: normalize whitespace, strip comments, insert/delete lines, in-place replace
✅ awk: field extraction, column formatting, sum, pivot table
✅ grep: errors/warnings, IPs/emails/URLs, timestamps, JSON fields, context windows
✅ log_summary(): error count, top-5 message clustering, hourly error timeline
✅ String templating: {{var}} substitution, {{env.VAR}}, for-each, repeat
✅ Word frequency, lexical diversity, concordance, TF-IDF approximation
✅ markdown_to_plain/html, csv_to_markdown_table, ini_to_json, env_to_json
✅ diff/patch: generate patch, apply, dir diff, three-way merge
✅ SQLite-backed log analyzer: import, error clusters, timeline, anomaly detection
✅ Encoding: detect, iconv convert, BOM removal, normalize newlines, base64/hex
┅ Pretty-print: column align, word wrap, truncate, progress bar, spinner, box

---

**→ Part 73: API Integration and HTTP Client Scripting**
