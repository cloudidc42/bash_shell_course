# Part 02: Variables, Data Types & Arithmetic
## หลักสูตร Bash/Shell Script ระดับมืออาชีพ

---

## 2.1 Variables พื้นฐาน

Bash ไม่มี strict type system — ทุกอย่างเป็น string โดยค่าเริ่มต้น

```bash
#!/bin/bash

# กำหนดค่า variable (ห้ามมีช่องว่างรอบ =)
name="John Doe"
age=25
pi=3.14159
is_active=true

# อ่านค่า variable
echo $name
echo ${name}          # ใช้ curly braces (แนะนำ)
echo "Hello, $name"
echo "Hello, ${name}!"

# ผิดพลาดบ่อย:
name = "John"         # ERROR: ห้ามมีช่องว่าง
echo name             # พิมพ์คำว่า "name" ไม่ใช่ค่า
echo "$name"          # ถูกต้อง - quote เพื่อความปลอดภัย
```

### Naming Rules

```bash
# ถูกต้อง:
my_var=1
MY_VAR=2
_private=3
var123=4
CONSTANT_VALUE=5

# ผิดต้อง:
1var=bad        # ห้ามขึ้นด้วยตัวเลข
my-var=bad      # ห้ามใช้ hyphen
my var=bad      # ห้ามมีช่องว่าง
my.var=bad      # ห้ามใช้ dot

# Convention:
lower_case=       # local variables
UPPER_CASE=       # constants / environment variables
_leading_under=   # private/internal
```

---

## 2.2 ประเภทข้อมูล (Types)

```bash
# 1. String
name="Alice"
greeting='Hello, World!'        # single quote = literal (no expansion)
message="Hi, ${name}!"          # double quote = with expansion

# 2. Integer
count=42
negative=-10
hex=0xFF                        # hexadecimal
octal=077                       # octal
binary=$((2#1010))              # binary → decimal

# 3. Float (Bash ไม่ support โดยตรง ต้องใช้ bc หรือ awk)
# ไม่ใช่ float จริงๆ - เป็นแค่ string!
pi=3.14159

# 4. Array
fruits=("apple" "banana" "cherry")
numbers=(1 2 3 4 5)

# 5. Associative Array (Bash 4+)
declare -A person
person[name]="Bob"
person[age]=30

# 6. Boolean (ไม่มีจริงๆ ใช้ 0/1 หรือ true/false string)
flag=true
if [ "$flag" = "true" ]; then echo "yes"; fi
```

---

## 2.3 Variable Attributes (declare)

```bash
# declare กำหนด attributes ให้ variable

# Integer
declare -i num=10
declare -i result=5*3    # คำนวณอัตโนมัติ
echo $result             # 15

# Readonly (constant)
declare -r PI=3.14159
readonly MAX_SIZE=100
PI=3.0    # ERROR: readonly variable

# Lowercase
declare -l text="HELLO"
echo $text               # hello

# Uppercase
declare -u text2="hello"
echo $text2              # HELLO

# Array
declare -a my_array
declare -A my_assoc_array

# Export (environment variable)
declare -x MY_VAR="exported"
export ANOTHER_VAR="also exported"

# Print all attributes
declare -p MY_VAR
# declare -x MY_VAR="exported"
```

---

## 2.4 Special Variables

```bash
# Positional Parameters
$0          # ชื่อ script
$1          # argument ที่ 1
$2          # argument ที่ 2
${10}       # argument ที่ 10 (ต้องใช้ braces)
$@          # arguments ทั้งหมด (แยกกัน)
$*          # arguments ทั้งหมด (รวมเป็น string)
$#          # จำนวน arguments

# Process Variables
$$          # PID ของ shell ปัจจุบัน
$!          # PID ของ background job ล่าสุด
$?          # exit code ของ command ล่าสุด
$-          # flags ที่เปิดอยู่ใน shell

# Example
#!/bin/bash
echo "Script name: $0"
echo "First arg: $1"
echo "All args: $@"
echo "Arg count: $#"

# รัน: bash script.sh hello world 123
# Script name: script.sh
# First arg: hello
# All args: hello world 123
# Arg count: 3

# $@ vs $*
args=("one" "two" "three words")
for arg in "$@"; do echo "[@] $arg"; done
# [@] one
# [@] two
# [@] three words    ← รักษา spacing

for arg in "$*"; do echo "[*] $arg"; done
# [*] one two three words   ← รวมเป็น string เดียว
```

---

