# 🐚 Tài liệu Bash Scripting nâng cao: Từ Zero đến Chuyên gia

> **Mục tiêu**: Viết Bash script production-ready: an toàn, hiệu quả, có test, có thể debug.

---

## 📑 Mục lục

| Phần | Nội dung |
|------|----------|
| [P0. Nền tảng](#p0) | Bash setup, options |
| [P1. Biến & Arrays](#p1) | String, array, map |
| [P2. Control flow](#p2) | if, for, while, case |
| [P3. Functions](#p3) | Định nghĩa, arguments, scope |
| [P4. Text processing](#p4) | grep, awk, sed, regex |
| [P5. File I/O](#p5) | Read, write, redirection |
| [P6. Process & Signal](#p6) | Jobs, signals, trap |
| [P7. Network scripting](#p7) | curl, ssh, socket |
| [P8. Advanced](#p8) | trap, debug, IPC, parallel |
| [P9. Testing](#p9) | bats, shellcheck |
| [P10. Production](#p10) | Best practice, packaging |

---

<a id="p0"></a>
## P0. Nền tảng

### Bước 1: Bash strict mode

```bash
#!/usr/bin/env bash
# Strict mode - LUÔN dùng ở đầu script
set -Eeuo pipefail
# -E: Trap ERR bị kế thừa trong function
# -e: Exit ngay khi command fail
# -u: Lỗi nếu dùng biến undefined
# -o pipefail: Pipe fail nếu bất kỳ command nào fail

# IFS an toàn
IFS=$'\n\t'

# Trap errors
trap 'echo "Error on line $LINENO"; exit 1' ERR
```

### Bước 2: Cú pháp shebang

```bash
#!/usr/bin/env bash    # Tìm bash trong PATH (khuyến nghị)
#!/bin/bash            # Đường dẫn tuyệt đối
#!/bin/sh              # POSIX sh (không có bash features)
#!/usr/bin/env -S bash    # Với args
#!/usr/bin/env -S bash -e    # Tự động strict mode
```

### Bước 3: Debug

```bash
# Debug full
bash -x script.sh

# Bật debug trong script
set -x        # Print commands
set +x        # Tắt
set -v        # Print input

# Debug chỉ 1 đoạn
set -x
complex_command
set +x

# Debug với PS4 (prompt)
export PS4='+(${BASH_SOURCE}:${LINENO}): ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'
set -x
```

---

<a id="p1"></a>
## P1. Biến & Arrays

### Bước 1: Biến cơ bản

```bash
# Khai báo
name="World"
readonly PI=3.14
declare -r VERSION="1.0"

# Sử dụng
echo "Hello, $name!"
echo "Hello, ${name}!"

# Phạm vi
local_var="only in function"  # Local nếu trong function
global_var="everywhere"

# Special variables
$0              # Script name
$1, $2, ...     # Arguments
$#              # Số arguments
$@              # Tất cả args (mảng)
$*              # Tất cả args (chuỗi)
$$              # PID hiện tại
$?              # Exit code của lệnh trước
$!              # PID của background job gần nhất
$RANDOM         # Random number
$LINENO         # Line hiện tại
$SECONDS        # Số giây từ khi script chạy
$BASH_SOURCE    # Source file
$FUNCNAME       # Tên function hiện tại
$BASHPID        # PID của bash process
$HOSTNAME       # Hostname
$USER           # Username
$HOME           # Home directory
$PWD            # Current directory
$PATH           # PATH env var
```

### Bước 2: String

```bash
str="Hello World"

# Length
${#str}                  # 11

# Substring
${str:0:5}               # Hello
${str:6}                 # World
${str:(-5)}              # World

# Replace
${str//o/0}              # Hell0 W0rld (all)
${str/o/0}               # Hell0 World (first)

# Remove pattern
${str#Hello}             # " World" (prefix)
${str##Hello*}           # "" (longest prefix)
${str%World}             # "Hello " (suffix)
${str%%World*}           # "" (longest suffix)

# Case
${str^}                  # Hello World -> Hello World (uppercase first)
${str^^}                 # HELLO WORLD
${str,}                  # hELLO wORLD (lowercase first)
${str,,}                 # hello world

# Default
${var:-default}          # Dùng default nếu var unset
${var:=default}          # Set default nếu var unset
${var:+alt}              # alt nếu var set
${var:?error}            # Error nếu var unset
${var:offset:length}     # Substring

# Trim
trim() {
    local var="$*"
    var="${var#"${var%%[![:space:]]*}"}"   # Trim leading
    var="${var%"${var##*[![:space:]]}"}"   # Trim trailing
    echo -n "$var"
}

# Multi-line
multi="line1
line2
line3"
echo "$multi"

# Heredoc
cat <<EOF
Multi-line
content with $variables
EOF

# Heredoc không expand (literal)
cat <<'EOF'
$variable sẽ không expand
EOF

# Heredoc với strip tab
cat <<-EOF
    Indented (tabs bị strip)
EOF
```

### Bước 3: Arrays

```bash
# Indexed array
arr=("apple" "banana" "cherry")
arr[3]="date"

echo "${arr[0]}"             # apple
echo "${arr[@]}"             # Tất cả
echo "${arr[*]}"             # Tất cả (một chuỗi)
echo "${#arr[@]}"            # Length
echo "${!arr[@]}"            # Indices

# Slice
echo "${arr[@]:1:2}"         # banana cherry

# Append
arr+=("elderberry")

# Loop
for item in "${arr[@]}"; do
    echo "$item"
done

# Index loop
for i in "${!arr[@]}"; do
    echo "$i: ${arr[$i]}"
done

# Associative array (Bash 4+)
declare -A user
user[name]="John"
user[age]=30
user[email]="[email protected]"

echo "${user[name]}"         # John
echo "${!user[@]}"          # Keys
echo "${user[@]}"           # Values

# Iterate
for key in "${!user[@]}"; do
    echo "$key = ${user[$key]}"
done
```

### Bước 4: Arithmetic

```bash
# Integer math
a=5
b=3
echo $((a + b))             # 8
echo $((a * b))             # 15
echo $((a / b))             # 1
echo $((a % b))             # 2
echo $((a ** 2))            # 25
echo $((a++))               # 5 (rồi a=6)
echo $((++a))               # 7 (rồi a=7)

# Compound assignment
((a += 5))                  # a=10
((a -= 3))                  # a=7

# Comparison
((a > b)) && echo "yes"
((a == b)) || echo "no"

# let
let "a = 5 + 3"
let "a++"

# Float math (dùng bc hoặc awk)
echo "scale=2; 10 / 3" | bc    # 3.33
awk 'BEGIN { printf "%.2f\n", 10/3 }'   # 3.33
```

### Bước 5: Brace expansion

```bash
# Range
echo {1..5}                 # 1 2 3 4 5
echo {a..e}                 # a b c d e
echo {1..10..2}             # 1 3 5 7 9
echo {01..10}               # 01 02 ... 10

# List
echo {a,b,c}                # a b c

# Combine
echo {a,b}{1,2}             # a1 a2 b1 b2
echo /etc/{nginx,apache2}/   # /etc/nginx/ /etc/apache2/

# Padding (zsh only)
# echo {01..10}

# Use case: tạo nhiều files
mkdir -p src/{js,css,img}
touch file_{1..5}.txt

# Backup
cp file.txt{,.bak}          # file.txt.bak
```

---

<a id="p2"></a>
## P2. Control flow

### Bước 1: if/elif/else

```bash
if [[ condition ]]; then
    commands
elif [[ condition2 ]]; then
    commands
else
    commands
fi

# === Test ===
# File
[[ -f file ]]              # File exists, regular
[[ -d dir ]]               # Directory
[[ -e path ]]              # Exists
[[ -r file ]]              # Readable
[[ -w file ]]              # Writable
[[ -x file ]]              # Executable
[[ -s file ]]              # Size > 0
[[ -L link ]]              # Symlink
[[ file1 -nt file2 ]]      # Newer than
[[ file1 -ot file2 ]]      # Older than

# String
[[ -z "$str" ]]            # Empty
[[ -n "$str" ]]            # Not empty
[[ "$a" == "$b" ]]         # Equal
[[ "$a" != "$b" ]]         # Not equal
[[ "$a" < "$b" ]]          # Less than (lexicographic)
[[ "$a" =~ regex ]]        # Regex match
[[ "$a" == *"text"* ]]     # Substring

# Integer
[[ $a -eq $b ]]            # Equal
[[ $a -ne $b ]]            # Not equal
[[ $a -lt $b ]]            # Less than
[[ $a -le $b ]]            # Less or equal
[[ $a -gt $b ]]            # Greater than
[[ $a -ge $b ]]            # Greater or equal

# Logical
[[ $a && $b ]]             # AND
[[ $a || $b ]]             # OR
[[ ! $a ]]                 # NOT

# Group
[[ ( $a -eq 1 || $a -eq 2 ) && $b -gt 10 ]]
```

### Bước 2: for loop

```bash
# List
for fruit in apple banana cherry; do
    echo "$fruit"
done

# Range
for i in {1..10}; do
    echo "$i"
done

# Sequence
for i in $(seq 1 10 2); do  # start, end, step
    echo "$i"
done

# C-style
for ((i = 0; i < 10; i++)); do
    echo "$i"
done

# Array
arr=(a b c d)
for item in "${arr[@]}"; do
    echo "$item"
done

# Files
for file in *.txt; do
    echo "$file"
done

# Find
find . -name "*.log" | while read -r file; do
    echo "Processing $file"
done

# Assoc array
for key in "${!config[@]}"; do
    echo "$key = ${config[$key]}"
done

# Parallel
for i in {1..10}; do
    (
        echo "Task $i"
        sleep 1
    ) &
done
wait
```

### Bước 3: while & until

```bash
# while
count=0
while [[ $count -lt 10 ]]; do
    echo "$count"
    ((count++))
done

# until
count=0
until [[ $count -ge 10 ]]; do
    echo "$count"
    ((count++))
done

# Read file
while IFS= read -r line; do
    echo "$line"
done < file.txt

# Read file với split
while IFS=':' read -r user pass uid gid comment home shell; do
    echo "User: $user, Home: $home"
done < /etc/passwd

# Infinite loop
while true; do
    echo "Press Ctrl+C to stop"
    sleep 1
done

# Menu
while true; do
    echo "1. Start"
    echo "2. Stop"
    echo "3. Exit"
    read -p "Choice: " choice
    case $choice in
        1) echo "Starting..." ;;
        2) echo "Stopping..." ;;
        3) break ;;
        *) echo "Invalid" ;;
    esac
done
```

### Bước 4: case

```bash
case "$1" in
    start)
        echo "Starting..."
        ;;
    stop)
        echo "Stopping..."
        ;;
    restart)
        echo "Restarting..."
        ;;
    status)
        echo "Status..."
        ;;
    *)
        echo "Usage: $0 {start|stop|restart|status}"
        exit 1
        ;;
esac

# Pattern matching
case "$file" in
    *.tar.gz|*.tgz) echo "tar.gz archive" ;;
    *.zip)          echo "zip archive" ;;
    *.jpg|*.jpeg)   echo "JPEG image" ;;
    *.png)          echo "PNG image" ;;
    *)              echo "Unknown" ;;
esac
```

### Bước 5: select

```bash
select option in "Start" "Stop" "Restart" "Exit"; do
    case $option in
        "Start") echo "Starting..." ;;
        "Stop") echo "Stopping..." ;;
        "Restart") echo "Restarting..." ;;
        "Exit") break ;;
        *) echo "Invalid option" ;;
    esac
done
```

---

<a id="p3"></a>
## P3. Functions

### Bước 1: Function cơ bản

```bash
# Định nghĩa
greet() {
    local name="${1:-World}"
    echo "Hello, $name!"
}

# Gọi
greet
greet "Alice"

# Định nghĩa khác (cũ, ít dùng)
function greet() {
    echo "Hello!"
}
```

### Bước 2: Arguments & return

```bash
my_func() {
    echo "Args: $@"
    echo "Count: $#"
    echo "First: $1"
    echo "All as array: ${@}"
}

# Return value
add() {
    local a=$1
    local b=$2
    echo $((a + b))        # Echo để return
}

result=$(add 5 3)
echo "5 + 3 = $result"

# Exit code
is_root() {
    [[ $EUID -eq 0 ]]
}

if is_root; then
    echo "Running as root"
fi

# Return code + capture output
sum() {
    local total=0
    for n in "$@"; do
        ((total += n))
    done
    echo "$total"
    return 0
}

total=$(sum 1 2 3 4 5)
```

### Bước 3: Variable scope

```bash
global_var="global"

func() {
    local local_var="local"
    global_var="modified"     # Có thể modify global

    echo "Inside: $local_var, $global_var"
}

func
echo "Outside: $global_var"    # modified

# Export để subprocess nhìn thấy
export MY_VAR="visible to children"

# Declare options
declare -r PI=3.14             # Readonly
declare -i count=10            # Integer
declare -a arr=("a" "b")      # Array
declare -A map                 # Associative array
declare -x EXPORTED="value"   # Export
declare -u upper="abc"         # Uppercase (chỉ assign)
declare -l lower="ABC"        # Lowercase

# Nameref (Bash 4.3+)
func() {
    local -n ref=$1
    ref="new value"
}

my_var="old"
func my_var
echo "$my_var"                  # new value
```

### Bước 4: Advanced functions

```bash
# Recursive
factorial() {
    local n=$1
    if [[ $n -le 1 ]]; then
        echo 1
    else
        local prev
        prev=$(factorial $((n - 1)))
        echo $((n * prev))
    fi
}

# Function library
# lib/logging.sh
log() {
    local level=$1
    shift
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $*" >&2
}
info() { log INFO "$@"; }
warn() { log WARN "$@"; }
error() { log ERROR "$@"; }
debug() {
    [[ ${DEBUG:-0} -eq 1 ]] && log DEBUG "$@"
}

# Source
source lib/logging.sh
info "Application started"
debug "Detailed info"

# Function trả về array
get_users() {
    printf '%s\n' alice bob charlie
}

mapfile -t users < <(get_users)
echo "${users[@]}"

# Callback
map_file() {
    local -n callback=$1
    while IFS= read -r line; do
        callback "$line"
    done
}

process() {
    echo "Processing: $1"
}

map_file process < file.txt
```

---

<a id="p4"></a>
## P4. Text processing

### Bước 1: grep

```bash
# Basic
grep "pattern" file.txt
grep -i "pattern" file.txt          # Case insensitive
grep -v "pattern" file.txt          # Invert (không match)
grep -c "pattern" file.txt          # Count
grep -l "pattern" *.txt             # Files chứa pattern
grep -n "pattern" file.txt          # Show line number
grep -r "pattern" directory/        # Recursive
grep -E "regex" file.txt            # Extended regex
grep -P "perl-regex" file.txt       # Perl regex
grep -F "literal" file.txt          # Fixed string (no regex)

# Patterns
grep "^start" file.txt              # Bắt đầu dòng
grep "end$" file.txt                # Cuối dòng
grep "^$" file.txt                  # Dòng rỗng
grep "[0-9]" file.txt               # Chứa số
grep "[a-zA-Z]" file.txt            # Chứa chữ cái
grep "a\|b" file.txt                # a hoặc b
grep -E "(ab)+" file.txt            # ab lặp lại

# Multiple files
grep "pattern" *.log
grep "pattern" file1 file2 file3
zgrep "pattern" file.gz              # Trong file gzip
grep "pattern" <(command)            # Từ process substitution

# Context
grep -A 3 "pattern" file.txt         # 3 dòng sau
grep -B 3 "pattern" file.txt         # 3 dòng trước
grep -C 3 "pattern" file.txt         # 3 dòng trước và sau
```

### Bước 2: sed

```bash
# Replace
sed 's/old/new/' file.txt            # First per line
sed 's/old/new/g' file.txt           # All occurrences
sed 's/old/new/2' file.txt           # 2nd per line

# In-place edit
sed -i 's/old/new/g' file.txt
sed -i.bak 's/old/new/g' file.txt    # Backup trước

# Delete
sed '/pattern/d' file.txt            # Xoá dòng match
sed '5d' file.txt                    # Xoá dòng 5
sed '5,10d' file.txt                 # Xoá 5-10
sed '$d' file.txt                    # Xoá dòng cuối
sed '/^$/d' file.txt                 # Xoá dòng rỗng

# Print
sed -n '5p' file.txt                 # In dòng 5
sed -n '5,10p' file.txt              # In 5-10
sed -n '/pattern/p' file.txt         # In dòng match (giống grep)

# Insert/Append
sed '5i\New line before 5' file.txt
sed '5a\New line after 5' file.txt

# Multiple commands
sed -e 's/a/b/' -e 's/c/d/' file.txt
sed 's/a/b/; s/c/d/' file.txt

# Group
sed 's/\(foo\)\(bar\)/\2\1/' file.txt   # Swap

# Address
sed '1,5s/old/new/' file.txt         # 1-5
sed '/start/,/end/s/old/new/' file.txt  # Range

# Transform
sed 'y/abc/xyz/' file.txt            # Replace chars
sed 'y/abcdefghijklmnopqrstuvwxyz/ABCDEFGHIJKLMNOPQRSTUVWXYZ/'  # Uppercase
```

### Bước 3: awk

```bash
# Print columns
awk '{print $1}' file.txt             # Cột 1
awk '{print $1, $3}' file.txt
awk '{print NF}' file.txt             # Số cột
awk '{print NR}' file.txt             # Line number
awk '{print $NF}' file.txt            # Cột cuối
awk '{print $0}' file.txt             # Toàn dòng

# Pattern
awk '/pattern/' file.txt             # Lines match
awk '!/pattern/' file.txt            # Lines không match
awk '$1 == "value"' file.txt         # Cột 1 = value
awk '$1 ~ /pattern/' file.txt        # Cột 1 match regex
awk '$1 > 100' file.txt              # Cột 1 > 100

# Field separator
awk -F: '{print $1}' /etc/passwd
awk -F',' '{print $1}' file.csv
awk -F'\t' '{print $1}' file.tsv

# BEGIN/END
awk 'BEGIN {print "Start"} {print $0} END {print "End"}' file.txt

# Sum
awk '{sum += $1} END {print sum}' file.txt
awk '{sum += $1; count++} END {print sum/count}' file.txt    # Average

# Variables
awk -v threshold=100 '$1 > threshold {print}' file.txt

# Functions
awk '{print toupper($0)}' file.txt           # Uppercase
awk '{print length($0)}' file.txt            # Length
awk 'BEGIN {print strftime("%Y-%m-%d")}'

# Complex
awk -F: '{
    if ($3 > 1000) {
        printf "User %s has UID %d\n", $1, $3
    }
}' /etc/passwd

# Count occurrences
awk '{count[$1]++} END {for (k in count) print k, count[k]}' file.txt

# Group by
awk '{sum[$1] += $2; count[$1]++} END {
    for (k in sum) print k, sum[k]/count[k]
}' file.txt
```

### Bước 4: Other tools

```bash
# cut
cut -d',' -f1,3 file.csv
cut -c1-10 file.txt                  # Ký tự 1-10
cut -d':' -f1 /etc/passwd

# sort
sort file.txt
sort -r file.txt                     # Reverse
sort -n file.txt                     # Numeric
sort -u file.txt                     # Unique
sort -k2 file.txt                    # By column 2
sort -t',' -k2 -n file.csv

# uniq
uniq file.txt
sort file.txt | uniq -c              # Count
sort file.txt | uniq -d              # Duplicate

# tr
echo "Hello" | tr 'a-z' 'A-Z'        # Uppercase
echo "Hello 123" | tr -d '0-9'       # Delete digits
echo "Hello" | tr ' ' '\t'           # Replace space -> tab

# paste
paste file1 file2                    # Merge side by side
paste -d',' file1 file2

# column
cat file.txt | column -t             # Pretty table

# xxd (hex)
xxd file.bin | head

# jq (JSON)
echo '{"name":"John","age":30}' | jq .
echo '[1,2,3]' | jq '.[0]'
cat data.json | jq '.users[] | .name'

# yq (YAML)
yq '.spec.replicas' deployment.yaml

# csvkit
csvcut -c name,age data.csv
csvstat data.csv
```

---

<a id="p5"></a>
## P5. File I/O

### Bước 1: Read & Write

```bash
# Read
echo "Enter name:"
read name
read -p "Enter name: " name
read -p "Password: " -s pass      # Silent
read -t 10 -p "Quick: " input     # Timeout
read -n 1 -p "Press any key: "    # 1 char

# Read từ file
content=$(<file.txt)
mapfile -t lines < file.txt       # Read vào array
IFS=$'\n' read -d '' -ra lines < file.txt  # Multi-line

# Read line by line
while IFS= read -r line; do
    echo "$line"
done < file.txt

# Write
echo "Hello" > file.txt            # Overwrite
echo "Hello" >> file.txt           # Append
printf "%s\n" "$var" > file.txt

# Multi-line
cat > file.txt <<EOF
Line 1
Line 2 with $var
EOF
```

### Bước 2: File operations

```bash
# File info
[[ -f file ]] && echo "regular file"
[[ -d dir ]] && echo "directory"
[[ -e path ]] && echo "exists"
[[ -s file ]] && echo "non-empty"
[[ -r file ]] && echo "readable"
[[ -w file ]] && echo "writable"
[[ -x file ]] && echo "executable"
[[ -L link ]] && echo "symlink"

# Stat
stat file.txt
stat -c "%s %y" file.txt          # Size, mtime

# Find
find . -name "*.txt"
find . -type f -size +1M
find . -mtime -7                   # 7 days
find . -newer file.txt

# Copy/Move
cp src dest
cp -r dir/ dest/
mv src dest
ln -s target link                 # Symlink

# Lock file
exec 200>/var/lock/myapp.lock
flock -n 200 || { echo "Already running"; exit 1; }
# ... do work ...
flock -u 200
```

### Bước 3: Temporary files

```bash
# Secure temp file
tmpfile=$(mktemp)
trap "rm -f '$tmpfile'" EXIT

# Hoặc directory
tmpdir=$(mktemp -d)
trap "rm -rf '$tmpdir'" EXIT
```

### Bước 4: Heredoc & Herestring

```bash
# Heredoc
cat <<EOF
Text with $variable expansion
EOF

cat <<'EOF'
Literal $variable
EOF

cat <<-EOF
    Indented (tabs stripped)
EOF

# Process substitution
diff <(ls dir1) <(ls dir2)
while read line; do
    echo "$line"
done < <(command)

# Herestring (Bash)
grep "pattern" <<< "$variable"
```

---

<a id="p6"></a>
## P6. Process & Signal

### Bước 1: Job control

```bash
# Background
command &
sleep 60 &
bg_pid=$!

# Wait
wait                       # All background jobs
wait $bg_pid               # Specific job

# Jobs
jobs                       # List background jobs
fg %1                      # Bring job 1 to foreground
bg %1                      # Resume in background

# Disown (chạy sau khi shell exit)
nohup long_command &
disown
```

### Bước 2: Signals

```bash
# Send signal
kill PID
kill -9 PID                # SIGKILL
kill -SIGTERM PID
kill -HUP PID              # Reload (Nginx, etc.)
kill -USR1 PID

# Trap signals
cleanup() {
    echo "Cleaning up..."
    rm -f /tmp/myapp.lock
    exit 1
}

trap cleanup EXIT
trap cleanup INT TERM ERR

# Trap specific signal
trap 'echo "Got SIGHUP"; reload_config' HUP

# Ignore
trap '' INT                # Ignore Ctrl+C
```

### Bước 3: Daemon script

```bash
#!/usr/bin/env bash
# Daemon template
set -Eeuo pipefail

LOCKFILE=/var/run/myapp.pid
LOGFILE=/var/log/myapp.log

start() {
    if [[ -f $LOCKFILE ]] && kill -0 "$(cat $LOCKFILE)" 2>/dev/null; then
        echo "Already running (PID $(cat $LOCKFILE))"
        return 1
    fi

    echo "Starting..."
    nohup ./myapp >> "$LOGFILE" 2>&1 &
    echo $! > "$LOCKFILE"
    echo "Started (PID $(cat $LOCKFILE))"
}

stop() {
    if [[ ! -f $LOCKFILE ]]; then
        echo "Not running"
        return 1
    fi

    local pid
    pid=$(cat "$LOCKFILE")
    echo "Stopping (PID $pid)..."
    kill "$pid"
    rm -f "$LOCKFILE"
}

status() {
    if [[ -f $LOCKFILE ]] && kill -0 "$(cat $LOCKFILE)" 2>/dev/null; then
        echo "Running (PID $(cat $LOCKFILE))"
    else
        echo "Not running"
        [[ -f $LOCKFILE ]] && rm -f "$LOCKFILE"
    fi
}

case "${1:-}" in
    start)   start ;;
    stop)    stop ;;
    restart) stop; start ;;
    status)  status ;;
    *)       echo "Usage: $0 {start|stop|restart|status}"; exit 1 ;;
esac
```

### Bước 4: Parallel execution

```bash
# GNU Parallel (cần cài)
brew install parallel
parallel -j 4 "process {}" ::: file1 file2 file3 file4

# xargs parallel
ls *.txt | xargs -P 4 -I {} sh -c 'echo "Processing {}"; sleep 1'

# Bash background
for file in *.txt; do
    process_file "$file" &
done
wait

# Giới hạn parallel
max_jobs=4
job_count=0
for file in *.txt; do
    process_file "$file" &
    ((job_count++))
    if ((job_count >= max_jobs)); then
        wait -n            # Wait for 1 job
        ((job_count--))
    fi
done
wait
```

---

<a id="p7"></a>
## P7. Network scripting

### Bước 1: curl

```bash
# GET
curl https://api.example.com/users
curl -s https://api.example.com      # Silent
curl -I https://example.com          # Headers only

# POST
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -d '{"name":"John","age":30}'

# Auth
curl -u user:pass https://api.example.com
curl -H "Authorization: Bearer $TOKEN" https://api.example.com

# Follow redirect
curl -L https://example.com

# Save to file
curl -o file.json https://api.example.com
curl -O https://example.com/file.zip    # Tên gốc

# Headers
curl -H "X-Custom: value" https://api.example.com

# Cookies
curl -c cookies.txt -b cookies.txt https://example.com

# POST file
curl -F "file=@/path/to/file" https://api.example.com/upload

# Verify SSL
curl --cacert /path/to/ca.crt https://example.com
curl -k https://self-signed.example.com   # Insecure

# Timeout
curl --max-time 30 --connect-timeout 5 https://example.com

# Retry
curl --retry 3 --retry-delay 2 https://example.com

# Response info
curl -w "%{http_code} %{time_total}\n" -s -o /dev/null https://example.com

# Multiple URLs
curl -O https://example.com/a.zip -O https://example.com/b.zip
```

### Bước 2: SSH scripting

```bash
# Single command
ssh user@server "uptime"

# Multiple commands
ssh user@server "cd /opt && ls -la"

# Local script chạy remote
ssh user@server 'bash -s' < local_script.sh

# SSH config
cat ~/.ssh/config
Host myserver
    HostName 192.168.1.10
    User deploy
    IdentityFile ~/.ssh/id_rsa
    Port 22

# rsync
rsync -avz --progress src/ user@server:/dest/
rsync -avz --delete src/ user@server:/dest/
rsync -avz -e "ssh -i key.pem" src/ user@server:/dest/

# SCP
scp file.txt user@server:/tmp/
scp -r directory/ user@server:/tmp/

# Loop over servers
for server in web1 web2 web3; do
    echo "=== $server ==="
    ssh "$server" "uptime"
done
```

### Bước 3: Socket & HTTP

```bash
# Raw HTTP
exec 3<>/dev/tcp/example.com/80
echo -e "GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n" >&3
cat <&3
exec 3>&-

# nc - netcat
echo "GET / HTTP/1.0\r\n\r\n" | nc example.com 80
nc -zv host port         # Check port

# Test HTTPS manually
openssl s_client -connect example.com:443 -servername example.com
```

---

<a id="p8"></a>
## P8. Advanced

### Bước 1: Trap & Debug

```bash
# Trap ERR
trap 'echo "Error at line $LINENO"; exit 1' ERR

# Trap với context
trap_handler() {
    local exit_code=$?
    local line_no=$1
    echo "Error $exit_code at line $line_no"
    echo "Last command: $BASH_COMMAND"
    exit $exit_code
}
trap 'trap_handler $LINENO' ERR

# Trap DEBUG (chạy trước mỗi command)
trap 'echo "+ $BASH_COMMAND"' DEBUG

# Trap EXIT (luôn chạy khi exit)
cleanup() {
    echo "Cleaning up..."
    rm -f /tmp/myapp-*
}
trap cleanup EXIT
```

### Bước 2: Strict mode toàn diện

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'

# Trap errors
trap 'echo "Error at line $LINENO: $BASH_COMMAND" >&2; exit 1' ERR

# Trap signals
trap 'echo "Interrupted"; exit 130' INT TERM

# Cleanup on exit
trap 'cleanup' EXIT
cleanup() {
    rm -f "$tmpfile" 2>/dev/null || true
}
tmpfile=$(mktemp)

main() {
    local input="${1:?Usage: $0 FILE}"
    [[ -f "$input" ]] || { echo "File not found: $input" >&2; exit 1; }

    # Process
    while IFS= read -r line; do
        echo "$line"
    done < "$input"
}

main "$@"
```

### Bước 3: IPC

```bash
# Named pipe (FIFO)
mkfifo /tmp/mypipe
exec 3<> /tmp/mypipe

# Process 1
echo "message" > /tmp/mypipe &

# Process 2
read line < /tmp/mypipe
echo "Got: $line"

rm /tmp/mypipe

# Temporary FIFO
fifo=$(mktemp -u)
mkfifo "$fifo"
exec 3<>"$fifo"
# ...
rm -f "$fifo"

# Co-process
coproc myproc {
    while read -r line; do
        echo "Echo: $line"
    done
}

echo "Hello" >&${myproc[1]}
read -r reply <&${myproc[0]}
echo "Reply: $reply"
```

### Bước 4: Bảo mật script

```bash
# Random
random_num=$RANDOM
random_str=$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 16 | head -n 1)

# UUID
uuid=$(cat /proc/sys/kernel/random/uuid)

# Password (openssl)
password=$(openssl rand -base64 16)

# Hash
echo -n "$password" | sha256sum

# Encryption với GPG
echo "secret" | gpg --symmetric --cipher-algo AES256 > secret.gpg
gpg --decrypt secret.gpg
```

### Bước 5: Logging chuyên nghiệp

```bash
# Log levels
log_debug() { [[ ${LOG_LEVEL:-info} =~ debug ]] && echo "[DEBUG] $*" >&2; }
log_info()  { [[ ${LOG_LEVEL:-info} =~ info|debug ]] && echo "[INFO]  $*" >&2; }
log_warn()  { [[ ${LOG_LEVEL:-info} =~ warn|info|debug ]] && echo "[WARN]  $*" >&2; }
log_error() { echo "[ERROR] $*" >&2; }

log_info "Starting..."
log_debug "Variable: $var"

# Timestamp log
log() {
    local level=$1
    shift
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $*" >&2
}

log INFO "Application started"

# Log to syslog
log() {
    logger -t "myapp" -p "user.$1" "${@:2}"
}

log info "Application started"

# Rotation
logfile=/var/log/myapp.log
exec 3>>"$logfile"

# Rotate khi file > 10MB
if [[ $(stat -c%s "$logfile") -gt 10485760 ]]; then
    mv "$logfile" "$logfile.old"
    exec 3>>"$logfile"
fi
```

---

<a id="p9"></a>
## P9. Testing với bats

### Bước 1: Cài bats

```bash
# macOS
brew install bats-core

# Linux
git clone https://github.com/bats-core/bats-core.git
cd bats-core
./install.sh /usr/local

# Plugins
git clone https://github.com/bats-core/bats-support.git /usr/local/lib/bats-support
git clone https://github.com/bats-core/bats-assert.git /usr/local/lib/bats-assert
```

### Bước 2: Test đầu tiên

```bash
# test.bats
#!/usr/bin/env bats

load 'bats-support/load'
load 'bats-assert/load'

# Setup chạy trước mỗi test
setup() {
    source ./lib.sh
}

# Teardown chạy sau mỗi test
teardown() {
    rm -f /tmp/test_output
}

@test "addition works" {
    result=$(add 2 3)
    assert_equal "$result" "5"
}

@test "addition with zero" {
    result=$(add 0 5)
    assert_equal "$result" "5"
}

@test "function exists" {
    run declare -f add
    assert_success
}

@test "greeting" {
    run greet "Alice"
    assert_success
    assert_output "Hello, Alice!"
}

@test "greeting default" {
    run greet
    assert_success
    assert_output "Hello, World!"
}

@test "fails on missing arg" {
    run command_that_requires_arg
    assert_failure
    assert_output --partial "required"
}

@test "file exists after write" {
    echo "test" > /tmp/test_output
    assert_file_exists /tmp/test_output
    assert_file_not_empty /tmp/test_output
}

@test "file contains expected content" {
    echo "hello world" > /tmp/test_output
    assert_file_contains /tmp/test_output "world"
}
```

```bash
bats test.bats
bats test.bats --verbose-run
```

### Bước 3: Mock & stub

```bash
# Sử dụng bats-mock
load 'bats-mock/load'

@test "curl is called" {
    stub curl "echo 'mocked response'"
    run my_function
    unstub curl
    assert_output "mocked response"
}

@test "curl with specific args" {
    stub curl "--data * : echo 'POST data: \$2'"
    run api_call
    unstub curl
}
```

### Bước 4: ShellCheck (lint)

```bash
# Cài
brew install shellcheck
apt install shellcheck

# Check
shellcheck script.sh

# Check tất cả
find . -name "*.sh" -exec shellcheck {} \;

# Disable rule
# shellcheck disable=SC2086
for file in $files; do  # Intended word splitting
    ...
done

# Config
cat > .shellcheckrc <<EOF
shell=bash
severity=warning
enable=add-default-case,deprecate-which
disable=SC1091  # Source file not found (OK trong CI)
EOF
```

---

<a id="p10"></a>
## P10. Production Best Practices

### Bước 1: Cấu trúc project

```bash
myapp/
├── bin/                  # Main scripts
│   ├── myapp
│   └── myapp-helper
├── lib/                  # Library functions
│   ├── logging.sh
│   ├── utils.sh
│   └── config.sh
├── tests/                # bats tests
│   ├── test_utils.bats
│   └── test_main.bats
├── conf/                 # Config files
│   └── myapp.conf
├── Makefile
├── README.md
└── CHANGELOG.md
```

### Bước 2: Script hoàn chỉnh

```bash
#!/usr/bin/env bash
#
# myapp - Application manager
#
# Usage: myapp <command> [options]
#
set -Eeuo pipefail
IFS=$'\n\t'

VERSION="1.0.0"
PROG="${0##*/}"
CONFIG="${MYAPP_CONFIG:-/etc/myapp/myapp.conf}"
LOG_LEVEL="${LOG_LEVEL:-info}"

# === Source libs ===
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/lib/logging.sh"
source "$SCRIPT_DIR/lib/utils.sh"

# === Defaults ===
DRY_RUN=0
VERBOSE=0

# === Functions ===
usage() {
    cat <<EOF
$PROG v$VERSION - MyApp management tool

Usage: $PROG <command> [options]

Commands:
    start       Start the application
    stop        Stop the application
    restart     Restart the application
    status      Show status
    logs        Show logs
    backup      Create backup
    help        Show this help

Options:
    -v, --verbose     Verbose output
    -d, --dry-run     Don't make changes
    -c, --config FILE Use custom config
    -h, --help        Show this help
    --version         Show version

Examples:
    $PROG start
    $PROG backup --dry-run
    $PROG --version
EOF
}

version() {
    echo "$PROG v$VERSION"
}

parse_args() {
    while [[ $# -gt 0 ]]; do
        case $1 in
            -v|--verbose)
                VERBOSE=1
                LOG_LEVEL=debug
                shift
                ;;
            -d|--dry-run)
                DRY_RUN=1
                shift
                ;;
            -c|--config)
                CONFIG="$2"
                shift 2
                ;;
            -h|--help)
                usage
                exit 0
                ;;
            --version)
                version
                exit 0
                ;;
            -*)
                die "Unknown option: $1"
                ;;
            *)
                COMMAND="$1"
                shift
                break
                ;;
        esac
    done
}

require_root() {
    [[ $EUID -eq 0 ]] || die "Must run as root"
}

require_cmd() {
    command -v "$1" >/dev/null 2>&1 || die "Missing command: $1"
}

die() {
    echo "[ERROR] $*" >&2
    exit 1
}

run_cmd() {
    if [[ $DRY_RUN -eq 1 ]]; then
        echo "[DRY-RUN] $*"
    else
        log_debug "Running: $*"
        "$@"
    fi
}

start() {
    log_info "Starting application..."
    require_cmd systemctl
    require_root

    if systemctl is-active --quiet myapp; then
        log_warn "Already running"
        return 0
    fi

    run_cmd systemctl start myapp
    log_info "Started"
}

stop() {
    log_info "Stopping application..."
    require_root

    if ! systemctl is-active --quiet myapp; then
        log_warn "Not running"
        return 0
    fi

    run_cmd systemctl stop myapp
    log_info "Stopped"
}

status() {
    if systemctl is-active --quiet myapp; then
        echo "Status: RUNNING"
        systemctl status myapp --no-pager
    else
        echo "Status: STOPPED"
    fi
}

backup() {
    log_info "Creating backup..."
    local dest="/var/backups/myapp-$(date +%Y%m%d-%H%M%S).tar.gz"

    run_cmd tar czf "$dest" \
        -C /opt/myapp data/ \
        -C /etc/myapp myapp.conf

    log_info "Backup saved to: $dest"
}

main() {
    parse_args "$@"

    [[ -z "${COMMAND:-}" ]] && { usage; exit 0; }

    case "$COMMAND" in
        start)   start ;;
        stop)    stop ;;
        restart) stop; start ;;
        status)  status ;;
        backup)  backup ;;
        help|--help|-h) usage ;;
        version|--version) version ;;
        *)       die "Unknown command: $COMMAND" ;;
    esac
}

