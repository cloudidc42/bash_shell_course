# Part 51: Advanced Text Processing (awk, sed, perl)
## หลักสูตร Bash/Shell Script ระดับ World-Class

---

## 51.1 AWK Mastery

```bash
#!/bin/bash
# awk_mastery.sh - Advanced AWK patterns

# ─── Multi-file Report Generator ──────────────────────────────
generate_report() {
    awk '
    BEGIN {
        FS = ","
        OFS = "\t"
        total = 0
        count = 0
        printf "%-20s %10s %10s %10s\n", "CATEGORY", "COUNT", "SUM", "AVG"
        printf "%s\n", "--------------------------------------------------------"
    }

    NR == 1 { next }  # Skip header

    {
        category[$1]++
        sum[$1] += $3
        total += $3
        count++
    }

    END {
        for (cat in category) {
            avg = (category[cat] > 0) ? sum[cat] / category[cat] : 0
            printf "%-20s %10d %10.2f %10.2f\n", cat, category[cat], sum[cat], avg
        }
        printf "%s\n", "--------------------------------------------------------"
        printf "%-20s %10d %10.2f %10.2f\n", "TOTAL", count, total, total/count
    }
    ' "$@"
}

# ─── Log Analysis ──────────────────────────────────────────────
analyze_access_log() {
    local log_file=$1

    awk '
    {
        # Status codes
        status[$9]++

        # IP addresses
        ip[$1]++

        # Request paths
        match($7, /^\/[^?]*/, path_match)
        paths[path_match[0]]++

        # Response sizes
        if ($10 ~ /^[0-9]+$/) total_bytes += $10

        # Time bucketing (hourly)
        if (match($4, /\[([0-9]+\/[A-Za-z]+\/[0-9]+):([0-9]+)/, dt)) {
            hour = dt[1] ":" dt[2]
            hourly[hour]++
        }
    }

    END {
        print "=== Status Codes ==="
        for (s in status) printf "  %s: %d\n", s, status[s]

        print "\n=== Top 10 IPs ==="
        for (i in ip) print ip[i], i | "sort -rn | head -10"
        close("sort -rn | head -10")

        print "\n=== Top 10 Paths ==="
        for (p in paths) print paths[p], p | "sort -rn | head -10"
        close("sort -rn | head -10")

        print "\n=== Total Transfer ==="
        printf "  %.2f MB\n", total_bytes/1048576
    }
    ' "$log_file"
}

# ─── CSV Processing ────────────────────────────────────────────
csv_to_json() {
    awk '
    BEGIN { FS = ","; first = 1 }
    NR == 1 {
        # Store headers
        for (i = 1; i <= NF; i++) {
            gsub(/[^a-zA-Z0-9_]/, "_", $i)
            headers[i] = $i
        }
        print "["
        next
    }
    {
        if (!first) print ","
        first = 0
        print "  {"
        for (i = 1; i <= NF; i++) {
            gsub(/"/, "\\\"", $i)
            comma = (i < NF) ? "," : ""
            if ($i ~ /^[0-9]+(\.[0-9]+)?$/)
                printf "    \"%s\": %s%s\n", headers[i], $i, comma
            else
                printf "    \"%s\": \"%s\"%s\n", headers[i], $i, comma
        }
        print "  }"
    }
    END { print "]" }
    '
}

# ─── AWK Functions ─────────────────────────────────────────────
awk_statistics() {
    local file=$1
    local column=${2:-1}

    awk -v col="$column" '
    function max(a, b) { return a > b ? a : b }
    function min(a, b) { return a < b ? a : b }
    function abs(x) { return x < 0 ? -x : x }

    {
        val = $col + 0
        if (NR == 1) {
            mn = mx = val
        }
        sum += val
        sum2 += val * val
        mn = min(mn, val)
        mx = max(mx, val)
        values[NR] = val
        count++
    }

    END {
        if (count == 0) exit
        avg = sum / count
        variance = (sum2 / count) - (avg * avg)
        stddev = sqrt(variance < 0 ? -variance : variance)

        # Median
        n = asort(values)
        if (n % 2 == 0)
            median = (values[n/2] + values[n/2+1]) / 2
        else
            median = values[(n+1)/2]

        printf "Count:    %d\n", count
        printf "Sum:      %.4f\n", sum
        printf "Mean:     %.4f\n", avg
        printf "Median:   %.4f\n", median
        printf "StdDev:   %.4f\n", stddev
        printf "Min:      %.4f\n", mn
        printf "Max:      %.4f\n", mx
        printf "Range:    %.4f\n", mx - mn
    }
    ' "$file"
}

# ─── Multi-dimensional Arrays ──────────────────────────────────
cross_tabulate() {
    local file=$1
    local row_col=${2:-1}
    local col_col=${3:-2}
    local val_col=${4:-3}

    awk -v rc="$row_col" -v cc="$col_col" -v vc="$val_col" '
    NR == 1 { next }
    {
        rows[$rc] = 1
        cols[$cc] = 1
        matrix[$rc, $cc] += $vc
    }
    END {
        # Print header
        printf "%-15s", ""
        for (c in cols) printf "%-12s", c
        print ""

        # Print data rows
        for (r in rows) {
            printf "%-15s", r
            for (c in cols) {
                printf "%-12.2f", matrix[r, c]+0
            }
            print ""
        }
    }
    ' "$file"
}
```

