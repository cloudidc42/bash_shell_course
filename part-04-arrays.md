# Part 04: Arrays & Associative Arrays
## หลักสูตร Bash/Shell Script ระดับมืออาชีพ

---

## 4.1 Indexed Arrays

```bash
#!/bin/bash

# การสร้าง Array
fruits=("apple" "banana" "cherry" "date")
numbers=(1 2 3 4 5)
mixed=("hello" 42 3.14 "world")

# สร้างแบบทีละ element
arr[0]="first"
arr[1]="second"
arr[2]="third"

# declare -a
declare -a my_array
my_array=("a" "b" "c")

# Array จาก command output
files=($(ls /tmp/*.txt 2>/dev/null))
lines=($(cat file.txt))
# ดีกว่า - ใช้ mapfile/readarray
mapfile -t lines < file.txt
readarray -t words < <(tr ' ' '\n' <<< "hello world foo")

# Access elements
echo "${fruits[0]}"          # apple (index เริ่มที่ 0)
echo "${fruits[1]}"          # banana
echo "${fruits[-1]}"         # date (ตัวสุดท้าย)
echo "${fruits[-2]}"         # cherry (สองตัวสุดท้าย)

# All elements
echo "${fruits[@]}"          # apple banana cherry date
echo "${fruits[*]}"          # apple banana cherry date (as string)

# Length
echo "${#fruits[@]}"         # 4 (จำนวน elements)
echo "${#fruits[0]}"         # 5 (ความยาวของ element แรก)

# All indices
echo "${!fruits[@]}"         # 0 1 2 3
```

---

## 4.2 Array Operations

```bash
fruits=("apple" "banana" "cherry")

# Append
fruits+=("date")
fruits+=("elderberry" "fig")
echo "${fruits[@]}"   # apple banana cherry date elderberry fig

# Prepend (ต้องสร้างใหม่)
fruits=("mango" "${fruits[@]}")

# Insert at position
idx=2
fruits=("${fruits[@]:0:$idx}" "inserted" "${fruits[@]:$idx}")

# Delete element
unset fruits[1]          # ลบ index 1 (เกิด gap!)
echo "${fruits[@]}"      # ยังคงมี gap ที่ index 1

# Delete and re-index
fruits=("${fruits[@]}")  # re-index หลัง unset

# Delete last element
unset fruits[-1]
fruits=("${fruits[@]:0:${#fruits[@]}-1}")

# Delete by value
delete_element() {
    local arr_name=$1
    local val=$2
    local -n arr=$arr_name
    local new_arr=()
    for item in "${arr[@]}"; do
        [[ "$item" != "$val" ]] && new_arr+=("$item")
    done
    arr=("${new_arr[@]}")
}

# Slice
echo "${fruits[@]:1:3}"      # 3 elements starting at index 1
echo "${fruits[@]:2}"        # from index 2 to end

# Copy array
copy=("${fruits[@]}")

# Reverse
reversed_arr=()
for ((i=${#fruits[@]}-1; i>=0; i--)); do
    reversed_arr+=("${fruits[$i]}")
done

# Sort
IFS=$'\n' sorted=($(printf '%s\n' "${fruits[@]}" | sort))
IFS=$'\n' sorted_rev=($(printf '%s\n' "${fruits[@]}" | sort -r))
IFS=$'\n' sorted_num=($(printf '%s\n' "${numbers[@]}" | sort -n))
```

---

## 4.3 Array Iteration

```bash
fruits=("apple" "banana" "cherry" "date")

# Loop ผ่าน elements
for fruit in "${fruits[@]}"; do
    echo "$fruit"
done

# Loop พร้อม index
for i in "${!fruits[@]}"; do
    echo "$i: ${fruits[$i]}"
done

# C-style loop
for ((i=0; i<${#fruits[@]}; i++)); do
    echo "$i: ${fruits[$i]}"
done

# while loop
i=0
while [[ $i -lt ${#fruits[@]} ]]; do
    echo "${fruits[$i]}"
    ((i++))
done

# Loop backward
for ((i=${#fruits[@]}-1; i>=0; i--)); do
    echo "${fruits[$i]}"
done

# Skip certain elements
for fruit in "${fruits[@]}"; do
    [[ "$fruit" == "banana" ]] && continue
    echo "$fruit"
done

# Break on condition
for fruit in "${fruits[@]}"; do
    [[ "$fruit" == "cherry" ]] && break
    echo "$fruit"
done
```