main "$@"
```

### Bước 3: Logging library

```bash
# lib/logging.sh
readonly LOG_LEVELS=(error warn info debug)

log_level_num() {
    case "$1" in
        error) echo 1 ;;
        warn)  echo 2 ;;
        info)  echo 3 ;;
        debug) echo 4 ;;
        *)     echo 3 ;;
    esac
}

_log() {
    local level=$1
    shift
    local current_level_num
    current_level_num=$(log_level_num "$LOG_LEVEL")
    local msg_level_num
    msg_level_num=$(log_level_num "$level")

    [[ $msg_level_num -le $current_level_num ]] || return 0

    local timestamp
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "[$timestamp] [$level] $*" >&2
}

log_error() { _log error "$@"; }
log_warn()  { _log warn "$@"; }
log_info()  { _log info "$@"; }
log_debug() { _log debug "$@"; }
```

### Bước 4: Makefile

```makefile
# Makefile
SHELL := /usr/bin/env bash

.PHONY: install test lint clean

install:
	install -d /usr/local/bin
	install -m 755 bin/myapp /usr/local/bin/
	install -d /usr/local/lib/myapp
	install -m 644 lib/*.sh /usr/local/lib/myapp/

test:
	bats tests/

lint:
	shellcheck bin/* lib/*.sh

clean:
	rm -f /tmp/myapp-*

run:
	./bin/myapp

dist:
	tar czf myapp-$(grep VERSION bin/myapp | head -1 | cut -d'"' -f2).tar.gz \
		bin/ lib/ README.md
```

### Bước 5: Distribution

```bash
# Tarball
make dist

# RPM
cat > myapp.spec <<EOF
Name: myapp
Version: 1.0.0
Release: 1
Summary: My Application
License: MIT

%description
Long description

%install
mkdir -p %{buildroot}/usr/local/bin
install -m 755 bin/myapp %{buildroot}/usr/local/bin/

%files
/usr/local/bin/myapp

%post
systemctl daemon-reload
EOF

rpmbuild -ba myapp.spec

# DEB
mkdir -p myapp-1.0.0/DEBIAN
cat > myapp-1.0.0/DEBIAN/control <<EOF
Package: myapp
Version: 1.0.0
Section: utils
Priority: optional
Architecture: amd64
Maintainer: Your Name <[email protected]>
Description: My Application
EOF

dpkg-deb --build myapp-1.0.0
```

---

## 🎯 Bài tập P0-P10

1. Viết script backup với strict mode, logging, trap errors
2. Parse CSV file bằng awk, thống kê theo cột
3. Script loop qua servers qua SSH, check uptime
4. bats test cho 3 functions: add, greet, parse_args
5. Daemon script có start/stop/status với PID file
6. Parallel download 10 files bằng xargs
7. Đóng gói script thành .deb package

---

> **💡 Tip cuối**: Bash càng strict càng tốt. Dùng `set -Eeuo pipefail` + `IFS=$'\n\t'`. Lint với `shellcheck`. Test với `bats`. Source lib thay vì copy-paste. Log everything.

---

*Tạo bởi tài liệu học Bash nâng cao - Chúc bạn thành công! 🚀*