## 2.5 Variable Expansion & Substitution

```bash
# Basic
var="hello"
echo ${var}                  # hello

# Default Values
echo ${var:-"default"}       # ใช้ default ถ้า var ว่างหรือ unset
echo ${var:="default"}       # กำหนด default ถ้า var ว่างหรือ unset
echo ${var:+"alternative"}   # ใช้ alternative ถ้า var มีค่า
echo ${var:?"error message"} # error ถ้า var ว่างหรือ unset

# String Length
name="Hello World"
echo ${#name}                # 11

# Substring
str="Hello, World!"
echo ${str:7}                # World!
echo ${str:7:5}              # World
echo ${str: -6}              # orld! (จากท้าย)
echo ${str:0:5}              # Hello

# String Removal (Prefix/Suffix)
filename="/home/user/file.txt"
echo ${filename#*/}          # home/user/file.txt (ลบ prefix สั้นสุด)
echo ${filename##*/}         # file.txt (ลบ prefix ยาวสุด)
echo ${filename%.*}          # /home/user/file (ลบ suffix สั้นสุด)
echo ${filename%%.*}         # /home/user/file (ลบ suffix ยาวสุด)
echo ${filename%/*}          # /home/user (dirname)
echo ${filename##*/}         # file.txt (basename)

# String Replacement
text="Hello World Hello"
echo ${text/Hello/Hi}        # Hi World Hello (แทนครั้งแรก)
echo ${text//Hello/Hi}       # Hi World Hi (แทนทั้งหมด)
echo ${text/#Hello/Hi}       # Hi World Hello (แทน prefix)
echo ${text/%Hello/Hi}       # Hello World Hi (แทน suffix)

# Case Conversion (Bash 4+)
name="hello world"
echo ${name^}                # Hello world (ตัวแรก uppercase)
echo ${name^^}               # HELLO WORLD (ทั้งหมด uppercase)

name2="HELLO WORLD"
echo ${name2,}               # hELLO WORLD (ตัวแรก lowercase)
echo ${name2,,}              # hello world (ทั้งหมด lowercase)
```

---

## 2.6 Arithmetic Operations

### Integer Arithmetic

```bash
# วิธีที่ 1: $(( )) - Arithmetic Expansion (แนะนำ)
echo $((5 + 3))              # 8
echo $((10 - 4))             # 6
echo $((3 * 7))              # 21
echo $((15 / 4))             # 3 (integer division)
echo $((15 % 4))             # 3 (modulo)
echo $((2 ** 10))            # 1024 (power)

# กำหนดค่า
num=10
echo $((num + 5))            # 15
echo $((num * 2))            # 20

# วิธีที่ 2: let
let "result = 5 + 3"
let result=5+3
echo $result

# วิธีที่ 3: expr (เก่า ไม่แนะนำ)
result=$(expr 5 + 3)
echo $result

# Increment/Decrement
i=5
((i++))      # post-increment
((i--))      # post-decrement
((++i))      # pre-increment
((--i))      # pre-decrement
((i += 10))  # add assign
((i -= 3))   # subtract assign
((i *= 2))   # multiply assign
((i /= 4))   # divide assign
((i **= 3))  # power assign

# Bitwise Operations
echo $((5 & 3))    # AND = 1
echo $((5 | 3))    # OR = 7
echo $((5 ^ 3))    # XOR = 6
echo $((~5))       # NOT = -6
echo $((5 << 1))   # Left shift = 10
echo $((10 >> 1))  # Right shift = 5

# Comparison in arithmetic
((5 > 3))    && echo "true"
((5 == 5))   && echo "equal"
((5 != 3))   && echo "not equal"
```

### Float Arithmetic (bc)

```bash
# bc - basic calculator
echo "scale=2; 10 / 3" | bc           # 3.33
echo "scale=5; sqrt(2)" | bc -l       # 1.41421
echo "scale=2; 3.14 * 2^2" | bc       # 12.56

# เก็บในตัวแปร
result=$(echo "scale=4; 22/7" | bc)
echo "Pi ≈ $result"

# Functions ใน bc
echo "scale=10; a(1)*4" | bc -l       # π (atan(1)*4)
echo "scale=5; e(1)" | bc -l          # e (exponential)
echo "scale=5; l(10)" | bc -l         # natural log of 10
echo "scale=5; s(3.14/2)" | bc -l     # sin(π/2) ≈ 1

# awk สำหรับ float
awk 'BEGIN {printf "%.4f\n", 22/7}'    # 3.1429
awk 'BEGIN {printf "%.2f\n", 3.14*2^2}'  # 12.56
result=$(awk 'BEGIN {print 10/3}')

# python3 alternative
python3 -c "print(22/7)"
python3 -c "import math; print(math.pi)"
```