---

## 4.4 Array Searching & Filtering

```bash
fruits=("apple" "banana" "apricot" "cherry" "avocado")

# Check if element exists
contains() {
    local arr_name=$1
    local val=$2
    local -n arr=$arr_name
    for item in "${arr[@]}"; do
        [[ "$item" == "$val" ]] && return 0
    done
    return 1
}

contains fruits "banana" && echo "found" || echo "not found"

# Find index of element
find_index() {
    local arr_name=$1
    local val=$2
    local -n arr=$arr_name
    for i in "${!arr[@]}"; do
        [[ "${arr[$i]}" == "$val" ]] && echo $i && return 0
    done
    return 1
}

idx=$(find_index fruits "cherry")
echo "cherry is at index: $idx"

# Filter array
filter_array() {
    local pattern=$1
    shift
    local arr=("$@")
    local result=()
    for item in "${arr[@]}"; do
        [[ "$item" == $pattern ]] && result+=("$item")
    done
    echo "${result[@]}"
}

# Filter fruits starting with 'a'
starting_a=($(filter_array "a*" "${fruits[@]}"))
echo "Starts with a: ${starting_a[@]}"

# grep-style filter
mapfile -t a_fruits < <(printf '%s\n' "${fruits[@]}" | grep "^a")
echo "Grep filter: ${a_fruits[@]}"

# Count matching elements
count=0
for fruit in "${fruits[@]}"; do
    [[ "$fruit" == a* ]] && ((count++))
done
echo "Count starting with a: $count"
```

---

## 4.5 Array Manipulation (Map, Filter, Reduce)

```bash
numbers=(1 2 3 4 5 6 7 8 9 10)

# Map: transform each element
square_array() {
    local -a result=()
    for n in "$@"; do
        result+=$(( n * n ))
        result+=($((n * n)))
    done
    echo "${result[@]}"
}

mapped=($(for n in "${numbers[@]}"; do echo $((n * n)); done))
echo "Squared: ${mapped[@]}"

# Filter: keep elements matching condition
evens=($(printf '%s\n' "${numbers[@]}" | awk '$1%2==0'))
odds=($(printf '%s\n' "${numbers[@]}" | awk '$1%2!=0'))
echo "Even: ${evens[@]}"
echo "Odd:  ${odds[@]}"

# Reduce: fold array to single value
sum=0
for n in "${numbers[@]}"; do ((sum += n)); done
echo "Sum: $sum"

product=1
for n in "${numbers[@]}"; do ((product *= n)); done
echo "Product: $product"

max=${numbers[0]}
for n in "${numbers[@]}"; do
    ((n > max)) && max=$n
done
echo "Max: $max"

min=${numbers[0]}
for n in "${numbers[@]}"; do
    ((n < min)) && min=$n
done
echo "Min: $min"

# Average
avg=$(echo "scale=2; $sum / ${#numbers[@]}" | bc)
echo "Average: $avg"
```

---

## 4.6 Associative Arrays (Dictionaries)

Associative arrays ต้องการ **Bash 4.0+** และต้อง `declare -A`

```bash
#!/bin/bash

# ประกาศ associative array
declare -A person
person[name]="Alice"
person[age]=30
person[city]="Bangkok"
person[email]="alice@example.com"

# สร้างพร้อมค่า (Bash 4+)
declare -A colors=(
    [red]="#FF0000"
    [green]="#00FF00"
    [blue]="#0000FF"
    [white]="#FFFFFF"
)

# Access
echo "${person[name]}"         # Alice
echo "${colors[red]}"          # #FF0000

# ทุก keys
echo "${!person[@]}"           # name age city email (unordered)

# ทุก values
echo "${person[@]}"            # Alice 30 Bangkok alice@example.com

# จำนวน entries
echo "${#person[@]}"           # 4

# Check if key exists
if [[ -v person[email] ]]; then
    echo "email key exists"
fi

if [[ -n "${person[email]+x}" ]]; then
    echo "email is set"
fi

# Default value
echo "${person[phone]:-'no phone'}"

# Delete key
unset person[email]
echo "${!person[@]}"           # name age city

# Iterate key-value pairs
for key in "${!person[@]}"; do
    echo "$key = ${person[$key]}"
done

# Sorted keys
for key in $(printf '%s\n' "${!colors[@]}" | sort); do
    echo "$key: ${colors[$key]}"
done
```

