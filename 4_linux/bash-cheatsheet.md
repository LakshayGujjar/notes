# 🔧 Bash Scripting Comprehensive Cheatsheet

> A complete reference for Bash scripting — covering theory, internals, quoting, arrays, functions, I/O, error handling, security, and real-world patterns.

---

## 📚 Table of Contents

1. [Core Concepts & Theory](#1-core-concepts--theory)
2. [Variables & Data Types](#2-variables--data-types)
3. [Parameter Expansion — The Complete Guide](#3-parameter-expansion--the-complete-guide)
4. [Quoting — The Definitive Guide](#4-quoting--the-definitive-guide)
5. [Arrays](#5-arrays)
6. [Arithmetic](#6-arithmetic)
7. [Conditionals & Tests](#7-conditionals--tests)
8. [Loops](#8-loops)
9. [Functions](#9-functions)
10. [I/O Redirection & Pipelines](#10-io-redirection--pipelines)
11. [String Operations](#11-string-operations)
12. [Error Handling & Defensive Scripting](#12-error-handling--defensive-scripting)
13. [Script Structure & Best Practices](#13-script-structure--best-practices)
14. [Process Management](#14-process-management)
15. [File & Path Operations](#15-file--path-operations)
16. [Text Processing in Scripts](#16-text-processing-in-scripts)
17. [Advanced Features](#17-advanced-features)
18. [Debugging & Profiling](#18-debugging--profiling)
19. [Security](#19-security)
20. [Real-World Script Patterns & Recipes](#20-real-world-script-patterns--recipes)

---

## 1. Core Concepts & Theory

### What is Bash?

Bash (**B**ourne **A**gain **SH**ell) was written by Brian Fox in 1989 as a free replacement for the Bourne shell (`sh`). It is the default shell on most Linux distributions and macOS (pre-Catalina).

| Shell | Standard | Notes |
|-------|---------|-------|
| `sh` | POSIX | Portable — runs everywhere; limited features |
| `bash` | Superset of POSIX | Rich scripting; arrays, `[[`, process substitution |
| `dash` | Near-POSIX | Fast; Ubuntu's `/bin/sh`; no arrays, no `[[` |
| `zsh` | Superset | Interactive power user shell; mostly bash-compatible |

**When to use Bash vs other languages:**

```
Use Bash when:                    Use Python/Ruby/Go when:
✅ Gluing CLI tools together      ❌ Complex data structures needed
✅ File/process manipulation      ❌ HTTP requests / JSON parsing
✅ CI/CD pipelines                ❌ Error handling is critical
✅ System admin tasks             ❌ Code > ~200 lines
✅ One-off automation             ❌ Cross-platform portability needed
✅ Reading/writing files          ❌ Mathematical computation
```

---

### How Bash Executes Scripts

```
Source text: echo "hello $name"
     │
     ▼  1. Tokenisation — split into words and operators
[ echo ] [ "hello $name" ]
     │
     ▼  2. Parsing — build command tree, identify syntax
Command: echo, arg: "hello $name"
     │
     ▼  3. Expansions (in order):
        a. Brace expansion        {a,b} → a b
        b. Tilde expansion        ~ → /home/user
        c. Parameter expansion    $name → Alice
        d. Arithmetic expansion   $(( 1+1 )) → 2
        e. Command substitution   $(date) → Mon Jan...
        f. Word splitting         split on IFS
        g. Filename expansion     *.txt → file1.txt file2.txt
     │
     ▼  4. Execution — run the command
```

```bash
# See the exact expansion order with set -x
set -x
name="Alice"
echo "hello $name"        # + echo 'hello Alice'
set +x
```

---

### Interactive vs Non-interactive Shells

```bash
# An interactive shell reads commands from a terminal
# A non-interactive shell reads commands from a script file

# Detect interactive shell
if [[ $- == *i* ]]; then       # $- contains current option flags; 'i' = interactive
    echo "This is interactive"
fi

# Or check if stdin is a terminal
if [[ -t 0 ]]; then            # -t 0 = file descriptor 0 (stdin) is a terminal
    echo "stdin is a terminal"
fi

# Why it matters:
# Interactive shells:   source ~/.bashrc, set PS1, enable job control
# Non-interactive:      skip startup files (usually), no prompts, pipelines OK
```

---

### Login vs Non-login Shells

```
Login shell startup files (read in order):
  /etc/profile
  └── /etc/profile.d/*.sh
  ~/.bash_profile  ← (or ~/.bash_login, or ~/.profile — first found wins)
      └── (good practice: source ~/.bashrc from here)

Non-login interactive shell:
  /etc/bash.bashrc
  ~/.bashrc

Non-interactive shell (scripts):
  Only $BASH_ENV (if set) — usually nothing

Login shell exit:
  ~/.bash_logout
```

```bash
# Detect login shell
if shopt -q login_shell; then     # shopt -q = quiet, returns exit code
    echo "Login shell"
fi

# Force login shell:  bash -l   or   bash --login
# Force non-interactive: bash -c "command"
```

---

### Subshells vs Child Processes

```
Parent shell
├── Subshell ( )          — fork of parent, inherits ALL variables
│   └── Variables set here do NOT affect parent
├── Child process cmd     — fork+exec, inherits only EXPORTED variables
│   └── A new program, not a shell copy
└── Command group { }     — runs IN the current shell, no fork
```

```bash
x=10

# Subshell — created by ( ), pipelines, $(), <()
(x=99; echo "inside subshell: $x")   # subshell: x=99
echo "parent: $x"                     # parent: x=10 — unchanged

# $BASH_SUBSHELL — depth of subshell nesting
echo "level: $BASH_SUBSHELL"          # 0 in main shell
( echo "level: $BASH_SUBSHELL" )      # 1 inside subshell

# Each command in a pipeline runs in a subshell
echo "hello" | read line              # read runs in subshell — $line lost!
echo "$line"                          # empty!

# Fix with lastpipe (bash 4.2+) or process substitution
shopt -s lastpipe                     # last pipeline cmd runs in current shell
echo "hello" | read line
echo "$line"                          # "hello" — works now!
```

---

### The Environment

```bash
# The environment = set of exported KEY=VALUE pairs passed to child processes
env                     # print all environment variables
printenv PATH           # print specific env var
printenv                # same as env

export VAR="value"      # export variable to child processes
export -p               # show all exported variables (bash declare -x format)

# Variables are NOT inherited unless exported
local_var="only in shell"
child_var="visible to children"
export child_var

bash -c 'echo "$local_var"'   # empty — not exported
bash -c 'echo "$child_var"'   # "visible to children"

# Run command with modified environment (without changing current env)
VAR=override command           # sets VAR only for this command
env -i PATH=/usr/bin bash      # clean environment with only PATH set
env -u VAR command             # unset VAR for this command only
```

---

### File Descriptors

```
Process FD table:
┌────┬────────────────────────────────────────┐
│ FD │ Default connection                     │
├────┼────────────────────────────────────────┤
│  0 │ stdin  — keyboard / piped input        │
│  1 │ stdout — terminal / piped output       │
│  2 │ stderr — terminal (separate channel)   │
│  3+ │ custom FDs opened by script           │
└────┴────────────────────────────────────────┘
```

```bash
# /dev/fd/ — access file descriptors as paths
ls /dev/fd/                   # shows current open FDs (0, 1, 2, ...)
echo "hi" > /dev/fd/1         # write to stdout (same as echo "hi")
cat /dev/fd/0                 # read from stdin

# Check if FD is open
if [[ -e /dev/fd/3 ]]; then   # check if FD 3 is open
    echo "FD 3 is open"
fi

# Custom FD example
exec 3> /tmp/mylog.txt        # open FD 3 for writing to a file
echo "log message" >&3        # write to the file via FD 3
exec 3>&-                     # close FD 3
```

---

### The Process Model

```bash
echo "Script PID:        $$"       # PID of current bash process
echo "Parent PID:        $PPID"    # PID of parent process
echo "Subshell PID:      $BASHPID" # PID of THIS bash (differs from $$ in subshells)

# $$ vs $BASHPID difference
echo "$$"                          # always the TOP-LEVEL shell's PID (even in subshell)
( echo "$$"; echo "$BASHPID" )     # $$ = parent PID, $BASHPID = subshell's own PID

# Use $BASHPID for unique temp files, NOT $$
tmpfile="/tmp/script.$$"           # ⚠️ $$ is same in all subshells — not unique!
tmpfile="/tmp/script.$BASHPID"     # ✅ unique per bash process
```

---

### Script Execution Methods

```bash
# Method 1: bash script.sh
# - Starts a NEW bash process
# - Reads script as argument
# - Does NOT need execute permission
# - Current directory not changed
bash script.sh

# Method 2: ./script.sh
# - Requires execute permission (chmod +x)
# - Requires valid shebang (#!/bin/bash)
# - Starts new process per shebang
chmod +x script.sh
./script.sh

# Method 3: source script.sh  (or . script.sh)
# - Runs script IN CURRENT SHELL — no new process
# - Variable changes persist after script ends
# - exit command exits the calling shell!
source script.sh
. script.sh           # POSIX synonym

# Method 4: exec script.sh
# - REPLACES current shell with the script
# - No return — when script ends, shell session ends
# - Used to save a process fork
exec script.sh        # ⚠️ no coming back from this!
```

---

### Exit Codes

```bash
# Exit codes: 0 = success, 1-255 = failure
# Convention: 1=general error, 2=misuse, 126=not executable, 127=not found

echo "last exit code: $?"     # $? = exit code of most recent command
true;  echo $?                 # 0 — true always succeeds
false; echo $?                 # 1 — false always fails
ls /nonexistent 2>/dev/null; echo $?  # 2 — ls error

# Exit codes for flow control
if grep -q "pattern" file.txt; then   # grep -q: quiet, use exit code only
    echo "found"
fi

command && echo "success"     # && runs only if command succeeded (exit 0)
command || echo "failed"      # || runs only if command failed (exit non-0)

# Custom exit codes in scripts
exit 0       # success
exit 1       # general failure
exit 2       # usage/argument error

# Capture exit code after a pipeline
ls /tmp | grep "something"
echo "${PIPESTATUS[@]}"       # exit code of EACH command in the pipeline
```

---

## 2. Variables & Data Types

### Variable Assignment Rules

```bash
# ✅ Correct: NO spaces around =
name="Alice"
count=42
PATH_PREFIX=/usr/local

# 🔴 Anti-pattern: spaces cause errors
name = "Alice"    # ERROR: tries to run command "name" with args "=" and "Alice"
count =42         # ERROR: runs "count" as command
count= 42         # Sets count="" then runs "42" as command

# Naming conventions
MY_CONSTANT=value        # UPPER_CASE: global, exported, or constant
local_variable=value     # lower_case: local to function or script
_private_var=value       # leading underscore: "private" by convention
```

---

### declare — Type Attributes

```bash
# declare gives variables type attributes
declare -i count=0         # integer: arithmetic forced in assignment
declare -r PI=3.14159      # readonly: cannot be changed
declare -l lower_only      # auto-lowercase on assignment
declare -u UPPER_ONLY      # auto-uppercase on assignment
declare -x EXPORTED_VAR    # export to child processes (same as export)
declare -a indexed_array   # indexed array
declare -A assoc_array     # associative array (hash map)
declare -n ref             # nameref — an alias to another variable
declare -p varname         # print variable with type info (great for debugging)

# Integer attribute behaviour
declare -i count
count="hello"              # sets count=0 (failed arithmetic = 0)
count="5 + 3"              # sets count=8 (arithmetic evaluated)
count=count+1              # increments by 1 (no $ needed!)

# Nameref — a reference to another variable (bash 4.3+)
declare -n ref=myvar       # ref is now an alias for myvar
myvar="original"
echo "$ref"                # "original"
ref="changed"              # modifies myvar
echo "$myvar"              # "changed"

# Print variable declaration (invaluable for debugging)
declare -p PATH            # declare -x PATH="/usr/local/bin:/usr/bin:..."
declare -p count           # declare -i count="8"
```

---

### local — Function Scope

```bash
# Variables in bash are GLOBAL by default — even inside functions!
# Always use 'local' for function variables

# 🔴 Anti-pattern: no local
bad_function() {
    result="computed"        # sets GLOBAL $result — side effect!
    temp="intermediate"      # also global!
}

# ✅ Right way: use local
good_function() {
    local result             # declare as local first
    local temp="intermediate" # can initialise at same time
    result="computed"
    echo "$result"
}

# local + declare can be combined
my_function() {
    local -i count=0         # local integer
    local -a items=()        # local array
    local -r CONST="fixed"   # local readonly
}

# local only affects the current function — child functions see globals
outer() {
    local x=10
    inner
}
inner() {
    echo "$x"    # sees outer's local x=10 (dynamic scoping)
}
```

---

### Null vs Empty vs Unset

```bash
# Three distinct states for a variable:
# Unset:  variable has never been set (or was unset'd)
# Empty:  variable exists but has value ""
# Set:    variable exists and has non-empty value

unset myvar          # now myvar is UNSET

myvar=""             # now myvar is EMPTY (set but null)

myvar="hello"        # now myvar is SET

# Testing the three states
[[ -v myvar ]]       # true if SET (any value, even empty) — bash 4.2+
[[ -z "$myvar" ]]    # true if EMPTY or UNSET (zero length)
[[ -n "$myvar" ]]    # true if SET and NON-EMPTY

# ${var+x} trick — POSIX way to check if set (even if empty)
if [[ "${myvar+x}" == "x" ]]; then    # expands to "x" only if set
    echo "myvar is set"
fi

# Practical example
function requires_arg() {
    if [[ ! -v 1 ]]; then              # check if $1 was passed at all
        echo "Error: argument required" >&2
        return 1
    fi
    if [[ -z "$1" ]]; then             # check if $1 is empty
        echo "Error: argument is empty" >&2
        return 1
    fi
}
```

---

## 3. Parameter Expansion — The Complete Guide

### Default Values

```bash
# ${var:-default}  — use default if var is unset OR empty
name=""
echo "${name:-Anonymous}"    # "Anonymous" — name is empty
unset name
echo "${name:-Anonymous}"    # "Anonymous" — name is unset
name="Alice"
echo "${name:-Anonymous}"    # "Alice"     — name is set

# ${var:=default}  — assign default to var if unset or empty (modifies var!)
unset name
echo "${name:=Anonymous}"    # "Anonymous" — also sets name="Anonymous"
echo "$name"                 # "Anonymous" — var was modified

# ${var:+alt}  — use alt if var IS set (opposite of :-)
name="Alice"
echo "${name:+Hello, $name}"  # "Hello, Alice" — var is set
unset name
echo "${name:+Hello, $name}"  # "" — var is unset, expansion is empty

# ${var:?error}  — exit with error if unset or empty
echo "${REQUIRED_VAR:?REQUIRED_VAR must be set}"   # exits script if not set

# Without colon — only triggers for UNSET (not empty)
name=""
echo "${name-default}"       # "" — name is SET (empty), don't use default
echo "${name:-default}"      # "default" — name is empty, use default
```

---

### Substring Extraction

```bash
str="Hello, World!"

# ${var:offset}      — from offset to end
echo "${str:7}"        # "World!"  — starts at index 7

# ${var:offset:len}  — length characters starting at offset
echo "${str:7:5}"      # "World"  — 5 chars from index 7

# Negative offset — count from end (space required before -)
echo "${str: -6}"      # "orld!"  — last 6 chars (note the space!)
echo "${str: -6:5}"    # "orld"   — 5 chars starting 6 from end

# Length
echo "${#str}"         # 13  — string length

# With arrays
arr=(a b c d e)
echo "${arr[@]:1:3}"   # "b c d" — 3 elements from index 1
echo "${arr[@]: -2}"   # "d e"   — last 2 elements
```

---

### Prefix and Suffix Removal

```bash
path="/home/alice/docs/report.tar.gz"

# Remove shortest prefix matching pattern
echo "${path#*/}"       # "home/alice/docs/report.tar.gz"  — removes up to first /

# Remove longest prefix matching pattern
echo "${path##*/}"      # "report.tar.gz"  — removes up to LAST / (like basename)

# Remove shortest suffix matching pattern
echo "${path%.*}"       # "/home/alice/docs/report.tar"  — removes last .xxx

# Remove longest suffix matching pattern
echo "${path%%.*}"      # "/home/alice/docs/report"  — removes from FIRST dot

# Practical: extract parts of filenames
filename="archive.tar.gz"
echo "${filename%%.*}"     # "archive"       — name without any extension
echo "${filename#*.}"      # "tar.gz"        — everything after first dot
echo "${filename##*.}"     # "gz"            — last extension only

# Extract directory from path (like dirname)
echo "${path%/*}"          # "/home/alice/docs"

# Extract filename from path (like basename)
echo "${path##*/}"         # "report.tar.gz"
```

---

### Pattern Substitution

```bash
str="hello world hello"

# ${var/pattern/replacement}  — replace FIRST match
echo "${str/hello/hi}"        # "hi world hello"

# ${var//pattern/replacement} — replace ALL matches
echo "${str//hello/hi}"       # "hi world hi"

# ${var/#pattern/replacement} — replace if pattern matches at START
echo "${str/#hello/hi}"       # "hi world hello"

# ${var/%pattern/replacement} — replace if pattern matches at END
echo "${str/%hello/hi}"       # "hello world hi"

# Delete (empty replacement)
echo "${str//hello/}"         # " world "

# Patterns can use globs
str="file_name_v2.txt"
echo "${str//_/-}"            # "file-name-v2.txt"  — replace all _ with -

# Practical: sanitise filename
filename="My File (2024).txt"
safe="${filename//[^a-zA-Z0-9.]/_}"   # replace non-alphanumeric/dot with _
echo "$safe"                           # "My_File__2024_.txt"
```

---

### Case Conversion

```bash
str="Hello World"

echo "${str^}"     # "Hello World"  — uppercase first char of string
echo "${str^^}"    # "HELLO WORLD"  — uppercase ALL chars
echo "${str,}"     # "hello World"  — lowercase first char
echo "${str,,}"    # "hello world"  — lowercase ALL chars

# With pattern — only matching characters are changed
echo "${str^^[aeiou]}"   # "hEllO wOrld"  — uppercase vowels only
echo "${str,,[A-Z]}"     # "hello world"  — lowercase uppercase letters only

# Case toggle (~~ toggles, ~ toggles first char)
str="Hello"
echo "${str~}"     # "hello"  — toggle first char's case
echo "${str~~}"    # "hELLO"  — toggle ALL chars' case
```

---

### Indirect Expansion and Variable Lists

```bash
# ${!var} — expand the variable NAMED by var (indirect reference)
fruit="apple"
apple="red"
echo "${!fruit}"        # "red" — expands $fruit → "apple", then expands $apple

# Practical use: dynamic variable names
for color in red green blue; do
    declare "${color}_val=255"   # create red_val=255, green_val=255, etc.
done
var="red_val"
echo "${!var}"              # "255" — access via indirect expansion

# ${!prefix*} / ${!prefix@} — list variable names matching prefix
foo=1; foobar=2; foobaz=3
echo "${!foo*}"             # "foo foobar foobaz"  — all vars starting with "foo"
for varname in "${!foo@}"; do   # iterate over matching variable names
    echo "$varname = ${!varname}"
done
```

---

## 4. Quoting — The Definitive Guide

### Why Quoting Matters

```bash
# Word splitting + glob expansion happen on UNQUOTED variable expansions

filename="my file.txt"
touch "my file.txt"           # create file with space in name

# 🔴 Anti-pattern: unquoted variable
rm $filename                  # tries to rm "my" and "file.txt" separately!

# ✅ Right way: always quote
rm "$filename"                # treats as single argument: "my file.txt"

# Glob expansion danger
pattern="*.txt"
ls $pattern                   # fine here (works like ls *.txt)
if [[ $pattern == *.txt ]]; then  # GOOD — [[ prevents glob expansion
    echo "is a txt pattern"
fi
[[ "$pattern" == "*.txt" ]]   # also fine — but now literal match, not glob!

# The rule: double-quote ALL variable expansions unless you explicitly want
# word splitting or glob expansion (which is rare)
```

---

### Single vs Double Quotes

```bash
name="Alice"

# Single quotes — COMPLETELY literal, no expansion at all
echo 'Hello $name'         # "Hello $name" — literal dollar sign
echo 'It'\''s a test'      # "It's a test" — embed single quote with '\''

# Double quotes — allow $, ``, \, and ! but prevent word splitting and glob
echo "Hello $name"         # "Hello Alice" — $name expanded
echo "Today: $(date)"      # command substitution works
echo "Path: $PATH"         # variable expanded
echo "Tab:\there"          # \t is NOT special in double quotes — use $'\t'

# $'...' — ANSI-C quoting: backslash sequences ARE interpreted
echo $'Hello\tWorld'       # "Hello	World" — tab character
echo $'Line1\nLine2'       # two lines
echo $'\e[31mRed\e[0m'     # red coloured text
echo $'\u2665'             # ♥ — Unicode character
echo $'\x41'               # "A" — hex escape

# Combining quoting styles
echo "Hello"$'\t'"World"   # "Hello	World" — mix styles for tab in double-quoted

# $"..." — locale translation (almost never used)
echo $"Hello"              # translates "Hello" using gettext if available
```

---

### Backticks vs $()

```bash
# 🔴 Anti-pattern: backticks (legacy)
result=`ls /tmp`           # harder to read, can't nest easily

# ✅ Right way: $() — command substitution
result=$(ls /tmp)          # cleaner, nestable, POSIX
nested=$(echo $(ls /tmp))  # easily nested
date=$(date +'%Y-%m-%d')   # clean, readable

# Nesting — where backticks become unusable
# Backtick nesting requires escaping:
outer=`echo \`date\``      # ugly and error-prone
# $() nesting is clean:
outer=$(echo $(date))      # clean and readable
deep=$(cat $(find . -name "*.conf" | head -1))  # deeply nested — still readable
```

---

### Quoting Arrays — Critical Differences

```bash
arr=("first item" "second item" "third item")

# "${arr[@]}" — CORRECT: each element as a separate word
for item in "${arr[@]}"; do     # 3 iterations, spaces preserved
    echo "item: $item"
done

# "${arr[*]}" — joins all elements with IFS[0] into ONE string
echo "${arr[*]}"                # "first item second item third item"

# ${arr[@]} — WRONG: unquoted, causes word splitting on spaces
for item in ${arr[@]}; do       # 6 iterations! Spaces split items
    echo "item: $item"
done

# Passing array to a command
printf '%s\n' "${arr[@]}"       # ✅ each element as separate argument
printf '%s\n' "${arr[*]}"       # ❌ entire array as one argument
```

---

### Heredocs

```bash
# <<EOF — standard heredoc (variables ARE expanded)
name="Alice"
cat <<EOF
Hello, $name
Today is $(date)
Path is $PATH
EOF

# <<'EOF' — quoted heredoc (NO expansion — completely literal)
cat <<'EOF'
Hello, $name          # literal $name — not expanded
$(date)               # literal — not expanded
EOF

# <<-EOF — strip leading TABS (allows indented heredoc in scripts)
if true; then
    cat <<-EOF
        This line has leading tabs stripped
        $name is expanded here
    EOF
fi

# Heredoc as string (redirect into variable via command substitution)
text=$(cat <<'EOF'
Line 1
Line 2
Line 3
EOF
)
echo "$text"     # multiline string

# Heredoc to file
cat > /tmp/config.txt <<EOF
server=localhost
port=8080
user=$name
EOF
```

---

### Word Splitting and IFS

```bash
# IFS (Internal Field Separator) controls word splitting
# Default IFS=$' \t\n' — space, tab, newline

# Splitting on default IFS
str="one two three"
read -ra arr <<< "$str"       # split into array
echo "${#arr[@]}"             # 3

# Change IFS for controlled splitting
line="alice:30:London"
IFS=: read -r name age city <<< "$line"
echo "$name $age $city"       # "alice 30 London"

# ⚠️ IFS is global — always restore after changing
OLD_IFS="$IFS"
IFS=,
arr=($csv_line)               # split on commas
IFS="$OLD_IFS"                # restore

# Better: use subshell to contain IFS change
result=$( IFS=,; echo "${arr[*]}" )   # join array with comma, IFS change is local

# Safest IFS for scripts — only split on newlines and tabs (not spaces)
IFS=$'\n\t'                   # set at top of script for safer splitting
```

---

## 5. Arrays

### Indexed Arrays

```bash
# Declaration and assignment
arr=()                        # empty array
arr=(one two three)           # initialise with values
arr[0]="first"                # assign by index
arr[5]="sixth"                # sparse: indices 1-4 are unset

# Access
echo "${arr[0]}"              # first element
echo "${arr[-1]}"             # last element (bash 4.3+)
echo "${arr[@]}"              # ALL elements (as separate words)
echo "${arr[*]}"              # all elements joined with IFS[0]

# Metadata
echo "${#arr[@]}"             # number of elements
echo "${!arr[@]}"             # all INDICES (useful for sparse arrays)

# Append
arr+=("four" "five")          # append multiple elements
arr+=("six")                  # append single element

# Delete
unset 'arr[2]'                # delete index 2 (leaves gap — sparse!)
unset arr                     # delete entire array

# Negative indexing
last="${arr[-1]}"             # last element
second_last="${arr[-2]}"      # second to last
```

---

### mapfile / readarray

```bash
# mapfile (=readarray) reads lines into an array safely
# Handles filenames with spaces correctly

# Read file into array
mapfile -t lines < /etc/passwd           # -t strips trailing newlines
echo "${#lines[@]}"                      # number of lines
echo "${lines[0]}"                       # first line

# Read command output into array
mapfile -t files < <(find /tmp -name "*.txt")  # process substitution
for f in "${files[@]}"; do              # iterate safely (handles spaces in names)
    echo "$f"
done

# Read with custom delimiter
mapfile -t -d ',' arr <<< "a,b,c,"     # split on comma, -d sets delimiter

# Read only N lines starting at offset
mapfile -t -n 5 -s 10 arr < bigfile.txt  # skip 10 lines, read next 5

# Callback: call function for each line read
process_line() { echo "Line $1: $2"; }  # called with index and line
mapfile -t -C process_line -c 1 arr < file.txt  # callback every 1 line
```

---

### Array Operations

```bash
arr=(a b c d e f)

# Slice — extract sub-array
echo "${arr[@]:2:3}"          # "c d e" — 3 elements starting at index 2
slice=("${arr[@]:1:4}")       # copy slice into new array

# Copy array
copy=("${arr[@]}")            # shallow copy

# Join array into string
IFS=, joined="${arr[*]}"      # "a,b,c,d,e,f" with IFS=,
joined=$(IFS=:; echo "${arr[*]}")  # "a:b:c:d:e:f" — subshell contains IFS change

# Reverse array (pure bash)
reversed=()
for (( i=${#arr[@]}-1; i>=0; i-- )); do
    reversed+=("${arr[$i]}")
done

# Search array for value
contains() {
    local needle="$1"; shift
    local element
    for element in "$@"; do
        [[ "$element" == "$needle" ]] && return 0   # found
    done
    return 1                                         # not found
}
contains "c" "${arr[@]}" && echo "found"

# Dedup array (preserve order)
dedup() {
    local -A seen=()
    local result=()
    for item in "$@"; do
        [[ -v seen["$item"] ]] && continue   # skip if already seen
        seen["$item"]=1
        result+=("$item")
    done
    printf '%s\n' "${result[@]}"
}
```

---

### Associative Arrays

```bash
# Declare FIRST — required for associative arrays
declare -A user

# Assignment
user[name]="Alice"
user[age]=30
user[city]="London"

# Initialise at once
declare -A config=([host]="localhost" [port]="8080" [debug]="false")

# Access
echo "${user[name]}"          # "Alice"
echo "${user[missing]}"       # "" — no error, just empty

# Check if key exists
if [[ -v user[city] ]]; then  # -v checks if key is set
    echo "city is set"
fi

# All keys
echo "${!user[@]}"            # "name age city" (order not guaranteed)

# All values
echo "${user[@]}"             # "Alice 30 London"

# Iterate
for key in "${!user[@]}"; do
    printf "%-10s = %s\n" "$key" "${user[$key]}"
done

# Delete key
unset 'user[age]'

# Number of keys
echo "${#user[@]}"            # 2 (after deleting age)
```

---

### Passing Arrays to Functions

```bash
# Arrays CANNOT be passed directly to functions — they collapse to a string
# Solutions: pass expanded elements, nameref, or serialize

# Method 1: pass expanded elements (read-only, copy)
process() {
    local arr=("$@")          # reconstruct array from arguments
    echo "count: ${#arr[@]}"
    echo "first: ${arr[0]}"
}
myarr=(one two three)
process "${myarr[@]}"         # pass all elements as separate args

# Method 2: nameref (bash 4.3+) — pass by reference (read/write)
modify_array() {
    local -n arr_ref=$1       # arr_ref is a reference to the caller's array
    arr_ref+=("new element")  # modifies the CALLER's array
    echo "inside: ${arr_ref[@]}"
}
myarr=(a b c)
modify_array myarr            # pass NAME, not ${myarr[@]}
echo "after: ${myarr[@]}"    # "a b c new element" — was modified!

# Returning an array from a function (nameref approach)
get_files() {
    local -n result_ref=$1    # output array passed by name
    mapfile -t result_ref < <(find /tmp -name "*.txt")
}
declare -a file_list
get_files file_list           # populate file_list
echo "${file_list[@]}"
```

---

### Common Array Patterns

```bash
# Stack (LIFO) — push and pop
stack=()
stack+=("item1")              # push
stack+=("item2")
top="${stack[-1]}"            # peek at top
unset 'stack[-1]'             # pop

# Queue (FIFO) — enqueue and dequeue
queue=()
queue+=("item1")              # enqueue
queue+=("item2")
front="${queue[0]}"           # peek at front
queue=("${queue[@]:1}")       # dequeue (remove first element)

# Set intersection
set_a=(a b c d e)
set_b=(b d f)
declare -A lookup=()
for item in "${set_b[@]}"; do lookup["$item"]=1; done
intersection=()
for item in "${set_a[@]}"; do
    [[ -v lookup["$item"] ]] && intersection+=("$item")
done
echo "${intersection[@]}"     # "b d"

# Set difference (a - b)
difference=()
for item in "${set_a[@]}"; do
    [[ ! -v lookup["$item"] ]] && difference+=("$item")
done
echo "${difference[@]}"       # "a c e"
```

## 6. Arithmetic

### $(( )) and (( ))

```bash
# $(( )) — arithmetic EXPANSION: returns result as string
result=$((5 + 3))            # result="8"
echo $((10 * 4 - 2))        # "38"
echo $((2 ** 10))            # "1024" — exponentiation
echo $((17 % 5))             # "2" — modulo
echo $((10 / 3))             # "3" — integer division (truncates)

# (( )) — arithmetic COMMAND: returns exit code 0 if result ≠ 0
(( 5 > 3 ))                  # exit 0 (true)
(( 5 < 3 ))                  # exit 1 (false)
(( x = 5 + 3 ))              # assigns x=8 and evaluates as true
(( x++ ))                    # increment x
(( x += 5 ))                 # x = x + 5

# Use (( )) directly in conditionals — no $ needed for vars!
x=5
if (( x > 3 )); then
    echo "x is greater than 3"
fi
(( x++ ))                    # increment — note: (( 0 )) is FALSE (returns 1)!
(( x == 1 )) && echo "x is 1"

# All arithmetic operators
echo $(( 5 + 3 ))    # 8   addition
echo $(( 5 - 3 ))    # 2   subtraction
echo $(( 5 * 3 ))    # 15  multiplication
echo $(( 8 / 3 ))    # 2   integer division
echo $(( 8 % 3 ))    # 2   modulo
echo $(( 2 ** 8 ))   # 256 exponentiation

# Bitwise operators
echo $(( 0b1010 & 0b1100 ))  # 8   AND
echo $(( 5 | 3 ))             # 7   OR
echo $(( 5 ^ 3 ))             # 6   XOR
echo $(( ~5 ))                # -6  NOT (two's complement)
echo $(( 1 << 4 ))            # 16  left shift
echo $(( 256 >> 4 ))          # 16  right shift

# Ternary operator
x=7
echo $(( x > 5 ? x * 2 : x + 1 ))   # 14  (x > 5 is true)

# Number bases
echo $(( 0xff ))      # 255  — hex
echo $(( 0755 ))      # 493  — octal
echo $(( 2#1010 ))    # 10   — binary using base#number notation
echo $(( 16#ff ))     # 255  — hex using base#number notation
```

---

### Floating Point

```bash
# Bash arithmetic is INTEGER ONLY — no floats in $(( ))

# ⚠️ Integer only
echo $(( 10 / 3 ))      # 3 — truncates, not 3.333

# ✅ bc — arbitrary precision calculator
echo "scale=4; 10/3" | bc          # 3.3333
echo "scale=2; sqrt(2)" | bc -l    # 1.41 (-l loads math library)
echo "10^3" | bc                    # 1000

# ✅ awk — floating point math
awk 'BEGIN { printf "%.4f\n", 10/3 }'       # 3.3333
awk 'BEGIN { printf "%.2f\n", sqrt(2) }'    # 1.41

# ✅ python3 — full floating point
python3 -c "print(10/3)"             # 3.3333333333333335
python3 -c "import math; print(math.sqrt(2))"  # 1.4142135623730951

# Comparing floats in bash (convert to integers by scaling)
a="3.14"; b="3.2"
# Multiply by 100 to compare as integers
a_scaled=$(echo "$a * 100" | bc | cut -d. -f1)
b_scaled=$(echo "$b * 100" | bc | cut -d. -f1)
(( a_scaled > b_scaled )) && echo "a > b" || echo "a <= b"
```

---

### Random Numbers

```bash
# $RANDOM — pseudo-random 0-32767 (seeded by PID + time)
echo $RANDOM                    # random number 0-32767
echo $RANDOM                    # different each time

# Range [min, max]
min=1; max=100
echo $(( RANDOM % (max - min + 1) + min ))   # random 1-100

# Seed RANDOM for reproducible sequences
RANDOM=42                       # set seed
echo $RANDOM; echo $RANDOM     # same sequence every time with seed 42

# shuf — better randomisation (uses /dev/urandom)
shuf -i 1-100 -n 1             # single random number 1-100
shuf -i 1-100 -n 5             # 5 unique random numbers
shuf -e one two three          # shuffle arguments, print random order
shuf array.txt                 # shuffle lines of a file

# /dev/urandom — cryptographically secure random bytes
head -c 16 /dev/urandom | xxd -p  # 16 random bytes as hex
head -c 32 /dev/urandom | base64  # random base64 string (good for secrets)

# Random string (letters and digits)
LC_ALL=C tr -dc 'a-zA-Z0-9' < /dev/urandom | head -c 16
```

---

## 7. Conditionals & Tests

### test, [, and [[

```bash
# test and [ are identical — POSIX, available everywhere
test -f file.txt && echo "is a file"
[ -f file.txt ] && echo "is a file"    # spaces around [ ] are REQUIRED

# [[ ]] — bash extended test — safer, more features
[[ -f file.txt ]] && echo "is a file"

# Key differences:
# [ ]  — external or builtin; needs quoting; uses -a/-o for AND/OR
# [[ ]] — bash keyword; safe without quoting; uses && || !; supports =~ and ==glob

# 🔴 Anti-pattern with [ ]
[ $name == "Alice" ]           # fails if $name is empty or has spaces!
[ -n $name ]                   # fails if $name has spaces

# ✅ Safe with [[ ]]
[[ $name == "Alice" ]]         # safe: no word splitting inside [[
[[ -n $name ]]                 # safe even without quotes
```

---

### String and File Tests

```bash
# String tests
[[ -z "$str" ]]         # true if string is empty (zero length)
[[ -n "$str" ]]         # true if string is non-empty
[[ "$a" == "$b" ]]      # string equality
[[ "$a" != "$b" ]]      # string inequality
[[ "$a" < "$b" ]]       # lexicographic less than (inside [[)
[[ "$a" > "$b" ]]       # lexicographic greater than

# Glob pattern matching with ==
[[ "$filename" == *.txt ]]      # true if filename ends in .txt
[[ "$name" == [Aa]lice ]]       # true if name is "Alice" or "alice"

# Regex matching with =~
[[ "$email" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]
# Capture groups stored in BASH_REMATCH
if [[ "2024-01-15" =~ ^([0-9]{4})-([0-9]{2})-([0-9]{2})$ ]]; then
    echo "Year:  ${BASH_REMATCH[1]}"   # 2024
    echo "Month: ${BASH_REMATCH[2]}"   # 01
    echo "Day:   ${BASH_REMATCH[3]}"   # 15
fi

# ⚠️ Regex quoting gotcha: don't quote the pattern in =~
pattern="^[0-9]+$"
[[ "123" =~ $pattern ]]   # ✅ works — unquoted variable
[[ "123" =~ "^[0-9]+$" ]] # ❌ treats as literal string, not regex!

# File tests
[[ -e path ]]    # exists (file or directory or anything)
[[ -f path ]]    # exists and is a regular FILE
[[ -d path ]]    # exists and is a DIRECTORY
[[ -L path ]]    # is a SYMLINK
[[ -r path ]]    # exists and is READABLE
[[ -w path ]]    # exists and is WRITABLE
[[ -x path ]]    # exists and is EXECUTABLE
[[ -s path ]]    # exists and has SIZE > 0
[[ -b path ]]    # block device
[[ -c path ]]    # character device
[[ -p path ]]    # named pipe (FIFO)
[[ -S path ]]    # socket
[[ -N path ]]    # modified since last read
[[ f1 -nt f2 ]]  # f1 is NEWER than f2
[[ f1 -ot f2 ]]  # f1 is OLDER than f2
[[ f1 -ef f2 ]]  # f1 and f2 are the SAME file (hard link or same path)
```

---

### if, case, select

```bash
# if / elif / else / fi
if [[ $score -ge 90 ]]; then
    grade="A"
elif [[ $score -ge 80 ]]; then
    grade="B"
elif [[ $score -ge 70 ]]; then
    grade="C"
else
    grade="F"
fi

# One-liners
[[ -f config.txt ]] && source config.txt || echo "No config" >&2
[[ $debug == true ]] && set -x

# case / esac — pattern matching switch
case "$os" in
    linux*)                      # glob pattern
        install_linux;;
    darwin | macos)              # multiple patterns with |
        install_mac;;
    windows)
        install_windows;;
    *)                           # default (catch-all)
        echo "Unknown OS: $os" >&2
        exit 1;;
esac

# case with fall-through
case "$cmd" in
    start)
        start_service
        ;&                       # ;& falls through to NEXT case body
    reload)
        reload_config;;          # both 'start' and 'reload' cases run this
    stop)
        stop_service;;
esac

# case with continue-fall-through
case "$opt" in
    -v | --verbose)
        verbose=true
        ;;&                      # ;;& tests remaining patterns too
    -v* )
        echo "Got -v flag";;     # matches as well
esac

# select — interactive numbered menu
PS3="Choose an option: "        # prompt for select
select option in "Start" "Stop" "Status" "Quit"; do
    case "$option" in
        Start)  start_service;;
        Stop)   stop_service;;
        Status) show_status;;
        Quit)   break;;
        *)      echo "Invalid option: $REPLY";;
    esac
done
```

---

## 8. Loops

### for Loops

```bash
# for-in: iterate over a list
for item in one two three; do
    echo "$item"
done

# iterate over array (CORRECT way)
fruits=("apple" "banana" "cherry")
for fruit in "${fruits[@]}"; do   # always quote with [@]
    echo "$fruit"
done

# iterate over files (glob)
for f in /tmp/*.txt; do           # glob expansion — handles spaces correctly
    [[ -f "$f" ]] || continue     # skip if no .txt files (glob not matched)
    echo "Processing: $f"
done

# iterate over command output — ⚠️ careful with newlines
while IFS= read -r line; do       # ✅ correct way for command output
    echo "$line"
done < <(find /tmp -name "*.log") # process substitution

# 🔴 Anti-pattern: for f in $(ls) — breaks on spaces!
for f in $(ls /tmp); do           # WRONG: spaces in filenames cause problems
    process "$f"
done

# C-style arithmetic loop
for (( i=0; i<10; i++ )); do
    echo "$i"
done

# Countdown
for (( i=10; i>=0; i-- )); do
    echo -ne "\r$i  "
    sleep 1
done; echo

# Iterate with index over array
arr=("a" "b" "c")
for i in "${!arr[@]}"; do         # ${!arr[@]} gives indices
    echo "arr[$i] = ${arr[$i]}"
done
```

---

### while and until

```bash
# while — loop while condition is TRUE (exit code 0)
count=0
while (( count < 5 )); do
    echo "$count"
    (( count++ ))
done

# until — loop while condition is FALSE
count=0
until (( count >= 5 )); do
    echo "$count"
    (( count++ ))
done

# while read — line-by-line file reading (THE canonical pattern)
while IFS= read -r line; do   # IFS=  : preserve leading/trailing whitespace
    echo "Line: $line"        # -r    : don't interpret backslashes
done < file.txt               # redirect from file

# Read from command output
while IFS= read -r line; do
    echo "$line"
done < <(some_command)        # process substitution

# Read multiple fields at once
while IFS=: read -r user pass uid gid info home shell; do
    echo "User: $user, Shell: $shell"
done < /etc/passwd

# while read with pipe — ⚠️ runs in subshell, variables lost after!
cat file.txt | while read line; do    # WRONG: subshell — $count changes lost
    (( count++ ))
done
echo "count: $count"                   # 0 — changes lost in subshell!

# ✅ Fix: use process substitution or herestring
count=0
while IFS= read -r line; do
    (( count++ ))
done < file.txt                        # redirected from file, not pipe
echo "count: $count"                   # correct count
```

---

### break, continue, Loop Control

```bash
# break — exit loop
for i in 1 2 3 4 5; do
    [[ $i -eq 3 ]] && break     # stop when i=3
    echo "$i"                    # prints 1 2
done

# continue — skip to next iteration
for i in 1 2 3 4 5; do
    [[ $i -eq 3 ]] && continue  # skip 3
    echo "$i"                    # prints 1 2 4 5
done

# break/continue with levels — for nested loops
for i in 1 2 3; do
    for j in a b c; do
        [[ $j == "b" ]] && break 2   # break out of BOTH loops
        echo "$i-$j"
    done
done

# Loop redirection — redirect entire loop output
for i in 1 2 3; do
    echo "Item $i"
done > output.txt               # all loop output to file

for i in 1 2 3; do
    echo "Item $i"
done | grep "2"                 # pipe entire loop output

# Background loops
for host in server1 server2 server3; do
    ping -c 1 "$host" &         # each ping in background
done
wait                            # wait for all background pings

# Parallel with xargs
echo "server1\nserver2\nserver3" | xargs -P 3 -I{} ping -c 1 {}  # 3 parallel
```

---

## 9. Functions

### Declaration and Arguments

```bash
# Three equivalent declaration syntaxes
greet() { echo "Hello, $1!"; }           # most common — POSIX compatible
function greet { echo "Hello, $1!"; }    # bash-specific keyword
function greet() { echo "Hello, $1!"; }  # combined (redundant but clear)

# Prefer name() syntax — most portable and idiomatic

# Arguments inside functions
show_args() {
    echo "Function name: ${FUNCNAME[0]}"  # current function name (not $0!)
    echo "Num args: $#"                   # number of arguments
    echo "First: $1"
    echo "Second: $2"
    echo "All (@): $@"                    # each arg as separate word (quoted)
    echo "All (*): $*"                    # all args as one word
    echo "Tenth: ${10}"                   # need braces for 10+
    shift                                 # remove first arg; $2 becomes $1
    echo "After shift, first: $1"
}
show_args one two three

# $0 inside a function — still the SCRIPT name, NOT the function name
my_func() {
    echo "$0"                   # script name, e.g., "/path/to/script.sh"
    echo "${FUNCNAME[0]}"       # function name: "my_func"
}
```

---

### Return Values

```bash
# return — returns an EXIT CODE (0-255 only, not a string!)
is_positive() {
    (( $1 > 0 ))    # arithmetic: exit 0 if true
}
is_positive 5 && echo "positive"
is_positive -3 || echo "not positive"

# Return string via stdout capture — the most common pattern
get_greeting() {
    local name="$1"
    echo "Hello, $name!"   # output is the "return value"
}
result=$(get_greeting "Alice")   # capture stdout
echo "$result"                   # "Hello, Alice!"

# Return via nameref (bash 4.3+) — efficient, no subshell
get_info() {
    local -n _result=$1    # nameref to caller's variable
    _result="computed value"
}
declare result
get_info result            # pass variable NAME
echo "$result"             # "computed value"

# Return via global variable (discouraged — side effects)
RESULT=""
compute() {
    RESULT="$1 * 2 = $(( $1 * 2 ))"
}
compute 5
echo "$RESULT"             # "5 * 2 = 10"
```

---

### Function Libraries and Exports

```bash
# Guard pattern — prevent double-sourcing a library
# lib/utils.sh
if [[ -n "${_UTILS_LOADED:-}" ]]; then
    return 0              # already loaded — skip
fi
_UTILS_LOADED=1           # mark as loaded

log() { echo "[$(date +'%H:%M:%S')] $*"; }
die() { echo "ERROR: $*" >&2; exit 1; }

# BASH_SOURCE — get the REAL path of the sourced file
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "${SCRIPT_DIR}/lib/utils.sh"   # reliable relative path

# Export function to subshells and env
my_func() { echo "I'm exported!"; }
export -f my_func                     # now available in child bash processes
bash -c 'my_func'                     # "I'm exported!"

# command, builtin — control which version runs
cd() {
    builtin cd "$@" && echo "Now in: $PWD"  # wrap builtin cd
}

# Override ls but still call real ls
ls() {
    command ls --color=auto "$@"   # command bypasses the function alias
}

# Check which version would run
type ls           # shows function, alias, or path
type -t ls        # just the type: "function", "alias", "file", "builtin"
```

---

### Recursive Functions

```bash
# Factorial — recursive
factorial() {
    local n=$1
    if (( n <= 1 )); then
        echo 1
        return
    fi
    local prev=$(factorial $(( n - 1 )))   # recursive call
    echo $(( n * prev ))
}
echo "5! = $(factorial 5)"    # "5! = 120"

# Tree traversal — print directory tree
tree_walk() {
    local dir="$1"
    local indent="${2:-}"
    local entry
    for entry in "$dir"/*/; do              # iterate subdirs
        [[ -d "$entry" ]] || continue
        echo "${indent}$(basename "$entry")"
        tree_walk "$entry" "${indent}  "   # recurse with increased indent
    done
    for entry in "$dir"/*; do              # iterate files
        [[ -f "$entry" ]] || continue
        echo "${indent}$(basename "$entry")"
    done
}
tree_walk /etc

# ⚠️ Warning: Bash has no tail-call optimisation — deep recursion hits stack limits
# Use iterative approaches for deep recursion
```

---

## 10. I/O Redirection & Pipelines

### File Descriptor Diagram

```
Process:
┌─────────────────────────────────────────────────────┐
│  FD 0 (stdin)  ←── keyboard / pipe / file / <<EOF  │
│  FD 1 (stdout) ──► terminal / pipe / file / >       │
│  FD 2 (stderr) ──► terminal / file / 2>             │
│  FD 3+         ──► custom files / sockets           │
└─────────────────────────────────────────────────────┘
```

```bash
# Output redirection
command > file.txt        # stdout → file (overwrite)
command >> file.txt       # stdout → file (append)
command >| file.txt       # stdout → file (force overwrite even with set -C)

# Input redirection
command < file.txt        # stdin ← file
command << EOF            # stdin ← heredoc until EOF
Line 1
EOF
command <<< "string"      # stdin ← herestring

# Stderr redirection
command 2> errors.txt     # stderr → file
command 2>> errors.txt    # stderr → file (append)
command 2>&1              # stderr → wherever stdout goes
command > out.txt 2>&1    # both stdout and stderr → file
command &> out.txt        # same (bash shorthand)
command &>> out.txt       # both → file (append)
command 2>/dev/null       # discard stderr
command > /dev/null 2>&1  # discard everything

# Order matters for 2>&1:
command > file 2>&1       # ✅ stderr → stdout → file (both to file)
command 2>&1 > file       # ❌ stderr → terminal (current stdout), stdout → file
```

---

### Custom File Descriptors

```bash
# Open custom FD for writing
exec 3> /tmp/mylog.txt          # open FD 3 → log file
echo "message" >&3              # write to FD 3 (appears in log file)
exec 3>&-                       # close FD 3

# Open custom FD for reading
exec 4< /etc/passwd             # open FD 4 ← passwd file
read -u 4 first_line            # read one line from FD 4
exec 4<&-                       # close FD 4

# Logging pattern: send all output to log AND terminal
exec > >(tee -a /var/log/script.log) 2>&1   # all output goes to tee
echo "This appears in terminal AND log file"
ls /nonexistent   # error also captured

# Save and restore stdout
exec 3>&1                       # save stdout to FD 3
exec > output.txt               # redirect stdout to file
echo "This goes to file"
exec 1>&3 3>&-                  # restore stdout from FD 3, close FD 3
echo "This goes to terminal"
```

---

### /dev/tcp and /dev/udp

```bash
# Bash can open TCP/UDP connections without netcat (if compiled with --enable-net-redirections)

# HTTP GET request via /dev/tcp
exec 3<>/dev/tcp/example.com/80   # open TCP socket to port 80
echo -e "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n" >&3  # send request
cat <&3                            # read response
exec 3>&-                          # close connection

# Check if port is open
check_port() {
    local host=$1 port=$2
    (echo >/dev/tcp/$host/$port) 2>/dev/null && echo "open" || echo "closed"
}
check_port localhost 22    # check if SSH is running

# Simple HTTP POST
exec 3<>/dev/tcp/httpbin.org/80
printf 'POST /post HTTP/1.0\r\nHost: httpbin.org\r\nContent-Length: 11\r\n\r\nhello world' >&3
cat <&3
exec 3>&-
```

---

### Pipelines and PIPESTATUS

```bash
# | pipes stdout of left to stdin of right
ls /tmp | wc -l              # count files in /tmp
cat file | sort | uniq -c | sort -rn   # word frequency

# |& pipes BOTH stdout and stderr
command |& grep "error"      # capture errors in pipeline

# Pipeline exit codes
ls /nonexistent | wc -l      # ls fails, wc succeeds
echo $?                       # 0 — only wc's exit code!

# set -o pipefail — pipeline fails if ANY stage fails
set -o pipefail
ls /nonexistent | wc -l      # now the whole pipeline fails
echo $?                       # 2 — ls's exit code propagates

# PIPESTATUS — exit code of EACH pipeline stage
ls /tmp | sort | wc -l
echo "${PIPESTATUS[@]}"       # "0 0 0" — all succeeded
ls /nope | sort | wc -l 2>/dev/null
echo "${PIPESTATUS[@]}"       # "2 0 0" — ls failed

# Capture specific pipeline stage exit code
ls /tmp | sort | wc -l
exit_of_ls="${PIPESTATUS[0]}"     # exit code of first command
exit_of_sort="${PIPESTATUS[1]}"   # exit code of second command
```

---

### Process Substitution

```bash
# <(cmd) — creates a temporary file-like FD with cmd's output
# Different from pipe: both sides run concurrently, no subshell for outer command

# Compare two command outputs without temp files
diff <(ls /dir1) <(ls /dir2)           # diff sees two "files" (FDs)
diff <(sort file1) <(sort file2)       # compare sorted versions

# Join two command outputs
paste <(cut -d: -f1 /etc/passwd) <(cut -d: -f6 /etc/passwd)

# >(cmd) — creates a FD that pipes to cmd's stdin
tee >(gzip > file.gz) >(wc -l) > /dev/null   # write to two destinations

# Process substitution doesn't create a subshell — variables persist!
count=0
while IFS= read -r line; do
    (( count++ ))
done < <(grep "error" /var/log/syslog)   # < <() feeds file-like FD to while
echo "Found $count errors"               # count IS visible here (no subshell)
```

---

### read Built-in

```bash
# read — reads one line from stdin into variables
read varname                    # read line into $varname
read first rest                 # first word → $first, rest → $rest
read -r line                    # -r: don't interpret backslashes (ALWAYS use -r)
read -p "Enter name: " name     # -p: display prompt
read -s password                # -s: silent (no echo — for passwords)
read -t 5 input                 # -t: timeout in seconds
read -n 1 char                  # -n: read exactly N characters (no Enter needed)
read -N 10 block                # -N: read EXACTLY N chars (doesn't stop at newline)
read -d ':' field               # -d: custom delimiter instead of newline
read -a arr                     # -a: read into ARRAY (splits on IFS)
read -u 3 line                  # -u: read from FD 3 instead of stdin
read -e input                   # -e: use readline (allows up-arrow history)

# The canonical line-reading pattern
while IFS= read -r line; do     # IFS=: preserve spaces; -r: preserve backslashes
    echo "$line"
done < file.txt

# Read into multiple variables
echo "Alice 30 London" | IFS=' ' read -r name age city
echo "$name $age $city"         # "Alice 30 London"

# Read entire file into variable
content=$(< file.txt)           # fast: no cat subprocess
IFS= read -r -d '' content < file.txt  # -d '': read until null byte (whole file)
```

---

### printf vs echo

```bash
# ⚠️ echo quirks — avoid for general-purpose output
echo "hello\n"           # may print literal \n or newline (implementation-dependent)
echo "-n"                # may print "-n" or suppress newline (flag?)
echo "hello\t world"     # \t may or may not be interpreted

# ✅ printf — predictable, portable, powerful
printf '%s\n' "hello"          # always prints hello + newline
printf '%s\t%s\n' "a" "b"     # tab-separated
printf '%d\n' 42               # integer
printf '%.2f\n' 3.14159        # float with 2 decimal places
printf '%05d\n' 42             # zero-padded: "00042"
printf '%-20s|%s\n' "left" "right"   # left-aligned in 20-char field
printf '%20s|%s\n' "right" "left"    # right-aligned

# printf for multiple items
printf '%s\n' "${arr[@]}"      # print each array element on its own line

# printf to variable
printf -v result 'Hello, %s! You are %d years old.' "$name" "$age"
echo "$result"

# Safe echo for arbitrary strings
printf '%s\n' "$arbitrary_string"   # safe: no interpretation of content
echo -- "$arbitrary_string"         # echo -- helps with leading dashes
```

---

## 11. String Operations

### Length, Substring, Search

```bash
str="Hello, World!"

# Length
echo "${#str}"                 # 13

# Substring extraction
echo "${str:7}"                # "World!" — from index 7 to end
echo "${str:7:5}"              # "World" — 5 chars from index 7
echo "${str: -6}"              # "orld!" — last 6 (note space before -)

# "indexOf" — find position of substring (no built-in, use parameter expansion or expr)
# Method 1: expr
pos=$(expr index "$str" "W")   # 8 — position of first W (1-indexed)

# Method 2: pure bash — strip up to match, measure what's left
prefix="${str%%World*}"        # "Hello, " — everything before "World"
echo "${#prefix}"              # 7 — that's the index of "World"

# Contains substring
if [[ "$str" == *"World"* ]]; then     # glob match
    echo "contains World"
fi

# Starts with
if [[ "$str" == "Hello"* ]]; then
    echo "starts with Hello"
fi

# Ends with
if [[ "$str" == *"!" ]]; then
    echo "ends with !"
fi
```

---

### Trimming Whitespace

```bash
# Pure bash — no external commands needed

# Trim leading whitespace
ltrim() {
    local str="$1"
    echo "${str#"${str%%[! $'\t']*}"}"   # remove leading spaces/tabs
}

# Trim trailing whitespace
rtrim() {
    local str="$1"
    echo "${str%"${str##*[! $'\t']}"}"   # remove trailing spaces/tabs
}

# Trim both sides
trim() {
    local str="$1"
    str="${str#"${str%%[! $'\t']*}"}"    # leading
    str="${str%"${str##*[! $'\t']}"}"    # trailing
    echo "$str"
}

# Simpler method using read (trims both by default)
str="   hello world   "
trimmed=$(echo "$str" | xargs)           # xargs collapses whitespace too
echo "|$trimmed|"                         # "|hello world|"

# Read method (trims leading/trailing only)
IFS=' ' read -r trimmed <<< "   hello   "
echo "|$trimmed|"                         # "|hello|"
```

---

### String Splitting

```bash
# Split on delimiter using IFS
str="alice:bob:charlie"
IFS=: read -ra users <<< "$str"    # split into array on :
echo "${users[@]}"                  # "alice bob charlie"
echo "${#users[@]}"                  # 3

# Split using parameter expansion
str="one/two/three"
first="${str%%/*}"    # "one"  — everything before first /
rest="${str#*/}"      # "two/three" — everything after first /

# Split into individual characters
str="hello"
chars=()
for (( i=0; i<${#str}; i++ )); do
    chars+=("${str:$i:1}")
done
echo "${chars[@]}"    # "h e l l o"

# Split on whitespace (default IFS)
sentence="one two three"
read -ra words <<< "$sentence"
echo "${words[@]}"    # "one two three"

# Split string into lines
IFS=$'\n' read -r -d '' -a lines <<< "$multiline_str"
```

---

### Regex and BASH_REMATCH

```bash
# =~ operator in [[ — Extended Regular Expressions
if [[ "user@example.com" =~ ^([^@]+)@(.+)$ ]]; then
    echo "Full match: ${BASH_REMATCH[0]}"    # "user@example.com"
    echo "Username:   ${BASH_REMATCH[1]}"    # "user"
    echo "Domain:     ${BASH_REMATCH[2]}"    # "example.com"
fi

# Validate IP address
validate_ip() {
    local ip="$1"
    local octet="(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)"
    local re="^${octet}\\.${octet}\\.${octet}\\.${octet}$"
    [[ "$ip" =~ $re ]]   # return exit code
}
validate_ip "192.168.1.1" && echo "valid"
validate_ip "999.0.0.1"   || echo "invalid"

# Parse log line with regex
log_line='2024-01-15 10:30:45 ERROR Failed to connect to 192.168.1.100'
re='^([0-9-]+) ([0-9:]+) ([A-Z]+) (.+)$'
if [[ "$log_line" =~ $re ]]; then
    date="${BASH_REMATCH[1]}"
    time="${BASH_REMATCH[2]}"
    level="${BASH_REMATCH[3]}"
    message="${BASH_REMATCH[4]}"
fi

# Check if string is a valid integer
is_integer() {
    [[ "$1" =~ ^-?[0-9]+$ ]]
}
is_integer "42"   && echo "is integer"
is_integer "3.14" || echo "not integer"
is_integer "-5"   && echo "is integer"

# Check if string is a valid float
is_float() {
    [[ "$1" =~ ^-?[0-9]*\.?[0-9]+$ ]]
}
```

---

### Building Strings Efficiently

```bash
# 🔴 Anti-pattern: string concatenation in a loop (slow — creates new string each iteration)
result=""
for i in $(seq 1 1000); do
    result="${result}item${i},"    # slow: N string copies
done

# ✅ Right way: build array, join at end
parts=()
for i in $(seq 1 1000); do
    parts+=("item${i}")            # fast: append to array
done
result=$(IFS=,; echo "${parts[*]}")  # join once at the end

# Joining array with separator (various methods)
arr=(a b c d e)
IFS=, echo "${arr[*]}"              # "a,b,c,d,e" — temporarily set IFS
result=$(IFS=:; printf '%s' "${arr[*]}")  # "a:b:c:d:e"
printf '%s,' "${arr[@]}"            # "a,b,c,d,e," (trailing comma)
printf '%s\n' "${arr[@]}" | paste -sd,  # "a,b,c,d,e" (using paste)

# Multiline string with heredoc
multiline=$(cat <<'EOF'
Line 1
Line 2
Line 3
EOF
)
echo "$multiline"

# Repeat a string N times
repeat() {
    local str="$1" n="$2" result=""
    printf -v result '%*s' "$n"    # printf trick: n spaces
    echo "${result// /$str}"       # replace each space with str
}
repeat "=-" 5    # "=-=-=-=-=-"

# URL encoding (pure bash)
urlencode() {
    local string="$1"
    local encoded=""
    local pos c o
    for (( pos=0; pos<${#string}; pos++ )); do
        c="${string:$pos:1}"
        case "$c" in
            [-_.~a-zA-Z0-9]) encoded+="$c";;
            *) printf -v o '%%%02x' "'$c"; encoded+="$o";;
        esac
    done
    echo "$encoded"
}
urlencode "hello world & more"    # "hello%20world%20%26%20more"
```

## 12. Error Handling & Defensive Scripting

### set Options — The Golden Combination

```bash
#!/usr/bin/env bash
set -Eeuo pipefail   # the standard safety header for robust scripts

# What each flag does:
# -E  errtrace:  ERR trap is inherited by functions, subshells, and command substitutions
# -e  errexit:   exit immediately if any command returns non-zero
# -u  nounset:   treat unset variables as errors (prevents silent bugs)
# -o pipefail:   pipeline fails if ANY stage fails (not just the last one)

# Individual explanations:

set -e   # Exit on error — but has important exceptions:
true     # fine
false    # ← script exits here
         # DOES NOT trigger for: commands in if/while/until conditions,
         # commands followed by ||, commands in && chains, negated with !

set -u   # Unset variable error
echo "$UNDEFINED_VAR"   # error: UNDEFINED_VAR: unbound variable
echo "${MAYBE_UNSET:-}" # safe: default to empty string (bypasses -u)

set -o pipefail         # Pipeline failure detection
ls /nonexistent | wc -l  # without pipefail: exit 0 (wc succeeds)
                         # with pipefail:    exit 2 (ls fails)

set -E                   # ERR trap inheritance — critical for error handling in functions
set -C                   # noclobber: prevent > from overwriting existing files
echo "hi" > existing.txt # fails with -C set
echo "hi" >| existing.txt # >| forces overwrite even with -C
```

> ⚠️ **Warning:** `set -e` has many subtle non-triggering cases. Don't rely on it alone — combine with explicit error checking for critical commands.

---

### trap — Complete Guide

```bash
# trap 'command' SIGNAL [SIGNAL...]
# Registers a command to run when the shell receives a signal or event

# EXIT — runs when script exits for ANY reason (normal, error, signal)
cleanup() {
    local exit_code=$?                     # capture exit code BEFORE cleanup
    rm -f "$TMPFILE"                       # delete temp file
    [[ -f "$LOCKFILE" ]] && rm -f "$LOCKFILE"  # remove lock
    echo "Exiting with code: $exit_code" >&2
}
trap cleanup EXIT                          # cleanup always runs, whatever happens

# ERR — runs when a command fails (respects set -E for function inheritance)
on_error() {
    local exit_code=$?
    local line_no="${BASH_LINENO[0]}"
    local command="$BASH_COMMAND"          # the command that failed
    echo "ERROR: command '$command' failed with exit code $exit_code at line $line_no" >&2
}
trap on_error ERR

# INT — Ctrl+C
trap 'echo "Interrupted!"; exit 130' INT  # 128 + signal number (2) = 130

# TERM — kill command (default signal)
trap 'echo "Terminated"; cleanup; exit 143' TERM  # 128 + 15 = 143

# HUP — terminal hangup / reload signal
trap 'reload_config' HUP

# DEBUG — runs BEFORE every command (powerful for tracing)
trap 'echo "About to run: $BASH_COMMAND"' DEBUG

# RETURN — runs when a function returns (or sourced file finishes)
trap 'echo "Function returned"' RETURN

# Remove a trap
trap - EXIT    # reset EXIT trap to default behaviour

# List current traps
trap -p        # show all traps
trap -p EXIT   # show EXIT trap only
```

---

### trap Stacking

```bash
# Adding to an existing trap without replacing it
add_trap() {
    local new_cmd="$1"
    local signal="$2"
    local existing
    existing=$(trap -p "$signal" | awk -F"'" '{print $2}')  # get current handler
    if [[ -n "$existing" ]]; then
        trap "${existing}; ${new_cmd}" "$signal"  # append to existing
    else
        trap "$new_cmd" "$signal"                  # set new trap
    fi
}

trap 'echo "First handler"' EXIT
add_trap 'echo "Second handler"' EXIT    # both run on exit
add_trap 'rm -f /tmp/myfile' EXIT        # and this one too
```

---

### die(), warn(), and Stack Traces

```bash
# Standard error output helpers — always send to stderr (FD 2)
readonly PROGNAME="${BASH_SOURCE[0]##*/}"    # script name without path

warn() {
    echo "${PROGNAME}: WARNING: $*" >&2      # >&2 sends to stderr
}

die() {
    echo "${PROGNAME}: ERROR: $*" >&2
    exit 1
}

die_with_code() {
    local code=$1; shift
    echo "${PROGNAME}: ERROR: $*" >&2
    exit "$code"
}

# Stack trace — show call stack on error
stack_trace() {
    local i=0
    echo "Stack trace:" >&2
    while caller $i >/dev/null 2>&1; do     # caller prints "line func file"
        local line func file
        read -r line func file < <(caller $i)
        printf "  [%d] %s() called from %s line %d\n" "$i" "$func" "$file" "$line" >&2
        (( i++ ))
    done
}

on_error() {
    echo "ERROR at line ${BASH_LINENO[0]}: $BASH_COMMAND" >&2
    stack_trace
}
trap on_error ERR

# Line numbers in messages
log_error() {
    echo "Error at ${BASH_SOURCE[1]}:${BASH_LINENO[0]} in ${FUNCNAME[1]}: $*" >&2
}
```

---

### Input Validation Patterns

```bash
# Check required arguments
require_args() {
    local min=$1; shift
    if [[ $# -lt $min ]]; then
        die "Requires at least $min arguments, got $#"
    fi
}

# Validate file exists and is readable
require_file() {
    local file="$1"
    [[ -f "$file" ]] || die "File not found: $file"
    [[ -r "$file" ]] || die "File not readable: $file"
}

# Validate integer
require_int() {
    local val="$1" name="${2:-value}"
    [[ "$val" =~ ^-?[0-9]+$ ]] || die "$name must be an integer, got: $val"
}

# Allowlist validation (safest approach for untrusted input)
require_one_of() {
    local val="$1"; shift
    local allowed=("$@")
    local item
    for item in "${allowed[@]}"; do
        [[ "$val" == "$item" ]] && return 0
    done
    die "Invalid value '$val'. Allowed: ${allowed[*]}"
}
require_one_of "$env" "dev" "staging" "prod"

# Validate path (prevent path traversal)
require_safe_path() {
    local path="$1" base="$2"
    local real_base real_path
    real_base=$(realpath -m "$base")
    real_path=$(realpath -m "$path")
    [[ "$real_path" == "$real_base"/* ]] || die "Path traversal detected: $path"
}
```

---

### flock — File Locking

```bash
# Prevent concurrent execution of a script
LOCKFILE="/tmp/${PROGNAME}.lock"

# Method 1: flock with FD (releases automatically on script exit)
exec 200>"$LOCKFILE"             # open FD 200 for writing
flock -n 200 || die "Another instance is already running"
echo $$ > "$LOCKFILE"            # write our PID into the lock file
trap 'rm -f "$LOCKFILE"' EXIT    # clean up lock on exit

# Method 2: flock as command wrapper (simpler)
(
    flock -n 9 || die "Already running"
    echo "Doing exclusive work..."
) 9>"$LOCKFILE"                  # FD 9 tied to lockfile, released when subshell exits

# Method 3: timeout on lock acquisition
flock --wait 30 "$LOCKFILE" bash -c "do_exclusive_work"  # wait up to 30s
```

---

### Timeout Patterns

```bash
# timeout command — run command with a deadline
timeout 30 long_running_command              # SIGTERM after 30s
timeout --kill-after=5 30 command            # SIGTERM at 30s, SIGKILL at 35s
timeout 5s some_command; echo "exit: $?"     # suffix: s=sec, m=min, h=hour

# read -t for interactive timeouts
if read -t 10 -p "Continue? [y/N]: " answer; then
    echo "You said: $answer"
else
    echo "Timed out — defaulting to No"
    answer="n"
fi

# Manual timeout using background job
run_with_timeout() {
    local timeout=$1; shift
    "$@" &                          # run command in background
    local pid=$!
    (sleep "$timeout"; kill -TERM $pid 2>/dev/null) &  # watchdog in background
    local watchdog=$!
    wait $pid                        # wait for command
    local exit_code=$?
    kill $watchdog 2>/dev/null       # cancel watchdog
    wait $watchdog 2>/dev/null       # reap watchdog
    return $exit_code
}
run_with_timeout 30 slow_command
```

---

## 13. Script Structure & Best Practices

### Shebang and Header

```bash
#!/usr/bin/env bash
# ^^^^^^^^^^^^^^^^^^^^
# env bash: finds bash in PATH — more portable (works with pyenv, asdf, custom installs)
# /bin/bash: hardcoded path — may fail on systems where bash is elsewhere (e.g. FreeBSD)

# Script header convention
# ─────────────────────────────────────────────────────────────────────────────
# SCRIPT:  deploy.sh
# DESCRIPTION: Deploy application to target environment
# USAGE:   ./deploy.sh [OPTIONS] <environment>
# OPTIONS:
#   -v, --verbose    Enable verbose output
#   -n, --dry-run    Show what would be done without doing it
#   -h, --help       Show this help message
# REQUIRES: aws-cli, jq, curl
# AUTHOR:  Alice Smith <alice@example.com>
# VERSION: 2.1.0
# CREATED: 2024-01-15
# ─────────────────────────────────────────────────────────────────────────────

set -Eeuo pipefail
IFS=$'\n\t'    # safer word splitting: only newline and tab, not space
```

---

### usage() and Argument Parsing

```bash
usage() {
    cat <<EOF
Usage: $(basename "$0") [OPTIONS] ENVIRONMENT

Deploy the application to ENVIRONMENT (dev|staging|prod).

Options:
  -v, --verbose     Enable verbose output
  -n, --dry-run     Show actions without executing them
  -t, --tag TAG     Docker image tag to deploy (default: latest)
  -h, --help        Show this help message

Examples:
  $(basename "$0") dev
  $(basename "$0") --tag v1.2.3 prod
  $(basename "$0") --dry-run staging

EOF
}

# Manual while/case argument parsing — supports long options
VERBOSE=false
DRY_RUN=false
TAG="latest"

while [[ $# -gt 0 ]]; do        # while args remain
    case "$1" in
        -v|--verbose)
            VERBOSE=true
            shift;;               # consume this argument
        -n|--dry-run)
            DRY_RUN=true
            shift;;
        -t|--tag)
            TAG="${2:?--tag requires an argument}"   # error if missing
            shift 2;;             # consume flag AND its value
        -h|--help)
            usage
            exit 0;;
        --)
            shift; break;;        # -- signals end of options
        -*)
            die "Unknown option: $1";;
        *)
            break;;               # first non-option arg — stop parsing
    esac
done

ENVIRONMENT="${1:?Environment argument required. Usage: $(basename "$0") [OPTIONS] ENV}"
shift   # consume the positional argument

# getopts — POSIX-compliant, short options only
while getopts ":vnt:h" opt; do   # leading : = silent error mode
    case "$opt" in
        v) VERBOSE=true;;
        n) DRY_RUN=true;;
        t) TAG="$OPTARG";;
        h) usage; exit 0;;
        :) die "Option -$OPTARG requires an argument";;
        \?) die "Unknown option: -$OPTARG";;
    esac
done
shift $(( OPTIND - 1 ))          # remove all parsed options
```

---

### The main() Pattern

```bash
# Wrapping script in main() allows the file to be safely sourced for testing
# Without main(), all code runs when the file is sourced — dangerous!

main() {
    parse_args "$@"
    validate_env
    run_deploy
}

parse_args() { ... }
validate_env() { ... }
run_deploy() { ... }

# Only run main when executed directly (not when sourced)
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$@"    # BASH_SOURCE[0] = this file; $0 = executing script name
fi              # if they match, we're being executed directly

# Sourcing for testing:
# source deploy.sh                 # loads functions but doesn't execute main
# parse_args --dry-run staging     # can test functions in isolation
```

---

### Logging

```bash
# Log levels: DEBUG < INFO < WARN < ERROR
readonly LOG_LEVEL="${LOG_LEVEL:-INFO}"   # configurable via environment

# ANSI colour codes
RED=$'\e[31m'; YELLOW=$'\e[33m'; GREEN=$'\e[32m'; BLUE=$'\e[34m'; RESET=$'\e[0m'

# Detect if terminal supports colour
if [[ -t 2 ]] && [[ "$(tput colors 2>/dev/null)" -ge 8 ]]; then
    COLOUR=true
else
    COLOUR=false; RED=''; YELLOW=''; GREEN=''; BLUE=''; RESET=''
fi

log() {
    local level="$1"; shift
    local message="$*"
    local timestamp
    timestamp=$(date +'%Y-%m-%d %H:%M:%S')

    # Check level filter
    case "$level" in
        DEBUG) [[ "$LOG_LEVEL" == "DEBUG" ]] || return 0;;
        INFO)  [[ "$LOG_LEVEL" =~ ^(DEBUG|INFO)$ ]] || return 0;;
    esac

    local colour=""
    case "$level" in
        DEBUG) colour="$BLUE";;
        INFO)  colour="$GREEN";;
        WARN)  colour="$YELLOW";;
        ERROR) colour="$RED";;
    esac

    printf '%s [%s%-5s%s] %s\n' "$timestamp" "$colour" "$level" "$RESET" "$message" >&2
}

log_debug() { log "DEBUG" "$@"; }
log_info()  { log "INFO"  "$@"; }
log_warn()  { log "WARN"  "$@"; }
log_error() { log "ERROR" "$@"; }

# Tee log to file
LOG_FILE="/var/log/myapp/deploy.log"
exec > >(tee -a "$LOG_FILE") 2>&1   # all output goes to both terminal and file
```

---

### Dry-run Mode

```bash
DRY_RUN=false

# run wrapper: executes or pretends based on DRY_RUN
run() {
    if [[ "$DRY_RUN" == true ]]; then
        echo "[DRY-RUN] $*" >&2    # show what would run
    else
        "$@"                        # actually run the command
    fi
}

# Usage — wrap every mutating command with run()
run rm -rf /old/directory
run systemctl restart nginx
run aws s3 cp local.tar.gz s3://bucket/

# For commands that produce output needed by later code:
if [[ "$DRY_RUN" == true ]]; then
    echo "[DRY-RUN] Would create database"
    DB_ID="dry-run-fake-id"          # use placeholder value
else
    DB_ID=$(create_database)
fi
```

---

### Script Directory Detection

```bash
# Reliably find the script's own directory — works even if called from another dir
# or via a symlink

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
# BASH_SOURCE[0]: path of THIS script file (not $0, which may differ when sourced)
# dirname: extract directory component
# cd && pwd: resolve to absolute path (handles relative paths)

# If script might be a symlink and you want the REAL file's directory:
REAL_SCRIPT="$(readlink -f "${BASH_SOURCE[0]}")"   # resolve all symlinks
SCRIPT_DIR="$(dirname "$REAL_SCRIPT")"

# Source files relative to script location
source "${SCRIPT_DIR}/lib/utils.sh"
source "${SCRIPT_DIR}/../config/defaults.conf"
```

---

### Idempotency Patterns

```bash
# Idempotent: safe to run multiple times — same result every time

# Create user only if doesn't exist
if ! id "appuser" &>/dev/null; then    # check before creating
    useradd --system --no-create-home appuser
fi

# Create directory only if needed
mkdir -p /etc/myapp/conf.d             # -p: no error if exists

# Add line to file only if not already there
grep -qxF "export PATH=/opt/bin:\$PATH" ~/.bashrc || \
    echo 'export PATH=/opt/bin:$PATH' >> ~/.bashrc

# Install package only if missing
if ! command -v jq &>/dev/null; then   # check command exists
    apt-get install -y jq
fi

# Symlink only if target is different
if [[ "$(readlink -f /etc/myapp)" != "/opt/myapp-v2" ]]; then
    ln -sfn /opt/myapp-v2 /etc/myapp   # -f: force, -n: don't follow existing symlink
fi
```

---

## 14. Process Management

### Background Jobs and wait

```bash
# & — run command in background
long_task &             # start in background
pid=$!                  # capture PID of last background job
echo "Started PID $pid"

# jobs — list background jobs in current shell
jobs                    # show all jobs
jobs -l                 # with PIDs
jobs -p                 # PIDs only

# fg, bg — foreground/background control
fg %1                   # bring job 1 to foreground
bg %1                   # send stopped job to background
Ctrl+Z                  # suspend foreground job (sends SIGTSTP)

# disown — detach job from shell (won't be killed when shell exits)
long_task &
disown $!               # detach last background job
disown %1               # detach job 1
disown -a               # detach all jobs

# wait — collect exit codes from background processes
task1 & pid1=$!
task2 & pid2=$!
task3 & pid3=$!

wait $pid1; echo "task1 exit: $?"
wait $pid2; echo "task2 exit: $?"
wait $pid3; echo "task3 exit: $?"

wait        # wait for ALL background jobs
```

---

### Parallel Job Queue with Semaphore

```bash
# Run N jobs in parallel, wait for slot before launching next
MAX_JOBS=4
running=0

run_parallel() {
    local job_id=$1
    echo "Running job $job_id (PID $$)"
    sleep $(( RANDOM % 5 + 1 ))     # simulate work
}

pids=()
for i in $(seq 1 20); do
    run_parallel "$i" &              # launch job
    pids+=($!)                       # track PID

    (( ++running ))
    if (( running >= MAX_JOBS )); then
        wait "${pids[0]}"            # wait for oldest job
        pids=("${pids[@]:1}")        # remove it from list
        (( running-- ))
    fi
done

# Wait for all remaining jobs
for pid in "${pids[@]}"; do
    wait "$pid"
done
echo "All jobs complete"

# Simpler approach with xargs -P
printf '%s\n' {1..20} | xargs -P 4 -I{} bash -c 'echo "job {}"'
# -P 4: max 4 parallel processes
# -I{}: replace {} with input line
```

---

### Signals

```bash
# Send signals to processes
kill -TERM $pid          # graceful shutdown (SIGTERM = 15, default)
kill -KILL $pid          # force kill (SIGKILL = 9, unblockable)
kill -HUP  $pid          # reload config (SIGHUP = 1)
kill -USR1 $pid          # user-defined (SIGUSR1 = 10)
kill -0    $pid          # check if process exists (no signal sent)

# Kill by name
pkill nginx              # SIGTERM to all nginx processes
pkill -HUP sshd          # reload sshd config
killall python3          # kill all python3 processes

# Trap signals in scripts
graceful_shutdown() {
    log_info "Received TERM signal, shutting down..."
    # Stop accepting new work
    kill -TERM "${worker_pids[@]}" 2>/dev/null
    wait "${worker_pids[@]}" 2>/dev/null
    exit 0
}
trap graceful_shutdown TERM INT

# Signal to process group (negative PID = process group)
kill -TERM -$pid         # send TERM to entire process group
```

---

### Coprocesses

```bash
# coproc — create a bidirectional pipe to a subprocess
# Useful for long-running processes you want to send multiple requests to

# Basic coprocess
coproc bc                   # start bc as coprocess
echo "5 + 3" >&${COPROC[1]}  # write to its stdin (FD index 1)
read -u ${COPROC[0]} result  # read from its stdout (FD index 0)
echo "Result: $result"        # "8"

# Named coprocess
coproc worker { python3 -u worker.py; }
echo "task1" >&${worker[1]}
read -u ${worker[0]} response
echo "Response: $response"

# Close coprocess
exec ${worker[1]}>&-         # close its stdin (signals EOF)
wait $worker_PID             # wait for it to finish

# Named pipe (FIFO) alternative for IPC
mkfifo /tmp/pipe
writer() { echo "data" > /tmp/pipe; }
reader() { read data < /tmp/pipe; echo "Got: $data"; }
writer &
reader
rm /tmp/pipe
```

---

### Subshell Performance

```bash
# Every $() and () creates a fork — expensive in tight loops

# 🔴 Anti-pattern: command substitution in loop
for i in $(seq 1 1000); do
    result=$(echo "$i" | tr '0-9' 'a-j')   # TWO forks per iteration!
done

# ✅ Better: use built-ins and avoid subshells
for (( i=1; i<=1000; i++ )); do            # C-style loop: no fork
    result="${i//0/a}"                       # parameter expansion: no fork
done

# Compare timing
time for i in $(seq 1 1000); do x=$(expr $i + 1); done   # slow: many forks
time for (( i=1; i<=1000; i++ )); do (( x = i + 1 )); done  # fast: no forks

# {} grouping runs in current shell (no fork)
{ echo "a"; echo "b"; echo "c"; } > file.txt   # redirect entire group, no subshell

# () grouping creates subshell (fork)
( echo "a"; echo "b"; echo "c"; ) > file.txt   # same but in subshell
```

---

## 15. File & Path Operations

### Safe rm and Atomic Operations

```bash
# Safe rm — verify before deleting
safe_rm() {
    local path="$1"
    [[ -e "$path" ]] || { echo "Warning: $path does not exist"; return 1; }
    echo "Deleting: $path"
    rm -rf -- "$path"          # -- prevents option injection if path starts with -
}

# ⚠️ Warning: never build rm targets with unchecked variables
rm -rf "$dir/"*                # CATASTROPHIC if $dir is empty or "/"

# ✅ Always validate before rm
[[ -n "$dir" ]] || die "dir is empty!"
[[ "$dir" != "/" ]] || die "refusing to rm /"
rm -rf "${dir:?dir is unset}/"*   # :? exits if unset

# Atomic file writes (write then rename)
# rename() is atomic on the same filesystem — either old or new, never partial
write_atomic() {
    local target="$1"
    local tmpfile
    tmpfile=$(mktemp "${target}.XXXXXX")    # temp file in same directory
    trap "rm -f '$tmpfile'" EXIT            # clean up on error
    cat > "$tmpfile"                        # write content
    mv -f "$tmpfile" "$target"             # atomic rename
}
echo "new content" | write_atomic /etc/myapp/config.conf
```

---

### Iterating Files Safely

```bash
# The ONLY safe way to iterate files with arbitrary names (spaces, newlines, globs)

# Method 1: find -print0 | xargs -0
find /tmp -name "*.log" -print0 | xargs -0 -I{} process_file {}

# Method 2: find -print0 with while read -d ''
while IFS= read -r -d '' file; do   # -d '': delimiter is null byte
    echo "Processing: $file"        # safe: handles ANY filename
    process "$file"
done < <(find /tmp -name "*.log" -print0)

# Method 3: find -exec (simpler for single commands)
find /tmp -name "*.log" -exec gzip {} \;     # one process per file
find /tmp -name "*.log" -exec gzip {} +      # batch (faster)

# Method 4: globbing with nullglob (pure bash)
shopt -s nullglob                   # glob expands to nothing if no match
files=(/tmp/*.log)                  # array of matching files
[[ ${#files[@]} -gt 0 ]] || { echo "No log files"; exit 0; }
for f in "${files[@]}"; do          # always quote array expansion
    process "$f"
done

# 🔴 Anti-pattern: for f in $(ls) or for f in $(find ...)
for f in $(ls /tmp/*.log); do       # WRONG: breaks on spaces, globs, newlines
    process "$f"
done
```

---

### Path Manipulation

```bash
path="/home/alice/docs/report.tar.gz"

# External commands
basename "$path"              # "report.tar.gz"
basename "$path" .gz          # "report.tar"  — strip extension
dirname  "$path"              # "/home/alice/docs"

# Pure bash alternatives (no subshell fork!)
echo "${path##*/}"            # "report.tar.gz" — like basename
echo "${path%/*}"             # "/home/alice/docs" — like dirname

# Canonical/real path
realpath "$path"              # resolve symlinks, make absolute
readlink -f "$path"           # same, widely available
readlink -e "$path"           # same, but requires path to exist

# Pure bash realpath (no symlink resolution)
abs_path() {
    if [[ "$1" == /* ]]; then
        echo "$1"             # already absolute
    else
        echo "$(pwd)/$1"      # prepend current dir
    fi
}

# Extension manipulation
file="archive.tar.gz"
echo "${file%%.*}"            # "archive"  — strip all extensions
echo "${file%.*}"             # "archive.tar" — strip last extension
echo "${file##*.}"            # "gz" — last extension
echo "${file#*.}"             # "tar.gz" — all after first dot

# Path traversal check
is_safe_path() {
    local base realbase target realtarget
    base="$(realpath "$1")"
    target="$(realpath "$2")"
    [[ "$target" == "$base/"* || "$target" == "$base" ]]
}
is_safe_path /var/www /var/www/uploads/file.txt   && echo "safe"
is_safe_path /var/www /etc/passwd                  || echo "traversal attempt!"
```

---

### Reading Structured Files

```bash
# key=value config file
load_config() {
    local config_file="$1"
    [[ -f "$config_file" ]] || die "Config not found: $config_file"

    while IFS='=' read -r key value; do
        key="${key%%#*}"          # strip inline comments
        key="${key// /}"          # trim spaces from key
        value="${value%%#*}"      # strip comment from value
        value="${value%"${value##*[! ]}"}"  # trim trailing space
        [[ -z "$key" ]] && continue  # skip empty/comment lines
        printf -v "$key" '%s' "$value"   # set variable named $key
    done < "$config_file"
}

# INI file sections
parse_ini() {
    local file="$1"
    local section=""
    while IFS= read -r line; do
        case "$line" in
            \[*\])   section="${line:1:-1}";;   # extract section name
            *=*)     key="${line%%=*}"; val="${line#*=}"
                     printf -v "${section}_${key}" '%s' "$val";;
            \#*|"")  ;;   # skip comments and blank lines
        esac
    done < "$file"
}

# CSV — simple (no quoted fields with commas)
while IFS=, read -r name age city; do
    echo "Name: $name, Age: $age, City: $city"
done < data.csv

# stat — portable file metadata
get_mtime() { stat -c '%Y' "$1" 2>/dev/null || stat -f '%m' "$1"; }  # Linux/macOS
get_size()  { stat -c '%s' "$1" 2>/dev/null || stat -f '%z' "$1"; }
get_perms() { stat -c '%a' "$1" 2>/dev/null || stat -f '%OLp' "$1"; }
```

---

### Temporary Files and Directories

```bash
# mktemp — create temp file with unpredictable name
TMPFILE=$(mktemp)                          # /tmp/tmp.XkZj8f
TMPFILE=$(mktemp /tmp/myapp.XXXXXX)        # /tmp/myapp.q9Bx3A (X's replaced randomly)
TMPDIR_WORK=$(mktemp -d)                   # create temp DIRECTORY
TMPDIR_WORK=$(mktemp -d "${TMPDIR:-/tmp}/myapp.XXXXXX")

# Always clean up temp files — even on error
cleanup() {
    rm -f "$TMPFILE"
    rm -rf "$TMPDIR_WORK"
}
trap cleanup EXIT    # runs on any exit: normal, error, signal

# Write to temp then atomically replace target
generate_config() {
    local target="$1"
    local tmp
    tmp=$(mktemp "$(dirname "$target")/config.XXXXXX")  # same filesystem as target!
    trap "rm -f '$tmp'" RETURN    # clean up if function returns early
    generate_content > "$tmp"
    mv "$tmp" "$target"           # atomic rename — readers always see complete file
}
```

---

## 16. Text Processing in Scripts

### Decision Matrix — Bash vs External Tools

```
Use BASH built-ins/parameter expansion when:
  ✅ Simple string manipulation (prefix/suffix removal, substitution)
  ✅ Checking if string contains/starts with/ends with
  ✅ Splitting on single fixed delimiter
  ✅ Processing < 100 items (loop performance acceptable)

Use grep when:
  ✅ Testing if pattern exists in output (exit code)
  ✅ Filtering lines by regex
  ✅ Counting matches

Use sed when:
  ✅ Stream editing: substitutions, deletions on text streams
  ✅ In-place file editing

Use awk when:
  ✅ Field-based processing (columns, CSV-like data)
  ✅ Arithmetic on fields
  ✅ Accumulating totals, counts, summaries
  ✅ Processing > 1000 lines (much faster than bash loops)

Use Python/Perl when:
  ✅ Complex data structures
  ✅ JSON/YAML/XML processing
  ✅ Multi-line regex
  ✅ Error handling is critical
```

---

### grep in Scripts

```bash
# grep for testing (exit code usage)
if grep -q "pattern" file.txt; then    # -q: quiet — no output, just exit code
    echo "found"
fi

# grep -c for counting (returns count as output)
count=$(grep -c "ERROR" /var/log/app.log)
echo "Found $count errors"

# Combining: check then count
if grep -q "ERROR" /var/log/app.log; then
    count=$(grep -c "ERROR" /var/log/app.log)
    warn "Found $count errors in log"
fi

# 🔴 Anti-pattern: parsing grep output with cut/awk when -o or groups are cleaner
line=$(grep "key=" config.txt)
value="${line#*=}"    # parameter expansion to extract value

# ✅ Cleaner: use grep -o or perl regex
value=$(grep -oP 'key=\K.*' config.txt)    # -P PCRE, \K = reset match start
value=$(grep -m1 "^key=" config.txt | cut -d= -f2-)  # first match, field 2+

# Grep in CI pipelines — fail on unexpected output
grep -q "PASSED" test_results.txt || die "Tests did not pass"
```

---

### Passing Variables into awk and sed

```bash
# ⚠️ Never interpolate shell variables directly into awk/sed code — injection risk!

# 🔴 Anti-pattern: interpolation (breaks if var contains special chars)
pattern="hello.world"
sed "s/$pattern/replaced/g" file.txt    # . in regex matches any char!

# ✅ sed: use -e with properly escaped value, or pass via env
# For literal strings in sed:
sed "s/$(printf '%s\n' "$pattern" | sed 's/[[\.*^$()+?{|]/\\&/g')/replaced/g" file.txt
# Simpler: use fgrep/grep -F for literal matching, or perl for complex cases

# ✅ awk: use -v flag to pass shell variables into awk
name="Alice"
threshold=100
awk -v name="$name" -v thresh="$threshold" '
    $1 == name && $2 > thresh { print "Match:", $0 }
' data.txt

# awk with ENVIRON — access shell env vars
export THRESHOLD=100
awk '$2 > ENVIRON["THRESHOLD"] { print }' data.txt

# Passing arrays to awk (join to string first)
IFS=: arr_str="${myarray[*]}"
awk -v data="$arr_str" 'BEGIN { n=split(data, arr, ":") } ...' file.txt
```

---

### Processing JSON with jq

```bash
# jq — JSON processor (install: apt install jq)

# Basic extraction
echo '{"name":"Alice","age":30}' | jq '.name'         # "Alice"
echo '{"name":"Alice","age":30}' | jq -r '.name'       # Alice (raw, no quotes)

# Array operations
curl -s api/users | jq '.[].name'                      # extract name from each
curl -s api/users | jq '[.[] | select(.active==true)]' # filter array
curl -s api/users | jq 'length'                        # count elements
curl -s api/users | jq '.[] | {name, email}'           # reshape objects

# Error handling with jq in scripts
get_user_name() {
    local json="$1"
    local name
    # jq exits non-zero if JSON is invalid
    name=$(jq -e -r '.name' <<< "$json" 2>/dev/null) || {
        warn "Invalid JSON or missing .name field"
        return 1
    }
    echo "$name"
}

# Build JSON with jq (avoid string concatenation)
jq -n \
    --arg name "$name" \          # pass as string
    --argjson age "$age" \        # pass as JSON number
    --arg env "$ENVIRONMENT" \
    '{"name": $name, "age": $age, "env": $env}'

# Iterate over jq output safely
while IFS= read -r item; do
    process_item "$item"
done < <(jq -c '.[]' data.json)  # -c: compact (one JSON object per line)
```

---

### Log Processing Patterns

```bash
# Count errors per minute from nginx log
# Format: 1.2.3.4 - - [01/Jan/2024:10:30:45 +0000] "GET /" 500 ...

# Frequency count by status code
awk '{print $9}' access.log | sort | uniq -c | sort -rn
# Output: 1234 200, 45 404, 12 500

# Errors in last hour
awk -v cutoff="$(date -d '1 hour ago' +'%d/%b/%Y:%H')" \
    '$4 ~ cutoff && $9 >= 500' access.log | wc -l

# Top 10 IPs
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# Summarise errors with context
grep "ERROR" app.log |
    awk -F' ERROR ' '{print $2}' |   # extract message after ERROR
    sort | uniq -c | sort -rn |      # count and sort
    head -20                          # top 20 error messages

# Generate report from structured log
generate_report() {
    local logfile="$1"
    cat <<EOF
=== Daily Report $(date +'%Y-%m-%d') ===
Total requests:  $(wc -l < "$logfile")
Error count:     $(grep -c " 5[0-9][0-9] " "$logfile")
Unique IPs:      $(awk '{print $1}' "$logfile" | sort -u | wc -l)
Top error URL:   $(awk '$9 >= 500 {print $7}' "$logfile" | sort | uniq -c | sort -rn | head -1)
EOF
}
```

---

### Generating Structured Output

```bash
# Generate JSON without jq (simple cases)
json_object() {
    local name="$1" version="$2" status="$3"
    # Escape special characters in values
    name="${name//\\/\\\\}"; name="${name//\"/\\\"}"
    printf '{"name":"%s","version":"%s","status":"%s"}\n' \
        "$name" "$version" "$status"
}
json_object "my app" "1.0.0" "running"

# Generate CSV with printf
print_csv_header() {
    printf '%s,%s,%s\n' "name" "age" "city"
}
print_csv_row() {
    local name="$1" age="$2" city="$3"
    # Wrap in quotes if value contains comma
    [[ "$name" == *,* ]] && name="\"$name\""
    printf '%s,%s,%s\n' "$name" "$age" "$city"
}

# Heredoc for config file generation
generate_nginx_config() {
    local domain="$1" port="$2"
    cat > "/etc/nginx/sites-available/${domain}.conf" <<EOF
server {
    listen 80;
    server_name ${domain};

    location / {
        proxy_pass http://localhost:${port};
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
    }
}
EOF
}
```

## 17. Advanced Features

### eval — Dangers and Alternatives

```bash
# eval — executes its arguments as a shell command AFTER expansion
# This means double-expansion: variables are expanded TWICE

var="hello"
eval "echo $var"        # echoes "hello"

# 🔴 Why eval is dangerous
user_input="world; rm -rf /"
eval "echo $user_input"   # EXECUTES rm -rf / — command injection!

# ✅ Alternative 1: declare -n (nameref) — avoids eval for dynamic var names
# Instead of: eval "${varname}=value"
declare -n dynamic_ref="$varname"
dynamic_ref="value"                  # sets the variable named by $varname

# ✅ Alternative 2: printf -v — assign to variable by name
printf -v "$varname" '%s' "value"    # safe: no execution

# ✅ Alternative 3: associative arrays — avoid dynamic variable names entirely
declare -A config
config["$key"]="$value"              # use key as index
echo "${config[$key]}"

# ✅ Alternative 4: indirect expansion ${!var}
varname="PATH"
echo "${!varname}"                   # expands $PATH — read-only, safe

# When eval IS acceptable (trusted, controlled input):
# Expanding aliases in scripts, sourcing trusted env files
eval "$(some-tool --print-env)"      # e.g., aws-vault, direnv — trusted output
```

---

### shopt — Shell Options

```bash
shopt                         # list all options with on/off status
shopt -s option               # set (enable) option
shopt -u option               # unset (disable) option
shopt -q option               # quiet check — returns exit code

# Globbing options
shopt -s globstar             # ** matches any path (including subdirs)
ls **/*.py                    # all .py files recursively

shopt -s nullglob             # glob expands to nothing if no match (instead of literal)
for f in /tmp/*.log; do       # without nullglob: if no .log files, $f = "/tmp/*.log"
    process "$f"              # with nullglob: loop body never runs if no files
done

shopt -s failglob             # glob that matches nothing = error (instead of literal)
shopt -s extglob              # enable extended glob patterns (see below)
shopt -s dotglob              # * matches .dotfiles too
shopt -s nocaseglob           # case-insensitive glob
shopt -s nocasematch          # case-insensitive [[ == ]] and [[ =~ ]]

# Behaviour options
shopt -s lastpipe             # last command in pipeline runs in current shell
shopt -s inherit_errexit      # subshells inherit -e (important with set -e)
shopt -s checkwinsize         # update LINES/COLUMNS after each command
shopt -s histappend           # append to history file, don't overwrite
shopt -s cmdhist              # save multi-line commands as single history entry
shopt -s lithist              # preserve newlines in multi-line history
```

---

### Extended Globbing

```bash
shopt -s extglob    # must enable first

# Pattern syntax:
# ?(pat)  — zero or one occurrence of pat
# *(pat)  — zero or more occurrences of pat
# +(pat)  — one or more occurrences of pat
# @(pat)  — exactly one occurrence of pat
# !(pat)  — anything EXCEPT pat

# Examples:
ls ?(foo)           # matches "" or "foo"
ls *(foo)           # matches "", "foo", "foofoo", "foofoofoo"
ls +(foo)           # matches "foo", "foofoo", "foofoofoo" (not empty)
ls @(foo|bar)       # matches exactly "foo" or "bar"
ls !(*.txt)         # matches everything EXCEPT .txt files

# Practical uses:
rm !(important.txt)           # delete everything except important.txt
ls +(*.jpg|*.png|*.gif)       # match any image file
case "$arg" in
    --@(verbose|debug)) verbose=true;;   # match either --verbose or --debug
esac

# Extended glob in parameter expansion
var="   hello   "
trimmed="${var#+( )}"    # remove leading spaces (+ = one or more spaces)
```

---

### Brace Expansion

```bash
# Brace expansion happens BEFORE other expansions — always expands
echo {a,b,c}                   # "a b c"
echo {1..5}                    # "1 2 3 4 5"
echo {01..10}                  # "01 02 03 04 05 06 07 08 09 10" (zero-padded)
echo {a..z}                    # "a b c d e f g h i j k l m n o p q r s t u v w x y z"
echo {A..Z}                    # uppercase alphabet
echo {1..10..2}                # "1 3 5 7 9" — step size 2

# Creating directory structures
mkdir -p project/{src,tests,docs,bin,config}
mkdir -p app/{frontend,backend}/{src,tests}

# File operations
cp file.txt{,.bak}             # copy file.txt to file.txt.bak
mv config.yml{,.old}           # rename config.yml to config.yml.old

# Nested braces
echo {a,b}{1,2}                # "a1 a2 b1 b2"
echo {x,y}{a,b}{1,2}           # "xa1 xa2 xb1 xb2 ya1 ya2 yb1 yb2"

# Sequence with variable (doesn't work with variables directly!)
end=5
echo {1..$end}                 # 🔴 WRONG: "{1..5}" — brace expansion is early!
seq 1 "$end"                   # ✅ use seq for variable-based ranges
eval echo "{1..$end}"          # ✅ eval workaround (use sparingly)
```

---

### Bash Special Variables Reference

```bash
# Script and shell info
echo "$BASH_VERSION"           # "5.2.15(1)-release"
echo "$BASH_VERSINFO[@]"       # (5 2 15 1 release x86_64-pc-linux-gnu)
echo "$BASHPID"                # PID of current bash process
echo "$BASH_SUBSHELL"          # depth of subshell nesting (0 = main shell)
echo "$BASH_COMMAND"           # command currently being executed
echo "$LINENO"                 # current line number in script

# Function/source tracking arrays
echo "${FUNCNAME[@]}"          # call stack: current function, its caller, ...
echo "${BASH_SOURCE[@]}"       # source files: current file, its caller's file, ...
echo "${BASH_LINENO[@]}"       # line numbers corresponding to FUNCNAME

# Example — print call stack
print_stack() {
    local i
    for (( i=1; i<${#FUNCNAME[@]}; i++ )); do
        printf "  %s() at %s:%d\n" \
            "${FUNCNAME[$i]}" "${BASH_SOURCE[$i]}" "${BASH_LINENO[$i-1]}"
    done
}

# Timing and profiling
echo "$SECONDS"                # seconds since shell started
echo "$EPOCHSECONDS"           # Unix timestamp (bash 5.0+)
echo "$EPOCHREALTIME"          # Unix timestamp with microseconds (bash 5.0+)

# History variables
echo "$HISTFILE"               # path to history file
echo "$HISTSIZE"               # max history entries in memory
echo "$HISTFILESIZE"           # max history entries in file
export HISTTIMEFORMAT="%Y-%m-%d %T "   # timestamps in history
export HISTCONTROL=ignoredups:erasedups # deduplicate history

# Terminal
echo "$TERM"                   # terminal type: xterm-256color, etc.
echo "$COLUMNS"                # terminal width
echo "$LINES"                  # terminal height

# REPLY — default variable for read and select
read -p "Enter value: "        # stored in $REPLY without naming a var
select item in a b c; do
    echo "$REPLY"              # the number the user typed
    break
done
```

---

### PS1, PS4, and Prompt Customisation

```bash
# PS4 — the prefix for set -x trace output (default: "+ ")
export PS4='+(${BASH_SOURCE}:${LINENO}): ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'
set -x
# Now xtrace shows: +(script.sh:42): myfunc(): the command here

# Coloured PS4 for readability
export PS4=$'\e[33m+ \e[0m\e[2m${BASH_SOURCE}:${LINENO}\e[0m '

# BASH_XTRACEFD — send xtrace to a file (keep stderr clean)
exec 9>/tmp/trace.log          # open FD 9 for trace output
export BASH_XTRACEFD=9         # send xtrace to FD 9
set -x                         # now trace goes to /tmp/trace.log, not stderr

# PS1 — primary prompt (for reference)
export PS1='\[\e[32m\]\u@\h\[\e[0m\]:\[\e[34m\]\w\[\e[0m\]\$ '
# \u = username, \h = hostname, \w = working dir
```

---

## 18. Debugging & Profiling

### Tracing and Dry-Run

```bash
# bash -x — trace from outside (no script modification needed)
bash -x script.sh              # trace all commands
bash -x script.sh 2> trace.log # save trace to file

# bash -n — syntax check only (parse but don't execute)
bash -n script.sh              # check for syntax errors
bash -n *.sh                   # check all scripts

# bash -v — verbose: print each line as it's read
bash -v script.sh

# set -x inside script — trace specific sections
set -x    # enable tracing
my_complex_function
set +x    # disable tracing

# Conditional tracing
[[ "${DEBUG:-false}" == "true" ]] && set -x

# Trace with informative PS4
PS4='+ ${BASH_SOURCE}:${LINENO} ${FUNCNAME[0]:+[${FUNCNAME[0]}]} '
set -x
```

---

### trap DEBUG and Assertions

```bash
# trap DEBUG — runs BEFORE every command
# Useful for stepping through scripts interactively
step_through() {
    trap 'echo "  → $BASH_COMMAND"; read -p "  [Enter to continue] " < /dev/tty' DEBUG
}
# Call step_through to enable interactive stepping

# Assertion function
assert() {
    local description="$1"
    shift
    if ! "$@" 2>/dev/null; then      # run the condition command
        echo "ASSERTION FAILED: $description" >&2
        echo "  Command: $*" >&2
        echo "  At: ${BASH_SOURCE[1]}:${BASH_LINENO[0]}" >&2
        exit 1
    fi
}

# Usage:
assert "directory exists" [[ -d "/var/www" ]]
assert "count is positive" (( count > 0 ))
assert "file is readable" test -r "$config_file"

# declare -p — inspect variable state mid-script
debug_vars() {
    for var in "$@"; do
        declare -p "$var" >&2 2>/dev/null || echo "  $var: (unset)" >&2
    done
}
debug_vars PATH HOME MYVAR myarray
```

---

### Profiling

```bash
# Time a block of code
{ time sleep 0.1; } 2>&1       # runs sleep, shows real/user/sys times

# Profile with EPOCHREALTIME (bash 5.0+)
profile_start() {
    _PROFILE_START="$EPOCHREALTIME"
}
profile_end() {
    local label="$1"
    local elapsed
    elapsed=$(awk "BEGIN { printf \"%.3f\", $EPOCHREALTIME - $_PROFILE_START }")
    echo "PROFILE [$label]: ${elapsed}s" >&2
}

profile_start
expensive_function
profile_end "expensive_function"

# SECONDS built-in — integer seconds since shell started
start=$SECONDS
do_work
echo "Elapsed: $(( SECONDS - start ))s"

# Trace with timestamps in PS4
PS4='+[$(date "+%H:%M:%S.%3N")] '   # timestamp each traced command
set -x

# shellcheck — most important checks
# SC2086: Double quote to prevent word splitting: echo $var → echo "$var"
# SC2046: Quote to prevent word splitting: cmd $(subshell) → cmd "$(subshell)"
# SC2006: Use $() instead of backticks
# SC2164: Use cd ... || exit (cd failure should not continue)
# SC2155: Declare and assign separately: local x=$(cmd) → local x; x=$(cmd)
# SC2166: Prefer [[ ]] over [ ] in bash
# SC2129: Consider using { cmd1; cmd2; } >> file instead of individual redirects

# Inline shellcheck disable (use sparingly)
# shellcheck disable=SC2086
echo $intentionally_unsplit_word
```

---

## 19. Security

### Injection Vulnerabilities

```bash
# Command injection via unquoted variables
user_input="malicious; rm -rf ~"
ls $user_input              # 🔴 EXECUTES rm -rf ~!
ls "$user_input"            # ✅ passes as single argument — ls fails safely

# eval injection
eval "echo $user_input"     # 🔴 executes injected commands
eval "echo '$user_input'"   # ⚠️ still dangerous if input contains single quotes
printf '%s\n' "$user_input" # ✅ safe — no evaluation

# Filename injection — files starting with - treated as options
touch -- "-rf /"            # create a file named "-rf /"
rm $dangerous_filename      # 🔴 rm -rf / !!!
rm -- "$dangerous_filename" # ✅ -- signals end of options
find . -name "$pattern"     # ✅ find handles this safely
```

---

### Secrets Management

```bash
# ✅ Secrets via environment variables
export DB_PASSWORD="secret"    # set before running script
echo "$DB_PASSWORD"            # use without exposing in command args

# ✅ Secrets via file (restricted permissions)
SECRET=$(< /run/secrets/db_password)   # read from secrets file
chmod 600 /run/secrets/db_password     # owner-only access

# 🔴 Never hardcode secrets
DB_PASS="mysecret123"          # in version control — disaster!

# 🔴 Never echo secrets (appears in logs, ps output)
echo "Password: $DB_PASSWORD"                          # in logs!
mysql -u root -p"$DB_PASSWORD"                        # appears in ps aux!

# ✅ Use stdin for passwords where possible
echo "$DB_PASSWORD" | mysql -u root --password=/dev/stdin
mysql -u root --password="$DB_PASSWORD"               # still in process list
MYSQL_PWD="$DB_PASSWORD" mysql -u root                # env var — some tools support

# Prevent secrets from appearing in xtrace log
set +x                         # disable tracing around sensitive operations
SECRET=$(get_secret_from_vault)
set -x                         # re-enable

# Mask in PS4/DEBUG (overwrite after use)
get_password() {
    local password
    password=$(vault read -field=password secret/db)
    echo "$password"
    return
}
password=$(get_password)
# Clear from memory when done
unset password
```

---

### Input Sanitisation

```bash
# Allowlist approach — only accept known-safe values
sanitise_env() {
    local val="$1"
    case "$val" in
        dev|staging|prod) echo "$val";;    # known good values
        *) die "Invalid environment: $val";;
    esac
}

# Strip dangerous characters from filenames
sanitise_filename() {
    local name="$1"
    name="${name//[^a-zA-Z0-9._-]/_}"    # replace anything not alphanumeric/.-_ with _
    name="${name#.}"                       # remove leading dot
    echo "$name"
}

# Validate and sanitise numeric input
sanitise_int() {
    local val="$1" min="${2:--99999}" max="${3:-99999}"
    [[ "$val" =~ ^-?[0-9]+$ ]] || die "Not an integer: $val"
    (( val >= min && val <= max )) || die "Out of range [$min,$max]: $val"
    echo "$val"
}

# IFS safety — reset at script start (prevent inherited IFS manipulation)
IFS=$' \t\n'                   # reset to known-safe default

# Secure coding checklist
# ✅ Quote all variable expansions: "$var"
# ✅ Use [[ ]] instead of [ ]
# ✅ Set -Eeuo pipefail
# ✅ Use mktemp for temp files
# ✅ Use -- to end options: rm -- "$file"
# ✅ Validate all external input
# ✅ Never use eval with untrusted data
# ✅ Store secrets in env/files, not in script
# ✅ Use set +x around secret operations
# ✅ Run shellcheck in CI
# ✅ Set restrictive umask: umask 077
# ✅ Use absolute paths for critical commands in privileged scripts
```

---

## 20. Real-World Script Patterns & Recipes

### Robust Script Template

```bash
#!/usr/bin/env bash
# ─────────────────────────────────────────────────────────────────────────────
# DESCRIPTION: Production-ready bash script template
# USAGE:       ./template.sh [OPTIONS] REQUIRED_ARG
# ─────────────────────────────────────────────────────────────────────────────
set -Eeuo pipefail
IFS=$'\n\t'

# ── Constants ──────────────────────────────────────────────────────────────
readonly VERSION="1.0.0"
readonly SCRIPT_NAME="${BASH_SOURCE[0]##*/}"
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# ── Colours ────────────────────────────────────────────────────────────────
if [[ -t 2 ]] && [[ "$(tput colors 2>/dev/null || echo 0)" -ge 8 ]]; then
    RED=$'\e[31m' GREEN=$'\e[32m' YELLOW=$'\e[33m' BLUE=$'\e[34m' RESET=$'\e[0m' BOLD=$'\e[1m'
else
    RED='' GREEN='' YELLOW='' BLUE='' RESET='' BOLD=''
fi

# ── Logging ────────────────────────────────────────────────────────────────
log()  { printf '%s [%sINFO%s]  %s\n'  "$(date +'%H:%M:%S')" "$GREEN"  "$RESET" "$*" >&2; }
warn() { printf '%s [%sWARN%s]  %s\n'  "$(date +'%H:%M:%S')" "$YELLOW" "$RESET" "$*" >&2; }
err()  { printf '%s [%sERROR%s] %s\n'  "$(date +'%H:%M:%S')" "$RED"    "$RESET" "$*" >&2; }
die()  { err "$@"; exit 1; }

# ── Cleanup ────────────────────────────────────────────────────────────────
TMPDIR_WORK=""
cleanup() {
    local exit_code=$?
    [[ -n "$TMPDIR_WORK" ]] && rm -rf "$TMPDIR_WORK"
    exit "$exit_code"
}
trap cleanup EXIT
trap 'die "Interrupted"' INT TERM

# ── Usage ──────────────────────────────────────────────────────────────────
usage() {
    cat <<EOF
${BOLD}Usage:${RESET} $SCRIPT_NAME [OPTIONS] REQUIRED_ARG

${BOLD}Options:${RESET}
  -v, --verbose     Enable verbose output
  -n, --dry-run     Show actions without executing
  -h, --help        Show this help

${BOLD}Version:${RESET} $VERSION
EOF
}

# ── Argument parsing ───────────────────────────────────────────────────────
VERBOSE=false
DRY_RUN=false

parse_args() {
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -v|--verbose) VERBOSE=true; shift;;
            -n|--dry-run) DRY_RUN=true; shift;;
            -h|--help)    usage; exit 0;;
            --)           shift; break;;
            -*)           die "Unknown option: $1";;
            *)            break;;
        esac
    done
    REQUIRED_ARG="${1:?Argument required. Run with --help for usage.}"
}

run() { "$DRY_RUN" && log "[DRY-RUN] $*" || "$@"; }

# ── Main ───────────────────────────────────────────────────────────────────
main() {
    parse_args "$@"
    TMPDIR_WORK=$(mktemp -d)
    log "Starting with arg: $REQUIRED_ARG"
    [[ "$VERBOSE" == true ]] && set -x
    # ... your script logic here ...
    log "Done"
}

[[ "${BASH_SOURCE[0]}" == "${0}" ]] && main "$@"
```

---

### Retry with Exponential Backoff

```bash
# Retry a command with exponential backoff
retry() {
    local max_attempts="${1:-5}"
    local initial_delay="${2:-1}"
    shift 2
    local cmd=("$@")
    local attempt=1
    local delay="$initial_delay"

    while (( attempt <= max_attempts )); do
        log "Attempt $attempt/$max_attempts: ${cmd[*]}"
        if "${cmd[@]}"; then
            return 0                              # success!
        fi
        if (( attempt == max_attempts )); then
            err "Command failed after $max_attempts attempts"
            return 1
        fi
        warn "Failed. Retrying in ${delay}s..."
        sleep "$delay"
        (( delay = delay * 2 ))                  # exponential backoff
        (( attempt++ ))
    done
}

# Usage:
retry 5 1 curl -sf https://api.example.com/health
retry 3 2 rsync -av src/ dest/
```

---

### Lock File Pattern

```bash
# Prevent multiple concurrent instances of a script
readonly LOCKFILE="/var/run/${SCRIPT_NAME%.sh}.pid"

acquire_lock() {
    exec 200>"$LOCKFILE"                          # open FD 200 for lockfile
    if ! flock -n 200; then                       # try to acquire exclusive lock
        local existing_pid
        existing_pid=$(<"$LOCKFILE")
        die "Already running as PID $existing_pid"
    fi
    echo $$ > "$LOCKFILE"                         # write our PID
    trap 'rm -f "$LOCKFILE"' EXIT                 # release on exit
}

acquire_lock
log "Lock acquired, running..."
```

---

### Dependency Checking

```bash
# Verify required commands exist before script starts
check_deps() {
    local missing=()
    local cmd
    for cmd in "$@"; do
        command -v "$cmd" &>/dev/null || missing+=("$cmd")
    done
    if (( ${#missing[@]} > 0 )); then
        err "Missing required commands: ${missing[*]}"
        err "Install with: apt-get install ${missing[*]}"
        return 1
    fi
}

check_deps jq curl aws git docker
check_deps awk sed grep find   # these should always exist, but useful in CI
```

---

### Progress Bar

```bash
# Pure bash progress bar
progress_bar() {
    local current="$1" total="$2" label="${3:-Progress}"
    local width=40
    local filled=$(( current * width / total ))
    local empty=$(( width - filled ))
    local pct=$(( current * 100 / total ))
    local bar
    printf -v bar '%*s' "$filled" ''
    bar="${bar// /#}"
    printf '\r%s: [%-*s] %3d%%' "$label" "$width" "$bar" "$pct" >&2
    (( current == total )) && echo >&2   # newline when done
}

# Usage:
total=100
for (( i=1; i<=total; i++ )); do
    do_work "$i"
    progress_bar "$i" "$total" "Processing"
done
```

---

### Spinner

```bash
# Animated spinner for long-running commands
spinner() {
    local pid=$1 msg="${2:-Working...}"
    local frames=('⠋' '⠙' '⠹' '⠸' '⠼' '⠴' '⠦' '⠧' '⠇' '⠏')
    local i=0
    while kill -0 "$pid" 2>/dev/null; do   # while process is running
        printf '\r%s %s' "${frames[$i]}" "$msg" >&2
        (( i = (i + 1) % ${#frames[@]} ))
        sleep 0.1
    done
    printf '\r✓ %s\n' "$msg" >&2           # done message
}

# Usage:
long_command &
spinner $! "Deploying application..."
wait $!   # collect exit code
```

---

### Config File Pattern with Validation

```bash
# Load config file with defaults and validation
load_config() {
    local config_file="${1:-$SCRIPT_DIR/config.conf}"

    # Defaults
    DB_HOST="localhost"
    DB_PORT=5432
    DB_NAME="myapp"
    LOG_LEVEL="INFO"

    # Load config file if it exists
    if [[ -f "$config_file" ]]; then
        log "Loading config: $config_file"
        # Only allow key=value lines (no arbitrary code execution)
        while IFS='=' read -r key value; do
            key="${key%%#*}"                    # strip comments
            key="$(echo "$key" | tr -d ' ')"   # strip spaces
            [[ -z "$key" ]] && continue
            # Allowlist of accepted config keys
            case "$key" in
                DB_HOST|DB_PORT|DB_NAME|LOG_LEVEL)
                    printf -v "$key" '%s' "${value%%#*}";;  # assign variable
                *)  warn "Unknown config key: $key (ignored)";;
            esac
        done < "$config_file"
    fi

    # Validate
    [[ -n "$DB_HOST" ]] || die "DB_HOST is required"
    [[ "$DB_PORT" =~ ^[0-9]+$ ]] || die "DB_PORT must be a number"
    [[ "$LOG_LEVEL" =~ ^(DEBUG|INFO|WARN|ERROR)$ ]] || die "Invalid LOG_LEVEL: $LOG_LEVEL"

    log "Config loaded: DB=${DB_HOST}:${DB_PORT}/${DB_NAME}"
}
```

---

### Plugin System

```bash
# Load all scripts from a plugins directory
load_plugins() {
    local plugin_dir="${1:-$SCRIPT_DIR/plugins}"
    [[ -d "$plugin_dir" ]] || return 0

    local plugin
    for plugin in "$plugin_dir"/*.sh; do
        [[ -f "$plugin" ]] || continue          # skip if no .sh files
        log "Loading plugin: ${plugin##*/}"
        # shellcheck source=/dev/null
        source "$plugin" || warn "Failed to load plugin: $plugin"
    done
}

# Each plugin can define hooks
# plugins/notify.sh:
# on_deploy_start() { send_slack_message "Deploy started"; }
# on_deploy_end()   { send_slack_message "Deploy complete"; }

# Call hooks if they exist
call_hook() {
    local hook="$1"; shift
    if declare -f "$hook" > /dev/null 2>&1; then   # check function exists
        log "Calling hook: $hook"
        "$hook" "$@"
    fi
}
call_hook "on_deploy_start" "$ENVIRONMENT"
deploy
call_hook "on_deploy_end" "$ENVIRONMENT" "$?"
```

---

### Sending HTTP with /dev/tcp

```bash
# HTTP GET request using only bash (no curl/wget)
http_get() {
    local host="$1" path="${2:-/}" port="${3:-80}"
    local response

    exec 3<>/dev/tcp/"$host"/"$port"        # open TCP socket
    printf 'GET %s HTTP/1.0\r\nHost: %s\r\nConnection: close\r\n\r\n' \
        "$path" "$host" >&3                  # send HTTP request
    response=$(cat <&3)                      # read full response
    exec 3>&-                               # close socket

    # Split headers from body
    echo "${response#*$'\r\n\r\n'}"        # return body only
}

# Simple health check
health_check() {
    local host="$1" port="${2:-80}"
    (echo >/dev/tcp/"$host"/"$port") 2>/dev/null && echo "UP" || echo "DOWN"
}
health_check localhost 8080
```

---

### Unit Testing with bats-core

```bash
# bats-core — Bash Automated Testing System
# Install: git clone https://github.com/bats-core/bats-core.git && ./install.sh /usr/local
# Run:     bats tests/

# tests/mylib.bats
#!/usr/bin/env bats

setup() {                                   # runs before each test
    load '../src/mylib.sh'                  # source the library under test
    TMPDIR_TEST=$(mktemp -d)               # temp dir per test
}

teardown() {                               # runs after each test
    rm -rf "$TMPDIR_TEST"
}

@test "trim() removes leading whitespace" {
    result=$(trim "   hello")
    [[ "$result" == "hello" ]]             # assertion: just a command with exit code
}

@test "trim() removes trailing whitespace" {
    result=$(trim "hello   ")
    [[ "$result" == "hello" ]]
}

@test "is_integer() accepts valid integers" {
    run is_integer "42"                    # 'run' captures output and exit code
    [[ "$status" -eq 0 ]]                 # check exit code
}

@test "is_integer() rejects floats" {
    run is_integer "3.14"
    [[ "$status" -ne 0 ]]
}

@test "deploy fails on unknown environment" {
    run deploy_to "invalid_env"
    [[ "$status" -eq 1 ]]
    [[ "$output" == *"Invalid environment"* ]]  # check output contains string
}

@test "config file is loaded correctly" {
    cat > "$TMPDIR_TEST/config.conf" <<'EOF'
DB_HOST=testdb
DB_PORT=5433
EOF
    load_config "$TMPDIR_TEST/config.conf"
    [[ "$DB_HOST" == "testdb" ]]
    [[ "$DB_PORT" == "5433" ]]
}
```

---

### Self-Updating Script

```bash
# Check for new version and update if available
self_update() {
    local script_url="https://raw.githubusercontent.com/org/repo/main/script.sh"
    local current_version="$VERSION"
    local remote_version

    log "Checking for updates..."
    remote_version=$(curl -sf "${script_url%.*}.version" 2>/dev/null) || {
        warn "Could not check for updates"
        return 0
    }

    if [[ "$remote_version" == "$current_version" ]]; then
        log "Already up to date (v$current_version)"
        return 0
    fi

    log "Update available: v$current_version → v$remote_version"
    read -r -p "Update now? [y/N]: " answer < /dev/tty
    [[ "$answer" =~ ^[Yy]$ ]] || return 0

    local tmpfile
    tmpfile=$(mktemp)
    curl -sf "$script_url" -o "$tmpfile" || die "Download failed"
    bash -n "$tmpfile" || die "Downloaded script has syntax errors"   # verify before replacing
    chmod +x "$tmpfile"
    mv "$tmpfile" "${BASH_SOURCE[0]}"       # replace ourselves
    log "Updated to v$remote_version — please re-run"
    exit 0
}
```

---

### Menu-Driven Script

```bash
# Interactive menu using select
show_menu() {
    local PS3=$'\nChoose an action: '      # PS3 is the select prompt
    local options=("Deploy" "Rollback" "Status" "Logs" "Quit")

    select choice in "${options[@]}"; do
        case "$REPLY" in                   # $REPLY = what user typed
            1) deploy;;
            2) rollback;;
            3) show_status;;
            4) show_logs;;
            5|q|Q) echo "Goodbye!"; break;;
            *) echo "Invalid choice: $REPLY";;
        esac
        echo    # blank line between iterations
    done
}

# whiptail/dialog — TUI menus (if available)
if command -v whiptail &>/dev/null; then
    choice=$(whiptail --title "Deploy" \
        --menu "Select environment:" 15 40 4 \
        "dev"     "Development" \
        "staging" "Staging" \
        "prod"    "Production" \
        3>&1 1>&2 2>&3)                    # 3>&1 1>&2 2>&3: swap stdout/stderr
    echo "Chose: $choice"
fi
```

---

## Appendix: Quick Reference

### set Options Summary

| Flag | Long name | Effect |
|------|-----------|--------|
| `-e` | `errexit` | Exit on any command failure |
| `-u` | `nounset` | Error on unset variables |
| `-x` | `xtrace` | Print commands before executing |
| `-v` | `verbose` | Print input as it's read |
| `-n` | `noexec` | Parse only, don't execute |
| `-f` | `noglob` | Disable filename expansion |
| `-C` | `noclobber` | Don't overwrite files with `>` |
| `-E` | `errtrace` | ERR trap inherited by functions |
| `-o pipefail` | — | Pipeline fails if any stage fails |

### Parameter Expansion Quick Reference

| Syntax | Meaning |
|--------|---------|
| `${var:-val}` | Use `val` if unset or empty |
| `${var:=val}` | Assign and use `val` if unset or empty |
| `${var:+val}` | Use `val` if set and non-empty |
| `${var:?msg}` | Error with `msg` if unset or empty |
| `${#var}` | Length of string |
| `${var:n:l}` | Substring at offset n, length l |
| `${var#pat}` | Remove shortest prefix matching pat |
| `${var##pat}` | Remove longest prefix matching pat |
| `${var%pat}` | Remove shortest suffix matching pat |
| `${var%%pat}` | Remove longest suffix matching pat |
| `${var/p/r}` | Replace first match of p with r |
| `${var//p/r}` | Replace all matches of p with r |
| `${var^^}` | Uppercase all characters |
| `${var,,}` | Lowercase all characters |
| `${!var}` | Expand variable named by `$var` |

### Trap Signals Summary

| Signal | When triggered | Typical use |
|--------|---------------|-------------|
| `EXIT` | Any exit (normal, error, signal) | Cleanup always |
| `ERR` | Non-zero exit code | Error handling/reporting |
| `INT` | Ctrl+C (SIGINT) | Graceful interrupt |
| `TERM` | `kill` default signal | Graceful shutdown |
| `HUP` | Terminal hangup / reload | Config reload |
| `DEBUG` | Before every command | Tracing/stepping |
| `RETURN` | Function returns | Function post-hook |

---

*Bash version: 5.x+ recommended. Test with `bash --version`.*
*Reference: [GNU Bash Manual](https://www.gnu.org/software/bash/manual/) | [ShellCheck](https://www.shellcheck.net) | [bats-core](https://bats-core.readthedocs.io)*