---

## 2.7 String Operations

```bash
# String concatenation
first="Hello"
second="World"
combined="${first} ${second}"
echo $combined               # Hello World

# String ซ้ำ
str="ab"
repeated=$(printf '%0.s'"$str" {1..5})
echo $repeated               # ababababab

# String ซ้ำด้วย bash 4
printf -v repeated '%*s' 5 | tr ' ' 'X'

# Multiline string
text="Line 1
Line 2
Line 3"
echo "$text"

# Here string
cat <<< "This is a here string"
read var <<< "hello"
echo $var

# String comparison
str1="apple"
str2="banana"

if [[ "$str1" == "$str2" ]]; then echo "equal"; fi
if [[ "$str1" != "$str2" ]]; then echo "not equal"; fi
if [[ "$str1" < "$str2" ]]; then echo "apple < banana (lexicographic)"; fi
if [[ "$str1" > "$str2" ]]; then echo "apple > banana"; fi

# String contains
if [[ "$str1" == *"ppl"* ]]; then echo "contains ppl"; fi

# Starts/Ends with
if [[ "$str1" == app* ]]; then echo "starts with app"; fi
if [[ "$str1" == *le ]]; then echo "ends with le"; fi

# String length
echo ${#str1}                # 5

# Check empty string
var=""
if [[ -z "$var" ]]; then echo "empty"; fi
if [[ -n "$var" ]]; then echo "not empty"; fi
```

---

## 2.8 Environment Variables

```bash
# ดู environment variables ทั้งหมด
env
printenv
declare -x    # แสดงเฉพาะ exported

# ดูตัวแปรเดียว
printenv HOME
printenv PATH

# กำหนด environment variable
export MY_VAR="hello"
export PATH="$PATH:/usr/local/myapp/bin"

# กำหนดชั่วคราว (เฉพาะ command นั้น)
MY_VAR=test command_here
DATABASE_URL=postgres://localhost/db python3 app.py

# Important Environment Variables
echo $HOME          # /home/username
echo $USER          # username
echo $SHELL         # /bin/bash
echo $PATH          # executable search path
echo $PWD           # current directory
echo $OLDPWD        # previous directory
echo $TERM          # terminal type
echo $LANG          # locale
echo $EDITOR        # default editor
echo $PAGER         # default pager (less/more)
echo $DISPLAY       # X11 display (GUI)
echo $TMPDIR        # temp directory
echo $LD_LIBRARY_PATH  # library search path
echo $MANPATH       # man page paths

# PATH Management
# แสดง PATH แบบ readable
echo $PATH | tr ':' '\n'

# เพิ่ม path
export PATH="$HOME/bin:$PATH"             # เพิ่มต้น
export PATH="$PATH:/opt/custom/bin"       # เพิ่มท้าย

# ลบ path (ต้องใช้ sed หรือ Python)
export PATH=$(echo $PATH | sed 's|:/old/path||')
```

---

## 2.9 Variable Scoping

```bash
#!/bin/bash

# Global variable
global_var="I am global"

function test_scope() {
    # Local variable
    local local_var="I am local"
    echo "Inside function: $global_var"    # เข้าถึง global ได้
    echo "Local var: $local_var"
    
    # แก้ไข global จากใน function
    global_var="Modified by function"
}

test_scope
echo "After function: $global_var"        # Modified by function
echo "Local from outside: ${local_var:-'undefined'}"  # undefined

# Subshell scoping
export outer="outer value"
(
    inner="inner value"
    echo "In subshell: $outer"            # เห็น outer
    outer="modified in subshell"
    echo "Modified: $outer"
)
echo "After subshell: $outer"             # ยังเป็น "outer value" (subshell ไม่กระทบ parent)

# Function with nameref (Bash 4.3+)
function modify_var() {
    local -n ref=$1                        # nameref
    ref="modified value"
}

my_var="original"
modify_var my_var
echo $my_var                              # modified value
```

---

## 2.10 Readonly และ Unset