---

## 4.7 Practical Associative Array Use Cases

```bash
# Counter/Frequency map
text="the quick brown fox jumps over the lazy dog the"
declare -A word_count

for word in $text; do
    ((word_count[$word]++))
done

echo "Word frequencies:"
for word in "${!word_count[@]}"; do
    echo "  '$word': ${word_count[$word]}"
done | sort -t: -k2 -rn

# Configuration store
declare -A config=(
    [db_host]="localhost"
    [db_port]="5432"
    [db_name]="myapp"
    [log_level]="INFO"
    [max_connections]="100"
)

get_config() { echo "${config[$1]:-${2:-}}"; }
set_config() { config[$1]="$2"; }
has_config() { [[ -v config[$1] ]]; }

echo "DB: $(get_config db_host):$(get_config db_port)/$(get_config db_name)"

# Cache / Memoization
declare -A cache

fibonacci() {
    local n=$1
    if [[ -v cache[$n] ]]; then
        echo "${cache[$n]}"
        return
    fi
    if ((n <= 1)); then
        cache[$n]=$n
    else
        local a=$(fibonacci $((n-1)))
        local b=$(fibonacci $((n-2)))
        cache[$n]=$((a + b))
    fi
    echo "${cache[$n]}"
}

for i in {0..20}; do
    printf "fib(%2d) = %d\n" $i $(fibonacci $i)
done

# HTTP Headers store
declare -A headers=(
    [Content-Type]="application/json"
    [Authorization]="Bearer token123"
    [X-Request-ID]="abc-123"
)

build_curl_cmd() {
    local url=$1
    local cmd="curl -s"
    for header in "${!headers[@]}"; do
        cmd+=" -H '${header}: ${headers[$header]}'"
    done
    cmd+=" '$url'"
    echo "$cmd"
}

echo "$(build_curl_cmd 'https://api.example.com/users')"
```

---

## 4.8 Multi-dimensional Arrays (Simulated)

Bash ไม่มี multi-dimensional arrays โดยตรง แต่ simulate ได้:

```bash
# Method 1: Naming convention
declare -A matrix

matrix[0,0]=1; matrix[0,1]=2; matrix[0,2]=3
matrix[1,0]=4; matrix[1,1]=5; matrix[1,2]=6
matrix[2,0]=7; matrix[2,1]=8; matrix[2,2]=9

# Print matrix
rows=3
cols=3
for ((i=0; i<rows; i++)); do
    for ((j=0; j<cols; j++)); do
        printf "%3d" "${matrix[$i,$j]}"
    done
    echo
done

# Method 2: Flat array with index calculation
rows=3
cols=3
flat_matrix=(1 2 3 4 5 6 7 8 9)

get_element() {
    local row=$1 col=$2
    echo "${flat_matrix[$((row * cols + col))]}"
}

set_element() {
    local row=$1 col=$2 val=$3
    flat_matrix[$((row * cols + col))]=$val
}

echo "Element [1][2]: $(get_element 1 2)"   # 6

# Method 3: Array of strings
row0="1 2 3"
row1="4 5 6"
row2="7 8 9"

get_row_col() {
    local row_var="row${1}"
    local col=$2
    local row="${!row_var}"
    echo $row | awk "{print \$$((col + 1))}"
}

echo "Element [2][1]: $(get_row_col 2 1)"   # 8

# Sparse matrix (associative array)
declare -A sparse
sparse[0,0]=1
sparse[1,2]=5
sparse[3,3]=9
# ประหยัด memory สำหรับ sparse data
```

---

## 4.9 Array of Structs (Simulated)

