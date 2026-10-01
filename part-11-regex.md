# Part 11: Regular Expressions
## หลักสูตร Bash/Shell Script ระดับมืออาชีพ

---

## 11.1 Regex Basics

Regular Expression (Regex) คือ pattern สำหรับ matching text

```bash
# เครื่องมือที่ใช้ regex
grep, egrep, sed, awk, bash [[ =~ ]]

# Types of regex:
# BRE  - Basic Regular Expression (grep, sed default)
# ERE  - Extended Regular Expression (grep -E, awk)
# PCRE - Perl Compatible (grep -P)

# สรุป metacharacters
.     # ตัวอักษรใดก็ได้ 1 ตัว (ยกเว้น newline)
*     # 0 หรือมากกว่า ของสิ่งก่อนหน้า
+     # 1 หรือมากกว่า (ERE/PCRE)
?     # 0 หรือ 1 ครั้ง (ERE/PCRE)
^     # ต้นบรรทัด (หรือ negation ใน character class)
$     # ท้ายบรรทัด
[]    # character class
[^]   # negated character class
()    # grouping
|     # alternation (OR) (ERE/PCRE)
\     # escape character
{n}   # exactly n times (ERE/PCRE)
{n,}  # n or more times
{n,m} # between n and m times
```

---

## 11.2 Character Classes

```bash
# Literal characters
echo "hello" | grep "hello"

# . - any character
echo "hat" | grep "h.t"    # matches hat, hot, h1t, etc.
echo "hot" | grep "h.t"    # matches
echo "ht" | grep "h.t"     # no match (needs char between h and t)

# Character classes [ ]
echo "cat" | grep "[abc]at"    # matches cat, aat, bat
echo "bat" | grep "[abc]at"    # matches
echo "dat" | grep "[abc]at"    # no match

# Range in character class
echo "a5" | grep "[a-z][0-9]"   # lowercase + digit
echo "A5" | grep "[A-Z][0-9]"   # uppercase + digit
echo "Az" | grep "[A-Za-z]"     # any letter

# Negation [^ ]
echo "hello" | grep "[^aeiou]"   # has consonants
echo "aeiou" | grep "[^aeiou]"   # no match

# POSIX Character Classes
# [:alpha:]  = letters
# [:digit:]  = digits 0-9
# [:alnum:]  = letters + digits
# [:space:]  = whitespace (space, tab, newline)
# [:upper:]  = uppercase A-Z
# [:lower:]  = lowercase a-z
# [:punct:]  = punctuation
# [:print:]  = printable characters
# [:graph:]  = printable non-space characters
# [:blank:]  = space and tab only
# [:cntrl:]  = control characters
# [:xdigit:] = hex digits 0-9a-fA-F

echo "Hello123" | grep "[[:alpha:]]"    # has letters
echo "test 123" | grep "[[:space:]]"    # has whitespace
echo "price: $10" | grep "[[:punct:]]" # has punctuation

# Escaped special chars in class
echo "[test]" | grep "\[.*\]"   # literal brackets
echo "file.txt" | grep "\."     # literal dot
echo "a*b" | grep "a\*b"        # literal asterisk
```

---

## 11.3 Anchors & Boundaries

```bash
# ^ - start of line
echo "hello world" | grep "^hello"   # matches (starts with hello)
echo "say hello" | grep "^hello"     # no match

# $ - end of line
echo "hello world" | grep "world$"   # matches
echo "world is" | grep "world$"      # no match

# ^$ - empty line
echo "" | grep "^$"                  # matches empty line
grep -c "^$" file.txt                # count empty lines

# \b - word boundary (grep -P or grep -w equivalent)
echo "test testing" | grep -P "\btest\b"    # only "test"
echo "testing" | grep -P "\btest\b"         # no match

# \B - non-word boundary
echo "testing" | grep -P "\Btest"           # middle of word
echo "test" | grep -P "\Btest"              # no match

# \< \> - word boundaries (BRE/ERE)
echo "test testing" | grep "\<test\>"       # whole word
echo "test testing" | grep -E "\btest\b"    # same with ERE

# \A \Z \z - string anchors (PCRE)
echo "hello" | grep -P "\Ahello\z"          # entire string match

# Multiline anchors
echo -e "line1\nline2" | grep -P "^line2$"  # matches second line
echo -e "line1\nline2" | grep -Pzo "(?s)\Aline1\nline2\z"  # whole string
```

---

## 11.4 Quantifiers