```bash
# Readonly
readonly CONST=100
CONST=200    # bash: CONST: readonly variable

# Unset variable
var="hello"
echo ${var}          # hello
unset var
echo ${var:-"gone"}  # gone

# Cannot unset readonly
readonly FIXED=5
unset FIXED          # bash: unset: FIXED: cannot unset

# Check if variable is set
if [[ -v myvar ]]; then
    echo "myvar is set"
else
    echo "myvar is not set"
fi

# Check if set and not empty
if [[ -n "${myvar+x}" ]]; then
    echo "myvar is set (possibly empty)"
fi
```

---

## 2.11 Indirect Variables (Variable Variables)

```bash
# Indirect expansion
day="monday"
monday_task="Write report"
echo "${!day}"               # Write report

# Dynamic variable names
for color in red green blue; do
    declare "${color}_code"="#$(openssl rand -hex 3)"
done
echo $red_code

# namerefs (Bash 4.3+)
declare -n alias_name=original_var
original_var="hello"
echo $alias_name     # hello
alias_name="world"
echo $original_var   # world
```

---

## 2.12 Practical Examples

### ตัวอย่าง 1: Calculator Script

```bash
#!/bin/bash
# simple_calc.sh - เครื่องคิดเลขง่ายๆ

set -euo pipefail

usage() {
    echo "Usage: $0 <num1> <operator> <num2>"
    echo "Operators: + - * / % ^"
    echo "Example: $0 10 + 5"
    exit 1
}

[[ $# -ne 3 ]] && usage

num1=$1
op=$2
num2=$3

# Validate numbers
if ! [[ "$num1" =~ ^-?[0-9]+(\.[0-9]+)?$ ]]; then
    echo "Error: '$num1' is not a valid number"
    exit 1
fi
if ! [[ "$num2" =~ ^-?[0-9]+(\.[0-9]+)?$ ]]; then
    echo "Error: '$num2' is not a valid number"
    exit 1
fi

case $op in
    "+") result=$(echo "$num1 + $num2" | bc) ;;
    "-") result=$(echo "$num1 - $num2" | bc) ;;
    "*") result=$(echo "$num1 * $num2" | bc) ;;
    "/")
        if [[ "$num2" == "0" ]]; then
            echo "Error: Division by zero"
            exit 1
        fi
        result=$(echo "scale=6; $num1 / $num2" | bc)
        ;;
    "%") result=$(echo "$num1 % $num2" | bc) ;;
    "^") result=$(echo "scale=6; $num1 ^ $num2" | bc) ;;
    *) echo "Unknown operator: $op"; usage ;;
esac

echo "$num1 $op $num2 = $result"
```

### ตัวอย่าง 2: Variable Management

```bash
#!/bin/bash
# config_manager.sh - จัดการ configuration variables

CONFIG_FILE="${HOME}/.myapp/config"
declare -A CONFIG

# Load config
load_config() {
    [[ ! -f "$CONFIG_FILE" ]] && return 0
    while IFS='=' read -r key value; do
        [[ "$key" =~ ^[[:space:]]*# ]] && continue  # skip comments
        [[ -z "$key" ]] && continue                  # skip empty lines
        key="${key// /}"                              # trim spaces
        value="${value// /}"
        CONFIG["$key"]="$value"
    done < "$CONFIG_FILE"
}

# Save config
save_config() {
    mkdir -p "$(dirname "$CONFIG_FILE")"
    {
        echo "# MyApp Configuration"
        echo "# Generated: $(date)"
        for key in "${!CONFIG[@]}"; do
            echo "${key}=${CONFIG[$key]}"
        done
    } > "$CONFIG_FILE"
}

# Get config value
get_config() {
    local key=$1
    local default=${2:-""}
    echo "${CONFIG[$key]:-$default}"
}

# Set config value
set_config() {
    local key=$1
    local value=$2
    CONFIG["$key"]="$value"
}

# Usage
load_config
set_config "database_host" "localhost"
set_config "database_port" "5432"
set_config "app_debug" "false"

echo "DB Host: $(get_config database_host)"
echo "DB Port: $(get_config database_port)"
echo "Debug: $(get_config app_debug 'false')"

save_config
echo "Config saved to: $CONFIG_FILE"
```

### ตัวอย่าง 3: String Processing

