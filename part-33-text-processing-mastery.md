# Part 33: Text Processing Mastery
## หลักสูตร Bash/Shell Script ระดับ Advanced

---

## 33.1 Advanced awk Patterns

```bash
# ─── Multi-file Processing ─────────────────────────────────────
awk '
FNR == 1 {
    if (NR > 1) print "─── End of", prev_file, "───"
    print "─── Start of", FILENAME, "───"
    prev_file = FILENAME
}
{ print NR, FNR, $0 }
END { print "─── End of", prev_file, "───" }
' file1.txt file2.txt

# ─── Join Files (SQL JOIN) ─────────────────────────────────────
# Inner join on field 1
awk '
NR == FNR { map[$1] = $0; next }
$1 in map { print map[$1], $0 }
' left.csv right.csv

# Left outer join
awk '
NR == FNR { map[$1] = $2; next }
{
    val = ($1 in map) ? map[$1] : "NULL"
    print $0, val
}
' right.csv left.csv

# ─── Pivot Table ───────────────────────────────────────────────
awk '
{
    date = $1; product = $2; qty = $3
    data[date][product] += qty
    products[product] = 1
    dates[date] = 1
}
END {
    printf "Date"
    for (p in products) printf "\t%s", p
    print ""
    
    for (d in dates) {
        printf "%s", d
        for (p in products) printf "\t%s", data[d][p]+0
        print ""
    }
}
' sales.txt | column -t

# ─── Running Totals & Moving Average ───────────────────────────
awk '
{
    values[NR] = $1
    sum += $1
    
    N = 5
    if (NR >= N) {
        window_sum = 0
        for (i = NR-N+1; i <= NR; i++) window_sum += values[i]
        printf "%d\t%.2f\t%.2f\n", $1, sum/NR, window_sum/N
    } else {
        printf "%d\t%.2f\t-\n", $1, sum/NR
    }
}
' numbers.txt

# ─── Report Generator ──────────────────────────────────────────
awk -F, '
BEGIN {
    print "Sales Report"
    print "============"
    OFS = "\t"
}
NR > 1 {
    region[$2] += $3
    product[$1] += $3
    total += $3
    count++
}
END {
    print "\nBy Region:"
    for (r in region) printf "  %-20s $%10.2f\n", r, region[r]
    
    print "\nTop Products:"
    for (p in product) printf "  %-20s $%10.2f\n", p, product[p]
    
    printf "\nTotal: $%.2f (%d transactions)\n", total, count
}
' sales.csv
```

---

## 33.2 Sed Advanced Techniques

```bash
# ─── Hold Space Patterns ───────────────────────────────────────
# Reverse file lines (tac alternative)
sed -n '1!G;h;$p' file.txt

# Print paragraph containing pattern
sed -n '/START/,/END/{
    /START/!{/END/!p}
}' file.txt

# Delete lines matching pattern AND next line
sed '/pattern/{N;d}' file.txt

# Print every other line
sed -n 'p;n' file.txt

# ─── In-place Editing ──────────────────────────────────────────
# With backup
sed -i.bak 's/old/new/g' file.txt

# Multiple files
sed -i 's/foo/bar/g' *.txt

# With conditions
sed -i '/^#/d' config.txt  # delete comment lines

# ─── Multi-line Patterns ───────────────────────────────────────
# Match multi-line block
sed -n '/BEGIN/,/END/p' file.txt

# Substitute across two lines
sed 'N;s/\n/ /' file.txt  # join pairs of lines

# Delete blank lines
sed '/^$/d' file.txt

# Remove multiple consecutive blank lines
sed '/^$/N;/^\n$/d' file.txt

# ─── Address Ranges ────────────────────────────────────────────
# Line range
sed -n '5,10p' file.txt

# Pattern range
sed -n '/start/,/end/p' file.txt

# Every Nth line (e.g., every 3rd)
sed -n '0~3p' file.txt

# First occurrence only
sed '0,/pattern/s/pattern/replacement/' file.txt

# ─── sed as Template Engine ────────────────────────────────────
render_template() {
    local template=$1
    local -n vars=$2
    
    local cmd="sed"
    for key in "${!vars[@]}"; do
        cmd+=" -e 's/{{${key}}}/${vars[$key]}/g'"
    done
    
    eval "$cmd" "$template"
}

declare -A values=(
    [NAME]="World"
    [DATE]=$(date +%Y-%m-%d)
    [VERSION]="1.0"
)

render_template template.txt values
```

---

## 33.3 Perl One-Liners

```bash
# ─── Basic Perl one-liners ─────────────────────────────────────
# Print matching lines (grep alternative)
perl -ne 'print if /pattern/' file.txt

# Substitute (sed alternative)
perl -pi -e 's/old/new/g' file.txt

# Print specific fields (awk alternative)
perl -lane 'print $F[0]' file.txt
perl -F: -lane 'print $F[0]' /etc/passwd

# ─── Perl for Complex Transforms ───────────────────────────────
# Deduplicate lines preserving order
perl -ne 'print unless $seen{$_}++' file.txt

# Extract emails
perl -ne 'print "$1\n" while /[\w.+-]+@[\w-]+\.[\w.]+/g' file.txt

# Process JSON lines
cat data.jsonl | perl -MJSON -ne '
    my $obj = decode_json($_);
    print "$obj->{id}: $obj->{name}\n";
'

# Calculate sum of column
perl -lane '$sum += $F[2]; END { print $sum }' data.txt

# Sliding window
perl -e '
    @a = (1..10);
    $n = 3;
    for my $i (0..$#a-$n+1) {
        my @w = @a[$i..$i+$n-1];
        printf "Window %d: %s\n", $i, join(",", @w);
    }
'

# ─── Perl as sed with lookahead ────────────────────────────────
echo "foo bar foo baz" | perl -pe 's/foo(?= baz)/qux/'

# Complex multi-line transforms
perl -0777 -pe 's/START.*?END/REPLACED/gs' file.txt
```