```bash
# * - 0 or more (greedy)
echo "color" | grep -E "colou*r"    # matches color, colour, colouur
echo "colour" | grep -E "colou*r"   # matches
echo "aaa" | grep -E "a*"           # matches (0 or more)
echo "" | grep -E "a*"              # matches (0 occurrences)

# + - 1 or more (ERE)
echo "hello" | grep -E "l+"         # matches ll
echo "helo" | grep -E "l+"          # matches l

# ? - 0 or 1 (ERE)
echo "color" | grep -E "colou?r"    # matches color and colour
echo "colour" | grep -E "colou?r"   # matches

# {n} - exactly n times
echo "aaa" | grep -E "a{3}"         # matches
echo "aa" | grep -E "a{3}"          # no match

# {n,} - n or more
echo "aaa" | grep -E "a{2,}"        # matches (3 >= 2)
echo "a" | grep -E "a{2,}"          # no match

# {n,m} - between n and m
echo "aaa" | grep -E "a{2,4}"       # matches (3 is between 2-4)
echo "aaaaa" | grep -E "a{2,4}"     # matches (4 within match)
echo "a" | grep -E "a{2,4}"         # no match

# BRE quantifiers (different escaping)
echo "aaa" | grep "a\{3\}"          # exactly 3 (BRE needs \{)
echo "aaa" | grep -E "a{3}"         # exactly 3 (ERE no backslash)

# Greedy vs lazy (PCRE)
echo "<b>bold</b>" | grep -oP "<.*>"    # greedy: <b>bold</b>
echo "<b>bold</b>" | grep -oP "<.*?>"   # lazy: <b>
```

---

## 11.5 Groups & Backreferences

```bash
# Grouping ()
echo "hello" | grep -E "(hell)o"    # group for repetition
echo "haha" | grep -E "(ha)+"       # one or more "ha"
echo "ababab" | grep -E "(ab)+"     # repeated "ab"

# Alternation |
echo "cat" | grep -E "cat|dog"      # cat or dog
echo "dog" | grep -E "cat|dog"      # matches
echo "fish" | grep -E "cat|dog"     # no match

echo "hello" | grep -E "h(ello|i)" # hello or hi

# Backreferences \1 \2 etc.
echo "hello hello" | grep -E "(hello) \1"    # repeated word
echo "the the" | grep -E "(\w+) \1"          # any repeated word

# Find duplicate words
grep -P "(\b\w+\b)\s+\1" file.txt

# Find palindromes (simple)
echo "racecar" | grep -P "^(.)(.)(.)\3\2\1$"

# Non-capturing group (?:...) (PCRE)
echo "hello" | grep -oP "(?:hel)(lo)"   # group 1 = lo (not hel)

# Named groups (?P<name>...) (PCRE)
echo "2024-01-15" | grep -oP "(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})"

# Extract named groups in bash
date_str="2024-01-15"
if [[ "$date_str" =~ ^([0-9]{4})-([0-9]{2})-([0-9]{2})$ ]]; then
    year="${BASH_REMATCH[1]}"
    month="${BASH_REMATCH[2]}"
    day="${BASH_REMATCH[3]}"
    echo "Year: $year, Month: $month, Day: $day"
fi
```

---

## 11.6 Lookahead & Lookbehind (PCRE)

```bash
# Lookahead (?=...) - match if followed by pattern
echo "100USD" | grep -oP "\d+(?=USD)"       # 100 (followed by USD)
echo "100EUR" | grep -oP "\d+(?=USD)"       # no match

# Negative lookahead (?!...)
echo "test1" | grep -oP "\d+(?!test)"       # 1 (digit not before "test")
echo "foobar" | grep -P "foo(?!baz)"        # matches (foo not before baz)

# Lookbehind (?<=...) - match if preceded by pattern
echo "price: $100" | grep -oP "(?<=\$)\d+"    # 100 (after $)
echo "name: Alice" | grep -oP "(?<=name: )\w+" # Alice

# Negative lookbehind (?<!...)
echo "value=100" | grep -oP "(?<!USD)\d{3}"   # 100
echo "price100" | grep -oP "(?<!price)\d{3}"  # no match

# Combined lookahead/lookbehind
echo "function hello() {" | grep -oP "(?<=function )\w+(?=\()"  # hello
echo "import {useState} from 'react'" | grep -oP "(?<=\{)\w+(?=\})"  # useState

# Practical examples
# Extract value from key=value
echo "timeout=30" | grep -oP "(?<=timeout=)\d+"      # 30
echo "host=localhost" | grep -oP "(?<=host=)\S+"       # localhost

# Find words before comma
echo "apple, banana, cherry" | grep -oP "\w+(?=,)"    # apple banana

# Extract content between tags
echo "<title>My Page</title>" | grep -oP "(?<=<title>).*(?=</title>)"
```