---

## 51.2 SED Mastery

```bash
#!/bin/bash
# sed_mastery.sh - Advanced sed patterns

# ─── Multi-line Processing ─────────────────────────────────────
# Join continuation lines (lines ending with \)
join_continuations() {
    sed ':loop; /\\$/ { N; s/\\\n//; b loop }' "$1"
}

# Paragraph-based processing
process_paragraphs() {
    local file=$1
    sed -n '/^$/!{ H; $!d }; x; /./{ s/^\n//; p }' "$file"
}

# ─── In-place Transforms ───────────────────────────────────────
batch_replace() {
    local dir=$1
    local pattern=$2
    local replacement=$3
    local file_glob=${4:-"*.sh"}

    find "$dir" -name "$file_glob" -type f | while IFS= read -r file; do
        if grep -q "$pattern" "$file" 2>/dev/null; then
            sed -i "s/${pattern}/${replacement}/g" "$file"
            echo "Updated: $file"
        fi
    done
}

# ─── Address Ranges ────────────────────────────────────────────
extract_section() {
    local file=$1
    local start_pattern=$2
    local end_pattern=$3

    sed -n "/${start_pattern}/,/${end_pattern}/p" "$file"
}

delete_between() {
    local file=$1
    local start=$2
    local end=$3
    sed "/${start}/,/${end}/d" "$file"
}

# ─── Advanced Substitutions ────────────────────────────────────
normalize_whitespace() {
    sed 's/[[:space:]]\+/ /g; s/^[[:space:]]//; s/[[:space:]]$//'
}

strip_comments() {
    local comment_char=${1:-#}
    sed "/^[[:space:]]*${comment_char}/d; s/[[:space:]]*${comment_char}.*$//"
}

add_line_numbers() {
    sed = | sed 'N; s/\n/\t/'
}

wrap_lines() {
    local width=${1:-80}
    sed "s/.\{${width}\}/&\n/g"
}

# ─── Template Processing ───────────────────────────────────────
apply_sed_template() {
    local template_file=$1
    shift
    local replacements=("$@")

    local sed_script=""
    for repl in "${replacements[@]}"; do
        local key="${repl%%=*}"
        local value="${repl#*=}"
        value="${value//\//\\/}"
        sed_script+="s/{{${key}}}/${value}/g;"
    done

    sed "$sed_script" "$template_file"
}

# ─── XML/HTML Processing ───────────────────────────────────────
strip_xml_tags() {
    sed 's/<[^>]*>//g; /^[[:space:]]*$/d'
}

extract_xml_attribute() {
    local attr=$1
    sed -n "s/.*${attr}=\"\([^\"]*\)\".*/\1/p"
}

html_entities_decode() {
    sed 's/&amp;/\&/g;
         s/&lt;/</g;
         s/&gt;/>/g;
         s/&quot;/"/g;
         s/&#039;/'"'"'/g;
         s/&nbsp;/ /g'
}
```