---

## 33.4 grep Advanced Usage

```bash
# ─── Context and Groups ────────────────────────────────────────
grep -A 3 "pattern" file.txt
grep -B 3 "pattern" file.txt
grep -C 3 "pattern" file.txt
grep -o "pattern" file.txt
grep -v "pattern" file.txt
grep -i "pattern" file.txt
grep -F "literal.string" file.txt

# ─── PCRE grep (grep -P) ───────────────────────────────────────
grep -P "foo(?=bar)" file.txt
grep -P "(?<=foo)bar" file.txt
grep -oP "start.*?end" file.txt
grep -oP "(?P<year>\d{4})-(?P<month>\d{2})" file.txt

# ─── Recursive and File Selection ──────────────────────────────
grep -r "pattern" /etc/
grep -r --include="*.py" "import" .
grep -r --exclude="*.log" --exclude-dir=".git" "pattern" .

# ─── Combining with Other Tools ────────────────────────────────
find . -name "*.sh" | xargs grep -l "TODO"
find . -name "*.log" -print0 | xargs -0 -P4 grep -l "ERROR"
grep -r -c "pattern" . | grep -v ":0$"
```

---

## 33.5 Complete Log Analysis Pipeline

```bash
#!/bin/bash
# log_pipeline.sh - Advanced log analysis

# ─── Parse access log into structured format ───────────────────
parse_access_log() {
    gawk '
    match($0, /^([0-9.]+) .+ \[([^\]]+)\] "([A-Z]+) ([^ ]+)[^"]*" ([0-9]+) ([0-9-]+) "([^"]*)" "([^"]*)"/, a) {
        ip = a[1]
        time = a[2]
        method = a[3]
        url = a[4]
        status = a[5]
        size = (a[6] == "-") ? 0 : a[6]
        referer = a[7]
        ua = a[8]
        
        split(time, t, /[/:[ ]/)
        day = t[1]; mon = t[2]; year = t[3]; hour = t[4]
        
        printf "%s\t%s\t%s-%s-%s\t%s\t%s\t%s\t%s\n",
            ip, method, year, mon, day, hour, url, status, size
    }'
}

# ─── Top analysis ──────────────────────────────────────────────
top_analysis() {
    local parsed_file=$1
    
    echo "=== Traffic Analysis ==="
    echo ""
    
    echo "Top 10 IPs:"
    awk '{count[$1]++} END {for(k in count) print count[k], k}' "$parsed_file" | \
        sort -rn | head -10 | \
        awk '{printf "  %-6d %s\n", $1, $2}'
    echo ""
    
    echo "HTTP Methods:"
    awk '{count[$2]++} END {for(k in count) print count[k], k}' "$parsed_file" | \
        sort -rn | \
        awk '{printf "  %-6d %s\n", $1, $2}'
    echo ""
    
    echo "Status Codes:"
    awk '{count[$8]++} END {for(k in count) print count[k], k}' "$parsed_file" | \
        sort -rn | \
        awk '{printf "  %-6d %s\n", $1, $2}'
    echo ""
    
    echo "Traffic by Hour:"
    awk '{count[$4]++} END {for(h in count) print h, count[h]}' "$parsed_file" | \
        sort | \
        awk '{printf "  %s:00  %d\n", $1, $2}'
}

# ─── Anomaly Detection ─────────────────────────────────────────
detect_anomalies() {
    local parsed_file=$1
    
    echo "=== Anomaly Detection ==="
    echo ""
    
    echo "High traffic IPs (>100 requests):"
    awk '{count[$1]++} END {for(k in count) if(count[k]>100) print count[k], k}' \
        "$parsed_file" | sort -rn | \
        awk '{printf "  %-6d %s\n", $1, $2}'
    echo ""
    
    echo "IPs with high error rates (>50%):"
    awk '{
        total[$1]++
        if ($8 >= 400) errors[$1]++
    }
    END {
        for (ip in total) {
            if (total[ip] >= 10) {
                rate = errors[ip]+0 / total[ip] * 100
                if (rate > 50) printf "  %s: %.0f%% errors (%d/%d)\n",
                    ip, rate, errors[ip]+0, total[ip]
            }
        }
    }' "$parsed_file" | sort -t: -k2 -rn
}

main() {
    local logfile="${1:-/var/log/nginx/access.log}"
    local tmpfile
    tmpfile=$(mktemp)
    
    echo "Parsing log: $logfile"
    parse_access_log < "$logfile" > "$tmpfile"
    echo "Parsed $(wc -l < "$tmpfile") entries"
    echo ""
    
    top_analysis "$tmpfile"
    detect_anomalies "$tmpfile"
    
    rm -f "$tmpfile"
}

main "$@"
```

---

## 33.6 Exercises

### Exercise 1: Log Diff
สร้าง tool เปรียบเทียบ logs สองวัน:
- New error types
- Resolved errors
- Traffic changes

### Exercise 2: Config File Editor
สร้าง tool แก้ไข config files:
- Add/remove settings
- Change values
- Comment/uncomment
- Multiple formats (ini, yaml, env)

### Exercise 3: Report Generator
สร้าง report generator:
- Input: raw data CSV
- Process with awk/sed
- Output: formatted report
- Include charts (ASCII)

---

## สรุป Part 33

✅ Advanced awk (join, pivot, moving average)  
✅ sed hold space, multi-line, template engine  
✅ Perl one-liners for complex transforms  
✅ grep PCRE, lookahead/behind  
✅ Complete log analysis pipeline  
✅ Anomaly detection  

---

**→ Part 34: Network Programming & Protocol Handling**