```bash
#!/bin/bash
# string_tools.sh - เครื่องมือจัดการ string

# URL parser
parse_url() {
    local url=$1
    
    # Protocol
    local proto="${url%%://*}"
    # Remove protocol
    local rest="${url#*://}"
    # Host
    local host="${rest%%/*}"
    # Path
    local path="/${rest#*/}"
    # Port
    local port=""
    if [[ "$host" == *:* ]]; then
        port="${host#*:}"
        host="${host%:*}"
    fi
    
    echo "Protocol: $proto"
    echo "Host:     $host"
    echo "Port:     ${port:-'default'}"
    echo "Path:     $path"
}

# File path parser
parse_path() {
    local filepath=$1
    echo "Full path:  $filepath"
    echo "Directory:  ${filepath%/*}"
    echo "Filename:   ${filepath##*/}"
    echo "Extension:  ${filepath##*.}"
    echo "Basename:   ${filepath##*/}"
    echo "No ext:     ${filepath%.*}"
}

# String statistics
string_stats() {
    local str=$1
    local length=${#str}
    local words=$(echo "$str" | wc -w)
    local lines=$(echo "$str" | wc -l)
    local upper=$(echo "$str" | tr '[:lower:]' '[:upper:]')
    local lower=$(echo "$str" | tr '[:upper:]' '[:lower:]')
    local reversed=$(echo "$str" | rev)
    
    echo "Original:  $str"
    echo "Length:    $length"
    echo "Words:     $words"
    echo "Lines:     $lines"
    echo "Upper:     $upper"
    echo "Lower:     $lower"
    echo "Reversed:  $reversed"
}

# Demo
echo "=== URL Parser ==="
parse_url "https://www.example.com:8080/path/to/page"

echo ""
echo "=== Path Parser ==="
parse_path "/home/user/documents/report.pdf"

echo ""
echo "=== String Stats ==="
string_stats "Hello World, I am learning Bash!"
```

---

## 2.13 Common Mistakes & Best Practices

```bash
# ❌ ผิด: ช่องว่างรอบ =
name = "John"

# ✅ ถูก: ไม่มีช่องว่าง
name="John"

# ❌ ผิด: ไม่ quote variable
echo $filename             # อาจมีปัญหากับ spaces
rm $file                   # อันตราย!

# ✅ ถูก: quote เสมอ
echo "$filename"
rm "$file"

# ❌ ผิด: ใช้ backticks (เก่า)
result=`command`

# ✅ ถูก: ใช้ $()
result=$(command)

# ❌ ผิด: arithmetic แบบเก่า
result=`expr $a + $b`

# ✅ ถูก: arithmetic expansion
result=$((a + b))

# ❌ ผิด: เปรียบเทียบตัวเลขด้วย string operator
if [ "$num" == "5" ]; then

# ✅ ถูก: ใช้ arithmetic comparison
if [ "$num" -eq 5 ]; then
if (( num == 5 )); then

# ✅ Best practices:
set -euo pipefail          # ตั้งแต่ต้น script
readonly CONFIG="/etc/app"  # ค่าคงที่เป็น uppercase readonly
declare -i count=0          # ระบุ type ชัดเจน
local var="value"           # ใช้ local ใน function
"${variable:-default}"      # ให้ default value เสมอ
```

---

## 2.14 Exercises

### Exercise 1: Basic Variables
```bash
#!/bin/bash
# สร้างตัวแปรต่อไปนี้และแสดงผล:
# - ชื่อเต็ม (first_name + last_name)
# - อายุ + 1 ปี
# - แปลง text เป็น uppercase
# - แสดง 5 ตัวอักษรจากกลาง string

first_name="John"
last_name="Doe"
age=25
text="Hello World"

# TODO: เขียนโค้ดต่อ
```

### Exercise 2: String Manipulation
```bash
# ให้ file path: /home/user/projects/myapp/src/main.py
# แสดง: directory, filename, extension, basename without extension

filepath="/home/user/projects/myapp/src/main.py"
# TODO: ใช้ parameter expansion ให้ได้ผลลัพธ์ที่ต้องการ
```

### Exercise 3: Calculator
สร้าง script ที่รับ 2 ตัวเลขและ operator เป็น arguments แล้วคำนวณผลลัพธ์

### Exercise 4: Environment Info
สร้าง script แสดงข้อมูล environment ที่สำคัญทั้งหมดในรูปแบบ table

---

## 2.15 สรุป Part 02

✅ การประกาศและใช้งาน variables  
✅ Variable attributes (declare)  
✅ Special variables ($0, $#, $@, $?, $$)  
✅ Parameter expansion (default, substring, replace)  
✅ Integer arithmetic ((()))  
✅ Float arithmetic (bc, awk)  
✅ String operations  
✅ Environment variables  
✅ Variable scoping  

---

**→ Part 03: String Manipulation**