---

## 51.3 Perl One-liners and Scripts

```bash
#!/bin/bash
# perl_text_processing.sh - Perl-powered text processing

# ─── Text Transformation ──────────────────────────────────────
perl_to_camelcase() {
    perl -pe 's/_([a-z])/uc($1)/ge'
}

perl_to_snakecase() {
    perl -pe '
        s/([A-Z])/"_".lc($1)/ge;
        s/^_//;
        s/-/_/g;
        tr/A-Z/a-z/
    '
}

perl_word_count() {
    perl -ne '
        $total_lines++;
        $total_words += scalar(split /\s+/, $_);
        $total_chars += length($_);
        END {
            printf "Lines: %d\nWords: %d\nChars: %d\n",
                $total_lines, $total_words, $total_chars;
        }
    ' "$@"
}

# ─── Data Extraction ───────────────────────────────────────────
extract_emails() {
    perl -nE '
        while (/[\w.+-]+@[\w-]+\.[\w.]+/g) {
            say $&;
        }
    ' "$@" | sort -u
}

extract_urls() {
    perl -nE '
        while (m{https?://[^\s<>"{}|\\^`\[\]]+}g) {
            say $&;
        }
    ' "$@" | sort -u
}

extract_ips() {
    perl -nE '
        while (/\b(\d{1,3}\.){3}\d{1,3}\b/g) {
            say $&;
        }
    ' "$@" | sort -u
}

# ─── JSON Processing ───────────────────────────────────────────
perl_json_transform() {
    local filter=$1

    perl -MJSON -ne '
        my $data = decode_json($_);
        '"$filter"'
        print encode_json($data) . "\n";
    '
}

# ─── Multi-line Patterns ───────────────────────────────────────
perl_multiline_replace() {
    local file=$1
    local pattern=$2
    local replacement=$3

    perl -0777 -pe "s/${pattern}/${replacement}/gs" "$file"
}

perl_extract_blocks() {
    local start_pattern=$1
    local end_pattern=$2

    perl -0777 -ne "
        while (/${start_pattern}(.*?)${end_pattern}/gs) {
            print \$1, \"\\n---\\n\";
        }
    "
}

# ─── CSV/TSV Processing ────────────────────────────────────────
perl_csv_transform() {
    perl -MText::CSV -ne '
        BEGIN {
            $csv = Text::CSV->new({binary => 1, auto_diag => 1});
        }
        if ($csv->parse($_)) {
            my @fields = $csv->fields();
            # Transform: uppercase first field
            $fields[0] = uc($fields[0]);
            print join(",", @fields) . "\n";
        }
    ' "$@"
}

# ─── String Generation ─────────────────────────────────────────
perl_random_string() {
    local length=${1:-16}
    local charset=${2:-'a-zA-Z0-9'}
    perl -e "
        my \$chars = join('', map { chr(\$_) } grep {/[${charset}]/} 32..126);
        print join('', map { substr(\$chars, int(rand(length(\$chars))), 1) } 1..${length}), \"\\n\";
    "
}

perl_uuid() {
    perl -e '
        my @hex = map { sprintf("%02x", int(rand(256))) } 1..16;
        $hex[6] = sprintf("%02x", (hex($hex[6]) & 0x0f) | 0x40);
        $hex[8] = sprintf("%02x", (hex($hex[8]) & 0x3f) | 0x80);
        printf "%s%s%s%s-%s%s-%s%s-%s%s-%s%s%s%s%s%s\n", @hex;
    '
}
```

---

## 51.4 jq Mastery

```bash
#!/bin/bash
# jq_mastery.sh - Advanced jq patterns

# ─── Data Transformation ──────────────────────────────────────
json_flatten() {
    jq '[
        paths(scalars) as $path |
        {
            key: ($path | join(".")),
            value: getpath($path)
        }
    ] | from_entries'
}