---

## 11.7 Common Regex Patterns

```bash
# Email address
EMAIL_REGEX='^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
echo "user@example.com" | grep -P "$EMAIL_REGEX"

# IP address (IPv4)
IP_REGEX='^((25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$'
echo "192.168.1.1" | grep -P "$IP_REGEX"
echo "999.999.999.999" | grep -P "$IP_REGEX"    # no match

# URL
URL_REGEX='^https?://[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}(/.*)?$'
echo "https://www.example.com/path" | grep -P "$URL_REGEX"

# Date (YYYY-MM-DD)
DATE_REGEX='^[0-9]{4}-(0[1-9]|1[0-2])-(0[1-9]|[12][0-9]|3[01])$'
echo "2024-01-15" | grep -P "$DATE_REGEX"
echo "2024-13-01" | grep -P "$DATE_REGEX"    # no match

# Time (HH:MM:SS)
TIME_REGEX='^([01][0-9]|2[0-3]):[0-5][0-9]:[0-5][0-9]$'
echo "14:30:00" | grep -P "$TIME_REGEX"

# Phone (US)
PHONE_REGEX='^\+?1?[-.\s]?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}$'
echo "555-123-4567" | grep -P "$PHONE_REGEX"
echo "(555) 123-4567" | grep -P "$PHONE_REGEX"

# Credit card (basic)
CC_REGEX='^[0-9]{4}[-\s]?[0-9]{4}[-\s]?[0-9]{4}[-\s]?[0-9]{4}$'
echo "4532-1234-5678-9012" | grep -P "$CC_REGEX"

# ZIP code (US)
ZIP_REGEX='^\d{5}(-\d{4})?$'
echo "12345" | grep -P "$ZIP_REGEX"
echo "12345-6789" | grep -P "$ZIP_REGEX"

# Username (alphanumeric + underscore, 3-16 chars)
USERNAME_REGEX='^[a-zA-Z0-9_]{3,16}$'
echo "john_doe" | grep -P "$USERNAME_REGEX"
echo "ab" | grep -P "$USERNAME_REGEX"    # too short

# Password (8+ chars, upper, lower, digit, special)
PASSWORD_REGEX='^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[!@#$%^&*]).{8,}$'
echo "Password123!" | grep -P "$PASSWORD_REGEX"
echo "password" | grep -P "$PASSWORD_REGEX"    # no match

# MAC address
MAC_REGEX='^([0-9A-Fa-f]{2}[:-]){5}[0-9A-Fa-f]{2}$'
echo "00:1A:2B:3C:4D:5E" | grep -P "$MAC_REGEX"

# IPv6
IPV6_REGEX='^([0-9a-fA-F]{0,4}:){2,7}[0-9a-fA-F]{0,4}$'
echo "2001:0db8:85a3:0000:0000:8a2e:0370:7334" | grep -P "$IPV6_REGEX"

# Semantic Version
SEMVER_REGEX='^(0|[1-9]\d*)\.(0|[1-9]\d*)\.(0|[1-9]\d*)$'
echo "1.2.3" | grep -P "$SEMVER_REGEX"
echo "1.02.3" | grep -P "$SEMVER_REGEX"    # no match (leading zero)
```

---

## 11.8 Regex in Bash Scripts