```bash
# Method 1: Parallel arrays
names=("Alice" "Bob" "Charlie")
ages=(30 25 35)
emails=("alice@ex.com" "bob@ex.com" "charlie@ex.com")

# Access
for i in "${!names[@]}"; do
    echo "Name: ${names[$i]}, Age: ${ages[$i]}, Email: ${emails[$i]}"
done

# Method 2: Encoded string
# Format: name:age:email
users=(
    "Alice:30:alice@example.com"
    "Bob:25:bob@example.com"
    "Charlie:35:charlie@example.com"
)

for user in "${users[@]}"; do
    IFS=':' read -r name age email <<< "$user"
    echo "Name: $name, Age: $age, Email: $email"
done

# Method 3: Prefix naming (verbose แต่ clear)
declare -A user1=(
    [name]="Alice"
    [age]=30
    [email]="alice@example.com"
)
declare -A user2=(
    [name]="Bob"
    [age]=25
    [email]="bob@example.com"
)

# Array of user variable names
users_list=("user1" "user2")

for user_var in "${users_list[@]}"; do
    declare -n user=$user_var
    echo "User: ${user[name]}, ${user[age]}, ${user[email]}"
done
```

---

## 4.10 Sorting Arrays

```bash
numbers=(42 17 8 99 3 55 21)
fruits=("banana" "apple" "cherry" "date" "avocado")

# Sort strings alphabetically
IFS=$'\n' sorted_fruits=($(printf '%s\n' "${fruits[@]}" | sort))
echo "Sorted: ${sorted_fruits[@]}"

# Sort reverse
IFS=$'\n' sorted_rev=($(printf '%s\n' "${fruits[@]}" | sort -r))
echo "Reversed: ${sorted_rev[@]}"

# Sort numbers
IFS=$'\n' sorted_nums=($(printf '%s\n' "${numbers[@]}" | sort -n))
echo "Sorted nums: ${sorted_nums[@]}"

# Sort unique
dupes=("apple" "banana" "apple" "cherry" "banana")
IFS=$'\n' unique_sorted=($(printf '%s\n' "${dupes[@]}" | sort -u))
echo "Unique sorted: ${unique_sorted[@]}"

# Custom sort: by string length
IFS=$'\n' by_length=($(
    printf '%s\n' "${fruits[@]}" | \
    awk '{print length, $0}' | \
    sort -n | \
    awk '{print $2}'
))
echo "By length: ${by_length[@]}"

# Sort parallel arrays together
sort_parallel() {
    local -n keys=$1
    local -n vals=$2
    local n=${#keys[@]}
    
    # Create temporary combined array
    local combined=()
    for ((i=0; i<n; i++)); do
        combined+=("${keys[$i]}:${vals[$i]}")
    done
    
    # Sort
    IFS=$'\n' sorted=($(printf '%s\n' "${combined[@]}" | sort))
    
    # Split back
    keys=()
    vals=()
    for item in "${sorted[@]}"; do
        keys+=("${item%%:*}")
        vals+=("${item#*:}")
    done
}

names=("Charlie" "Alice" "Bob")
scores=(85 92 78)
sort_parallel names scores
for i in "${!names[@]}"; do
    echo "${names[$i]}: ${scores[$i]}"
done
```

---

## 4.11 Stack & Queue using Arrays