json_unflatten() {
    jq 'to_entries |
        reduce .[] as $item (
            {};
            setpath(($item.key | split(".")); $item.value)
        )'
}

json_group_by_field() {
    local field=$1
    jq --arg field "$field" '
        group_by(.[$field]) |
        map({
            key: .[0][$field],
            value: .
        }) |
        from_entries
    '
}

json_pivot() {
    local key_field=$1
    local value_field=$2

    jq --arg k "$key_field" --arg v "$value_field" '
        reduce .[] as $item (
            {};
            . + {($item[$k]): $item[$v]}
        )
    '
}

# ─── Array Operations ──────────────────────────────────────────
json_unique_by() {
    local field=$1
    jq --arg f "$field" 'unique_by(.[$f])'
}

json_sort_by() {
    local field=$1
    local reverse=${2:-false}
    if $reverse; then
        jq --arg f "$field" 'sort_by(.[$f]) | reverse'
    else
        jq --arg f "$field" 'sort_by(.[$f])'
    fi
}

json_filter_where() {
    local field=$1
    local operator=$2
    local value=$3

    case "$operator" in
        "==") jq --arg f "$field" --arg v "$value" '[.[] | select(.[$f] == $v)]' ;;
        "!=") jq --arg f "$field" --arg v "$value" '[.[] | select(.[$f] != $v)]' ;;
        ">")  jq --arg f "$field" --argjson v "$value" '[.[] | select(.[$f] > $v)]' ;;
        "<")  jq --arg f "$field" --argjson v "$value" '[.[] | select(.[$f] < $v)]' ;;
        "contains") jq --arg f "$field" --arg v "$value" '[.[] | select(.[$f] | contains($v))]' ;;
    esac
}

# ─── Aggregation ───────────────────────────────────────────────
json_aggregate() {
    local group_field=$1
    local sum_field=$2

    jq --arg g "$group_field" --arg s "$sum_field" '
        group_by(.[$g]) |
        map({
            group: .[0][$g],
            count: length,
            sum: (map(.[$s]) | add),
            avg: (map(.[$s]) | add) / length,
            min: (map(.[$s]) | min),
            max: (map(.[$s]) | max)
        })
    '
}

# ─── Schema Generation ─────────────────────────────────────────
json_schema() {
    jq '
        def type_of:
            type as $t |
            if $t == "array" then "array<" + (.[0] | type_of) + ">"
            elif $t == "object" then "object"
            else $t
            end;
        def schema:
            if type == "object" then
                with_entries(.value = (.value | type_of))
            elif type == "array" then
                .[0] | schema
            else type
            end;
        schema
    '
}
```

---

## 51.5 Exercises

### Exercise 1: Log Analyzer Pipeline
สร้าง pipeline ที่:
- Parse Nginx logs ด้วย AWK
- Transform ด้วย sed
- Aggregate ด้วย Perl
- Output JSON ด้วย jq
- Dashboard แบบ real-time

### Exercise 2: Data Migration Toolkit
สร้าง toolkit ที่:
- CSV → JSON, JSON → CSV
- XML → JSON
- TSV → formatted report
- Handle encoding (UTF-8)

### Exercise 3: Code Formatter
สร้าง formatter สำหรับ shell scripts ที่:
- Normalize indentation
- Sort imports/sources
- Remove trailing whitespace
- Standardize quoting style

---

## สรุป Part 51

✅ AWK multi-file reporting and statistics
✅ AWK cross-tabulation with multi-dimensional arrays
✅ AWK CSV to JSON conversion
✅ SED multi-line processing and paragraph handling
✅ SED template variable substitution
✅ SED XML/HTML tag stripping
✅ Perl text transformation (camelCase, snake_case)
✅ Perl email/URL/IP extraction with regex
✅ Perl multi-line pattern matching
✅ jq JSON flatten/unflatten, pivot, aggregate, schema

---

**→ Part 52: Network Programming and Automation**