```bash
#!/bin/bash

# Using =~ operator
validate() {
    local type=$1
    local value=$2
    
    case $type in
        email)
            [[ "$value" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]
            ;;
        ip)
            [[ "$value" =~ ^([0-9]{1,3}\.){3}[0-9]{1,3}$ ]] && {
                IFS='.' read -ra octets <<< "$value"
                for octet in "${octets[@]}"; do
                    (( octet >= 0 && octet <= 255 )) || return 1
                done
            }
            ;;
        date)
            [[ "$value" =~ ^[0-9]{4}-[0-9]{2}-[0-9]{2}$ ]]
            ;;
        number)
            [[ "$value" =~ ^-?[0-9]+(\.[0-9]+)?$ ]]
            ;;
        *)
            return 1
            ;;
    esac
}

# Test validations
tests=(
    "email:user@example.com:valid"
    "email:notanemail:invalid"
    "ip:192.168.1.1:valid"
    "ip:256.1.1.1:invalid"
    "date:2024-01-15:valid"
    "date:2024-13-45:invalid"
    "number:3.14:valid"
    "number:abc:invalid"
)

for test in "${tests[@]}"; do
    IFS=: read -r type value expected <<< "$test"
    if validate "$type" "$value"; then
        result="valid"
    else
        result="invalid"
    fi
    
    if [[ "$result" == "$expected" ]]; then
        printf "✓ %-10s %-20s = %s\n" "$type" "$value" "$result"
    else
        printf "✗ %-10s %-20s = %s (expected: %s)\n" "$type" "$value" "$result" "$expected"
    fi
done

# Extract all URLs from a file
extract_urls() {
    local file=$1
    grep -oP 'https?://[^\s<>"]+' "$file" | sort -u
}

# Extract all IPs
extract_ips() {
    local file=$1
    grep -oP '\b(?:[0-9]{1,3}\.){3}[0-9]{1,3}\b' "$file" | sort -u
}

# Extract all emails
extract_emails() {
    local file=$1
    grep -oP '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' "$file" | sort -u
}
```

---

## 11.9 sed with Advanced Regex

```bash
# Capture groups in sed
echo "John Smith" | sed 's/\([A-Za-z]*\) \([A-Za-z]*\)/\2, \1/'
# → Smith, John

# GNU sed extended regex
echo "John Smith" | sed -E 's/([A-Za-z]+) ([A-Za-z]+)/\2, \1/'

# Remove HTML tags
echo "<p>Hello <b>World</b></p>" | sed 's/<[^>]*>//g'
# → Hello World

# Extract between quotes
echo 'name: "Alice Smith"' | sed -n 's/.*"\(.*\)".*/\1/p'
# → Alice Smith

# Add prefix to each line
sed 's/^/PREFIX: /' file.txt

# Remove trailing whitespace
sed 's/[[:space:]]*$//' file.txt

# Normalize multiple spaces
echo "a  b   c" | sed 's/  */ /g'
# → a b c

# Replace with contents of match
echo "hello world" | sed 's/\b\w/[\0]/g'   # GNU: wrap first char
echo "hello" | sed 's/.*/[&]/'             # wrap entire match

# Multiline: join continuation lines (ending with \)
sed ':a;/\\$/{N;s/\\\n//;ba}' file.txt

# Delete between markers
sed '/START/,/END/d' file.txt
```

---

## 11.10 awk with Regex

```bash
# awk regex matching
awk '/pattern/' file.txt                # print matching lines
awk '!/pattern/' file.txt               # print non-matching
awk '$3 ~ /regex/' file.txt             # field 3 matches
awk '$3 !~ /regex/' file.txt            # field 3 doesn't match
awk '/start/,/end/' file.txt            # range pattern

# Case insensitive (GNU awk)
awk 'BEGIN{IGNORECASE=1} /hello/' file.txt
echo "HELLO" | awk '{if (tolower($0) ~ /hello/) print}'

# sub/gsub functions
echo "hello world" | awk '{gsub(/world/, "bash"); print}'
echo "aaa bbb" | awk '{sub(/a+/, "X"); print}'    # first only

# match function
echo "hello123world" | awk '{
    if (match($0, /[0-9]+/)) {
        print "Found:", substr($0, RSTART, RLENGTH), "at pos:", RSTART
    }
}'

# gensub (GNU awk) - with backreferences
echo "John Smith" | awk '{print gensub(/([A-Za-z]+) ([A-Za-z]+)/, "\\2, \\1", "g")}'

# Extract data with regex
awk 'match($0, /ERROR: (.+)/, arr) {print arr[1]}' logfile.txt

# Validate and process
awk '
$1 ~ /^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$/ {
    print "IP:", $1, "Requests:", $2
}
' data.txt
```

---

## 11.11 PCRE Features (grep -P)