```bash
#!/bin/bash
# stack_queue.sh - Data structures using bash arrays

# ===== STACK (LIFO) =====
declare -a stack=()

push() {
    stack+=("$1")
}

pop() {
    if [[ ${#stack[@]} -eq 0 ]]; then
        echo "Stack is empty" >&2
        return 1
    fi
    local top="${stack[-1]}"
    unset stack[-1]
    echo "$top"
}

peek() {
    [[ ${#stack[@]} -eq 0 ]] && echo "Empty" && return 1
    echo "${stack[-1]}"
}

stack_size() {
    echo "${#stack[@]}"
}

# Stack demo
echo "=== STACK DEMO ==="
push "first"
push "second"
push "third"
echo "Size: $(stack_size)"
echo "Peek: $(peek)"
echo "Pop: $(pop)"
echo "Pop: $(pop)"
echo "Size: $(stack_size)"

# ===== QUEUE (FIFO) =====
declare -a queue=()

enqueue() {
    queue+=("$1")
}

dequeue() {
    if [[ ${#queue[@]} -eq 0 ]]; then
        echo "Queue is empty" >&2
        return 1
    fi
    local front="${queue[0]}"
    queue=("${queue[@]:1}")
    echo "$front"
}

front() {
    [[ ${#queue[@]} -eq 0 ]] && echo "Empty" && return 1
    echo "${queue[0]}"
}

queue_size() {
    echo "${#queue[@]}"
}

# Queue demo
echo ""
echo "=== QUEUE DEMO ==="
enqueue "first"
enqueue "second"
enqueue "third"
echo "Size: $(queue_size)"
echo "Front: $(front)"
echo "Dequeue: $(dequeue)"
echo "Dequeue: $(dequeue)"
echo "Size: $(queue_size)"

# ===== PRIORITY QUEUE (simplified) =====
declare -a pq_items=()
declare -a pq_priorities=()

pq_insert() {
    local priority=$1
    local item=$2
    local pos=0
    
    # Find insertion position (sorted by priority)
    for ((i=0; i<${#pq_priorities[@]}; i++)); do
        [[ $priority -lt ${pq_priorities[$i]} ]] && break
        ((pos++))
    done
    
    # Insert at position
    pq_priorities=("${pq_priorities[@]:0:$pos}" "$priority" "${pq_priorities[@]:$pos}")
    pq_items=("${pq_items[@]:0:$pos}" "$item" "${pq_items[@]:$pos}")
}

pq_extract_min() {
    [[ ${#pq_items[@]} -eq 0 ]] && echo "Empty" && return 1
    echo "${pq_items[0]}"
    pq_items=("${pq_items[@]:1}")
    pq_priorities=("${pq_priorities[@]:1}")
}

echo ""
echo "=== PRIORITY QUEUE DEMO ==="
pq_insert 3 "low priority"
pq_insert 1 "high priority"
pq_insert 2 "medium priority"
pq_insert 1 "also high priority"

while [[ ${#pq_items[@]} -gt 0 ]]; do
    echo "Extract: $(pq_extract_min)"
done
```

---

## 4.12 Arrays ใน Functions

```bash
#!/bin/bash

# Pass array to function
process_array() {
    local -n arr=$1    # nameref (Bash 4.3+)
    echo "Array size: ${#arr[@]}"
    for item in "${arr[@]}"; do
        echo "  - $item"
    done
}

fruits=("apple" "banana" "cherry")
process_array fruits

# Return array from function
get_filtered() {
    local pattern=$1
    local -n input=$2
    local -n output=$3
    
    output=()
    for item in "${input[@]}"; do
        [[ "$item" == *"$pattern"* ]] && output+=("$item")
    done
}

items=("apple" "apricot" "banana" "avocado" "cherry")
declare -a filtered
get_filtered "a" items filtered
echo "Filtered: ${filtered[@]}"

# Modify array in function (nameref)
add_prefix() {
    local prefix=$1
    local -n arr=$2
    
    for i in "${!arr[@]}"; do
        arr[$i]="${prefix}${arr[$i]}"
    done
}

words=("cat" "dog" "fish")
add_prefix "my_" words
echo "Prefixed: ${words[@]}"

# Function returning associative array
load_user() {
    local id=$1
    local -n result=$2
    
    case $id in
        1) result=([name]="Alice" [age]=30 [role]="admin") ;;
        2) result=([name]="Bob"   [age]=25 [role]="user")  ;;
        *) return 1 ;;
    esac
}

declare -A user
load_user 1 user
echo "User: ${user[name]}, ${user[age]}, ${user[role]}"
```

---

## 4.13 Array Utilities Script