```bash
# \d \w \s shortcuts
echo "test123" | grep -P "^\w+\d+$"      # word chars + digits
echo "hello world" | grep -P "\w+\s\w+"  # word space word

# \d = [0-9]
# \D = [^0-9]
# \w = [a-zA-Z0-9_]
# \W = [^a-zA-Z0-9_]
# \s = [ \t\n\r\f\v]
# \S = [^ \t\n\r\f\v]
# \b = word boundary
# \B = non-word boundary

# Non-greedy
echo "<a>text</a>" | grep -oP "<a>.*?</a>"   # lazy: smallest match

# Atomic groups (?>...)
echo "hello" | grep -P "(?>hel)lo"

# Possessive quantifiers
echo "aaa" | grep -P "a++b"    # fails fast (possessive)

# Inline flags
echo "HELLO" | grep -P "(?i)hello"    # case insensitive inline
echo -e "line1\nline2" | grep -P "(?m)^line"  # multiline mode
echo -e "a\nb\nc" | grep -Pzo "(?s)a.*c"     # dot matches newline

# Conditional (?condition|yes|no)
echo "abc" | grep -P "(?(1)a|b)"   # complex conditionals

# Unicode properties (requires PCRE2)
echo "café" | grep -P "\p{L}+"    # unicode letters
```

---

## 11.12 Practical Regex Tools

```bash
#!/usr/bin/env bash
# regex_tools.sh - Practical regex utilities

# Test regex against string
regex_test() {
    local pattern=$1
    local string=$2
    local type=${3:-P}   # P=PCRE, E=ERE, G=BRE
    
    echo -n "Pattern '$pattern' matches '$string': "
    if echo "$string" | grep -q"-${type}" "$pattern"; then
        echo "YES"
        echo "Matched parts:"
        echo "$string" | grep -o"-${type}" "$pattern"
    else
        echo "NO"
    fi
}

# Extract all matches
extract_all() {
    local pattern=$1
    local input=$2
    echo "$input" | grep -oP "$pattern"
}

# Replace with function (using perl)
regex_replace_fn() {
    local pattern=$1
    local replacement=$2   # can include $1 etc.
    local input=$3
    
    echo "$input" | perl -pe "s/$pattern/$replacement/g"
}

# Find files containing regex
find_in_files() {
    local pattern=$1
    local dir=${2:-.}
    local extension=${3:-"*"}
    
    find "$dir" -name "*.$extension" -type f | \
        xargs grep -lP "$pattern" 2>/dev/null
}

# Count pattern occurrences across files
count_pattern() {
    local pattern=$1
    local dir=${2:-.}
    
    grep -rP --include="*.txt" -c "$pattern" "$dir" 2>/dev/null | \
        awk -F: '$2>0 {print $2, $1}' | \
        sort -rn
}

# Interactive regex tester
regex_repl() {
    echo "Regex REPL - Type pattern, then 'quit' to exit"
    
    while true; do
        read -rp "Pattern (PCRE): " pattern
        [[ "$pattern" == "quit" ]] && break
        
        read -rp "String: " string
        
        if [[ "$string" =~ $pattern ]]; then
            echo "MATCH!"
            echo "Full match: ${BASH_REMATCH[0]}"
            for i in "${!BASH_REMATCH[@]}"; do
                [[ $i -eq 0 ]] && continue
                echo "Group $i: ${BASH_REMATCH[$i]}"
            done
        else
            echo "No match"
        fi
        echo ""
    done
}
```

---

## 11.13 Exercises

### Exercise 1: Validator Library
สร้าง bash library ที่ validate:
- Email, URL, IP (v4/v6), MAC address
- Date formats (YYYY-MM-DD, DD/MM/YYYY)
- Phone numbers (Thai format: 0x-xxx-xxxx)
- Thai ID card (13 digits + checksum)

### Exercise 2: Log Parser
จาก nginx access log ด้วย regex:
- Extract: IP, timestamp, method, URL, status, bytes
- Store ใน associative arrays
- Generate report

### Exercise 3: Code Scanner
สร้าง script ที่ scan source code:
- หา TODO, FIXME, HACK, XXX comments
- หา hardcoded credentials (password=, secret=, api_key=)
- หา IP addresses ที่ hardcode
- Export report

### Exercise 4: HTML Extractor
Parse HTML file:
- Extract all links (href)
- Extract all images (src)
- Extract all email addresses
- Extract all phone numbers

---

## สรุป Part 11

✅ BRE, ERE, PCRE types  
✅ Metacharacters (. * + ? ^ $ [])  
✅ Character classes (POSIX classes)  
✅ Anchors (^ $ \b \B)  
✅ Quantifiers (* + ? {n,m})  
✅ Groups & backreferences  
✅ Lookahead & lookbehind  
✅ Common patterns (email, IP, URL, date)  
✅ Regex in bash (=~, BASH_REMATCH)  
✅ sed & awk with advanced regex  

---

**→ Part 12: Process Management**