```bash
#!/bin/bash
# array_utils.sh - Comprehensive array utility functions

# Print array as formatted table
print_array_table() {
    local -n arr=$1
    local title=${2:-"Array"}
    local col_width=20
    
    echo "┌─────────┬$(printf '─%.0s' $(seq 1 $col_width))┐"
    printf "│ %-7s │ %-${col_width}s│\n" "Index" "$title"
    echo "├─────────┼$(printf '─%.0s' $(seq 1 $col_width))┤"
    for i in "${!arr[@]}"; do
        printf "│ %-7s │ %-${col_width}s│\n" "$i" "${arr[$i]}"
    done
    echo "└─────────┴$(printf '─%.0s' $(seq 1 $col_width))┘"
}

# Flatten nested array representation
join_with() {
    local sep=$1
    local -n arr=$2
    local result=""
    for i in "${!arr[@]}"; do
        [[ $i -gt 0 ]] && result+="$sep"
        result+="${arr[$i]}"
    done
    echo "$result"
}

# Intersection of two arrays
array_intersect() {
    local -n arr1=$1
    local -n arr2=$2
    local -n result=$3
    
    result=()
    for item in "${arr1[@]}"; do
        for item2 in "${arr2[@]}"; do
            [[ "$item" == "$item2" ]] && result+=("$item") && break
        done
    done
}

# Union of two arrays (unique)
array_union() {
    local -n arr1=$1
    local -n arr2=$2
    local -n result=$3
    
    declare -A seen
    result=()
    for item in "${arr1[@]}" "${arr2[@]}"; do
        if [[ ! -v seen[$item] ]]; then
            result+=("$item")
            seen[$item]=1
        fi
    done
}

# Difference: elements in arr1 but not arr2
array_diff() {
    local -n arr1=$1
    local -n arr2=$2
    local -n result=$3
    
    result=()
    for item in "${arr1[@]}"; do
        local found=0
        for item2 in "${arr2[@]}"; do
            [[ "$item" == "$item2" ]] && found=1 && break
        done
        [[ $found -eq 0 ]] && result+=("$item")
    done
}

# Shuffle array (Fisher-Yates)
array_shuffle() {
    local -n arr=$1
    local n=${#arr[@]}
    
    for ((i=n-1; i>0; i--)); do
        local j=$((RANDOM % (i+1)))
        local temp="${arr[$i]}"
        arr[$i]="${arr[$j]}"
        arr[$j]="$temp"
    done
}

# Remove duplicates preserving order
array_unique() {
    local -n input=$1
    local -n output=$2
    
    declare -A seen
    output=()
    for item in "${input[@]}"; do
        if [[ ! -v seen[$item] ]]; then
            output+=("$item")
            seen[$item]=1
        fi
    done
}

# Demo
echo "=== Array Utilities Demo ==="

a=("apple" "banana" "cherry" "date")
b=("banana" "date" "elderberry" "fig")

declare -a inter union diff unique shuffled

array_intersect a b inter
echo "Intersection: $(join_with ", " inter)"

array_union a b union
echo "Union: $(join_with ", " union)"

array_diff a b diff
echo "A - B: $(join_with ", " diff)"

nums=(1 2 3 2 4 1 5 3)
declare -a uniq
array_unique nums uniq
echo "Unique: $(join_with ", " uniq)"

deck=(A K Q J 10 9 8 7 6 5 4 3 2)
array_shuffle deck
echo "Shuffled deck: ${deck[@]}"

print_array_table a "Fruits"
```

---

## 4.14 Exercises

### Exercise 1: Array Operations
```bash
# สร้าง array ของตัวเลข 1-20
# แสดง: sum, average, max, min
# กรองเฉพาะ prime numbers
```

### Exercise 2: Associative Array
สร้าง grade book ด้วย associative array:
- เก็บชื่อนักเรียนและคะแนน
- คำนวณค่าเฉลี่ย
- แสดง grade (A=90+, B=80+, C=70+, D=60+, F=<60)

### Exercise 3: Contact Book
สร้าง script บันทึก contacts ด้วย associative arrays:
- เพิ่ม contact
- ค้นหาชื่อ
- ลบ contact
- แสดงทั้งหมด

### Exercise 4: Matrix Operations
สร้างฟังก์ชัน:
- Matrix addition
- Matrix multiplication (3x3)
- Transpose

---

## สรุป Part 04

✅ Indexed arrays (สร้าง, เข้าถึง, แก้ไข)  
✅ Array operations (append, delete, slice, sort)  
✅ Array iteration (for, while, with index)  
✅ Searching & filtering  
✅ Functional operations (map, filter, reduce)  
✅ Associative arrays (declare -A)  
✅ Multi-dimensional arrays (simulated)  
✅ Stack & Queue  
✅ Arrays ใน functions (nameref)  
✅ Utility functions  

---

**→ Part 05: Conditional Statements**
