# 🐍 Python Comprehensive Cheatsheet

> A complete reference for Python — covering theory, internals, syntax, OOP, concurrency, typing, testing, and modern best practices.

---

## 📚 Table of Contents

1. [Core Concepts & Theory](#1-core-concepts--theory)
2. [Setup & Environment](#2-setup--environment)
3. [Built-in Types & Data Structures](#3-built-in-types--data-structures)
4. [Operators & Expressions](#4-operators--expressions)
5. [Control Flow](#5-control-flow)
6. [Functions](#6-functions)
7. [Classes & Object-Oriented Programming](#7-classes--object-oriented-programming)
8. [Comprehensions & Functional Patterns](#8-comprehensions--functional-patterns)
9. [Modules & Packages](#9-modules--packages)
10. [Error Handling & Exceptions](#10-error-handling--exceptions)
11. [File I/O & Pathlib](#11-file-io--pathlib)
12. [Concurrency & Parallelism](#12-concurrency--parallelism)
13. [Type Hints & Static Analysis](#13-type-hints--static-analysis)
14. [Testing](#14-testing)
15. [Performance & Profiling](#15-performance--profiling)
16. [Pythonic Idioms & Best Practices](#16-pythonic-idioms--best-practices)
17. [Modern Python Features (3.8–3.13)](#17-modern-python-features-38--313)
18. [Quick Reference Tables](#18-quick-reference-tables)

---

## 1. Core Concepts & Theory

### What is Python?

Python is an **interpreted, dynamically typed, garbage-collected, multi-paradigm** programming language created by Guido van Rossum in 1991.

| Property | Meaning |
|----------|---------|
| **Interpreted** | Source runs via an interpreter, not compiled to native machine code directly |
| **Dynamically typed** | Variable types are checked at runtime, not compile time |
| **Garbage-collected** | Memory is managed automatically (reference counting + cyclic GC) |
| **Multi-paradigm** | Supports procedural, object-oriented, and functional styles |

---

### CPython vs PyPy vs Jython

| Implementation | Language | Notes |
|----------------|----------|-------|
| **CPython** | C | Reference implementation — what `python` means by default |
| **PyPy** | RPython | JIT-compiled — often 5–10× faster for CPU-bound loops |
| **Jython** | Java | Runs on the JVM — no GIL, integrates with Java libraries |
| **GraalPy** | Java | GraalVM-based — high-performance polyglot |
| **MicroPython** | C | Minimal Python for microcontrollers |

---

### The Python Execution Model

```
Source Code (.py)
       │
       ▼  (compilation — automatic)
  Bytecode (.pyc in __pycache__/)
       │
       ▼  (interpretation)
Python Virtual Machine (PVM / CPython interpreter)
       │
       ▼
  Operating System / Hardware
```

```python
import dis  # disassembler module

def add(a, b):
    return a + b

dis.dis(add)  # show the bytecode instructions for this function
# Output:
#   LOAD_FAST   'a'
#   LOAD_FAST   'b'
#   BINARY_OP   +
#   RETURN_VALUE
```

---

### The GIL (Global Interpreter Lock)

The GIL is a mutex in CPython that allows only **one thread to execute Python bytecode at a time**.

```
Threading with GIL:                   Multiprocessing (no GIL):
┌─────────────────────┐              ┌──────────┐ ┌──────────┐
│     Thread 1  ████  │              │Process 1 │ │Process 2 │
│     Thread 2   ████ │              │  ██████  │ │  ██████  │
│ (one at a time)     │              │(parallel)│ │(parallel)│
└─────────────────────┘              └──────────┘ └──────────┘
```

```python
# GIL implications:
# ✅ Threads help with I/O-bound tasks (GIL released during I/O)
# ❌ Threads don't speed up CPU-bound tasks (GIL prevents true parallelism)
# ✅ multiprocessing bypasses the GIL (separate Python processes)
# ✅ Python 3.13+ introduces experimental no-GIL (free-threaded) mode

import threading
import time

# I/O-bound: threading is effective
def fetch_url(url):
    time.sleep(1)  # simulates network I/O — GIL is released during sleep

# CPU-bound: threading won't help; use multiprocessing instead
def compute():
    sum(range(10_000_000))  # pure Python computation — GIL is held
```

---

### Everything is an Object

```python
# In Python, every value is an object with an identity, type, and value
x = 42

id(x)      # identity: memory address (CPython implementation detail)
type(x)    # type: <class 'int'>
x          # value: 42

# Even functions, classes, and modules are objects
type(len)       # <class 'builtin_function_or_method'>
type(int)       # <class 'type'>
type(type)      # <class 'type'>  — type is its own metaclass

# Functions are first-class objects
def greet(name):
    return f"Hello, {name}"

f = greet           # assign function to variable
f("Alice")          # call via variable → "Hello, Alice"
print(type(greet))  # <class 'function'>
```

---

### Mutability vs Immutability

```python
# IMMUTABLE: object cannot be changed after creation
# int, float, complex, bool, str, bytes, tuple, frozenset, None

x = "hello"
id_before = id(x)
x += " world"       # creates a NEW string object — doesn't modify in place
id(x) == id_before  # False — x points to a new object

# MUTABLE: object can be changed in place
# list, dict, set, bytearray, user-defined classes

my_list = [1, 2, 3]
id_before = id(my_list)
my_list.append(4)           # modifies IN PLACE — same object
id(my_list) == id_before    # True — same object

# Why it matters: shared references to mutable objects
a = [1, 2, 3]
b = a               # b and a point to the SAME list
b.append(4)
print(a)            # [1, 2, 3, 4] — a was also changed!

# Fix: make a copy
b = a.copy()        # or: b = a[:]  or: b = list(a)
b.append(5)
print(a)            # [1, 2, 3, 4] — a is unchanged
```

---

### Reference Semantics: `is` vs `==`

```python
# == : value equality (calls __eq__)
# is : identity equality (same object in memory)

a = [1, 2, 3]
b = [1, 2, 3]

a == b    # True  — same values
a is b    # False — different objects

# Small integer caching (CPython implementation detail)
x = 256
y = 256
x is y    # True  — CPython caches -5 to 256

x = 257
y = 257
x is y    # False — outside cache range (don't rely on this!)

# Always use 'is' to compare with None, True, False
value = None
if value is None:       # ✅ correct
    pass
if value == None:       # ⚠️ works but bad practice
    pass

# Shallow vs deep copy
import copy
original = [[1, 2], [3, 4]]
shallow  = original.copy()        # shallow: inner lists still shared
deep     = copy.deepcopy(original) # deep: fully independent copy

shallow[0].append(99)
print(original)   # [[1, 2, 99], [3, 4]] — inner list was shared!
deep[0].append(99)
print(original)   # [[1, 2, 99], [3, 4]] — deep copy is independent
```

---

### Python's Memory Model

```
Reference counting:

x = [1, 2, 3]    → list object, refcount = 1
y = x            → refcount = 2
del x            → refcount = 1
del y            → refcount = 0 → object deallocated immediately

Cyclic garbage collector handles reference cycles:

a = {}
b = {}
a['ref'] = b     → b refcount = 2
b['ref'] = a     → a refcount = 2
del a            → a refcount = 1 (still referenced by b)
del b            → b refcount = 1 (still referenced by a)
# Neither reaches 0 — cyclic GC finds and collects these
```

```python
import gc
import sys

x = [1, 2, 3]
sys.getrefcount(x)   # reference count (always +1 due to getrefcount's own reference)

gc.collect()         # manually trigger cyclic garbage collection
gc.get_count()       # (gen0_count, gen1_count, gen2_count) — 3-generation GC
gc.disable()         # disable cyclic GC (safe if you manage cycles manually)
```

---

### Namespaces and Scope — LEGB Rule

```
LEGB Resolution Order (inner → outer):

┌─────────────────────────────────────┐
│  Built-in  (len, print, int, ...)   │  ← outermost
├─────────────────────────────────────┤
│  Global    (module-level names)     │
├─────────────────────────────────────┤
│  Enclosing (outer function's scope) │
├─────────────────────────────────────┤
│  Local     (current function)       │  ← innermost (searched first)
└─────────────────────────────────────┘
```

```python
x = "global"          # Global scope

def outer():
    x = "enclosing"   # Enclosing scope

    def inner():
        x = "local"   # Local scope
        print(x)      # "local" — finds local first

    inner()
    print(x)          # "enclosing"

outer()
print(x)              # "global"

# global keyword: modify global variable from inside function
counter = 0
def increment():
    global counter    # declare intent to modify global
    counter += 1

# nonlocal keyword: modify enclosing (non-global) variable
def make_counter():
    count = 0
    def increment():
        nonlocal count   # modify enclosing function's variable
        count += 1
        return count
    return increment
```

---

### The Python Data Model — Dunder Methods

```python
# Python operators call dunder methods under the hood
# a + b   → a.__add__(b)
# len(x)  → x.__len__()
# x[i]    → x.__getitem__(i)
# str(x)  → x.__str__()
# x == y  → x.__eq__(y)

class Vector:
    def __init__(self, x, y):     # called by Vector(x, y)
        self.x = x
        self.y = y

    def __repr__(self):           # unambiguous string — called by repr(v)
        return f"Vector({self.x}, {self.y})"

    def __str__(self):            # human-readable — called by str(v) and print(v)
        return f"({self.x}, {self.y})"

    def __add__(self, other):     # called by v1 + v2
        return Vector(self.x + other.x, self.y + other.y)

    def __len__(self):            # called by len(v)
        return 2

    def __eq__(self, other):      # called by v1 == v2
        return self.x == other.x and self.y == other.y

    def __abs__(self):            # called by abs(v)
        return (self.x**2 + self.y**2) ** 0.5

    def __bool__(self):           # called by bool(v) and truthiness tests
        return bool(self.x or self.y)

v1 = Vector(1, 2)
v2 = Vector(3, 4)
print(v1 + v2)    # calls __add__ → Vector(4, 6) → calls __repr__
len(v1)           # calls __len__ → 2
abs(v2)           # calls __abs__ → 5.0
```

---

## 2. Setup & Environment

### Installing Python — pyenv

```bash
# pyenv: manage multiple Python versions side by side

# Install pyenv (macOS/Linux)
curl https://pyenv.run | bash
# Add to ~/.bashrc or ~/.zshrc:
# export PYENV_ROOT="$HOME/.pyenv"
# export PATH="$PYENV_ROOT/bin:$PATH"
# eval "$(pyenv init -)"

# Usage
pyenv install --list              # list all available Python versions
pyenv install 3.12.3              # install a specific version
pyenv install 3.11.9              # install another version
pyenv versions                    # list installed versions (* = active)
pyenv global 3.12.3               # set global default version
pyenv local 3.11.9                # set version for current directory (.python-version file)
pyenv shell 3.10.14               # set version for current shell session only

# Verify
python --version                  # should show pyenv-managed version
which python                      # should be in ~/.pyenv/shims/
```

---

### Virtual Environments

```bash
# venv (built-in, recommended for most projects)
python -m venv .venv              # create virtual environment in .venv/
source .venv/bin/activate         # activate (Linux/macOS)
.venv\Scripts\activate            # activate (Windows)
pip install requests              # installs into .venv, not globally
deactivate                        # deactivate environment
rm -rf .venv                      # delete environment

# virtualenv (third-party, more features)
pip install virtualenv
virtualenv .venv --python=python3.11  # specify Python version
virtualenv .venv --copies             # copy instead of symlink

# conda (data science — manages Python + non-Python packages)
conda create -n myenv python=3.12   # create named environment
conda activate myenv                 # activate
conda install numpy pandas           # install packages (uses conda repos)
conda env export > environment.yml   # export environment spec
conda env create -f environment.yml  # recreate from spec
conda deactivate
conda remove -n myenv --all          # delete environment
```

---

### Package Management

```bash
# pip — standard package manager
pip install requests                   # install latest
pip install requests==2.31.0           # install specific version
pip install "requests>=2.28,<3.0"      # install with constraints
pip install -r requirements.txt        # install from file
pip install -e .                       # install current package in editable mode
pip install --upgrade pip              # upgrade pip itself
pip list                               # list installed packages
pip show requests                      # details about a package
pip uninstall requests                 # uninstall
pip freeze > requirements.txt          # snapshot current environment

# pip-tools — better dependency management
pip install pip-tools
# Write requirements.in with direct deps:
# requests>=2.28
# fastapi
pip-compile requirements.in           # generates requirements.txt with pinned versions
pip-sync requirements.txt             # install exactly what's in requirements.txt

# pipx — install CLI tools in isolated environments
pip install pipx
pipx install black                    # installs black in isolation
pipx install ruff
pipx run cowsay "hello"               # run without installing
pipx list                             # list installed tools
pipx upgrade-all                      # upgrade all tools
```

---

### pyproject.toml

```toml
# pyproject.toml — modern project configuration (PEP 517/518/621)
[build-system]
requires      = ["hatchling"]          # build backend
build-backend = "hatchling.build"

[project]
name        = "mypackage"
version     = "1.0.0"
description = "A great Python package"
readme      = "README.md"
requires-python = ">=3.11"

dependencies = [                       # runtime dependencies
    "requests>=2.28",
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = [                                # extras: pip install mypackage[dev]
    "pytest>=7.0",
    "ruff",
    "mypy",
]

[project.scripts]
myapp = "mypackage.cli:main"           # creates CLI entrypoint

[tool.ruff]                            # ruff linter config
line-length = 88
select = ["E", "F", "I"]

[tool.mypy]                            # mypy type checker config
strict = true
python_version = "3.12"

[tool.pytest.ini_options]              # pytest config
testpaths = ["tests"]
addopts = "-v --cov=mypackage"
```

---

### uv — Fast Modern Package Manager

```bash
# uv: Rust-based Python package manager (10-100x faster than pip)
# Install:
curl -LsSf https://astral.sh/uv/install.sh | sh

# Project management
uv init myproject                  # create new project with pyproject.toml
uv add requests                    # add dependency (updates pyproject.toml + uv.lock)
uv add --dev pytest ruff mypy      # add dev dependencies
uv remove requests                 # remove dependency
uv sync                            # install all deps from uv.lock
uv run python main.py              # run script in project environment
uv run pytest                      # run tests

# Python version management
uv python install 3.12             # install Python version
uv python list                     # list available versions
uv python pin 3.12                 # pin version for this project

# Virtual environments
uv venv                            # create .venv with current Python
uv venv --python 3.11              # create with specific version

# Package operations (like pip)
uv pip install requests            # fast pip-compatible install
uv pip compile requirements.in     # like pip-compile
uv tool install ruff               # like pipx install
```

---

### Project Structure Conventions

```
myproject/
├── src/
│   └── mypackage/          # src layout (recommended)
│       ├── __init__.py
│       ├── core.py
│       ├── utils.py
│       └── cli.py
├── tests/
│   ├── conftest.py         # pytest fixtures
│   ├── test_core.py
│   └── test_utils.py
├── docs/
├── .venv/                  # virtual environment (gitignored)
├── pyproject.toml          # project config + tool configs
├── uv.lock                 # or requirements.txt
├── README.md
├── .gitignore
└── .python-version         # pyenv version pin
```

---

## 3. Built-in Types & Data Structures

### Numeric Types

```python
# int — arbitrary precision (no overflow!)
big = 2 ** 1000             # works perfectly
type(42)                    # <class 'int'>
0xFF                        # 255 (hex literal)
0b1010                      # 10  (binary literal)
0o17                        # 15  (octal literal)
1_000_000                   # 1000000 (underscores for readability)

# float — IEEE 754 double precision (64-bit)
type(3.14)                  # <class 'float'>
1e10                        # 10000000000.0 (scientific notation)
float('inf')                # infinity
float('nan')                # not-a-number
import math
math.isinf(float('inf'))    # True
math.isnan(float('nan'))    # True

# ⚠️ Float precision gotcha — classic footgun
0.1 + 0.2 == 0.3            # False! (0.30000000000000004)
abs(0.1 + 0.2 - 0.3) < 1e-9  # True — compare with tolerance
math.isclose(0.1 + 0.2, 0.3)  # True — preferred for float comparison

# Decimal — fixed-point arithmetic (no float errors)
from decimal import Decimal, getcontext
getcontext().prec = 50      # set global precision
Decimal('0.1') + Decimal('0.2')  # Decimal('0.3') — exact!
Decimal('0.1') + Decimal('0.2') == Decimal('0.3')  # True

# Fraction — exact rational arithmetic
from fractions import Fraction
Fraction(1, 3) + Fraction(1, 6)   # Fraction(1, 2) — exact!

# complex
z = 3 + 4j
z.real     # 3.0
z.imag     # 4.0
abs(z)     # 5.0 (magnitude)
z.conjugate()  # (3-4j)
```

---

### Strings

```python
# Strings are immutable sequences of Unicode code points
s = "hello"
s = 'hello'             # single or double quotes — equivalent
s = """multi
line"""                 # triple-quoted string
s = r"\n is not a newline"  # raw string — backslashes are literal
s = b"bytes"            # bytes literal (type: bytes, not str)
s = f"value: {42*2}"   # f-string (Python 3.6+) — evaluated at runtime

# Immutability
s = "hello"
s[0] = "H"              # TypeError: 'str' object does not support item assignment
s = "H" + s[1:]         # creates a new string

# String interning (CPython optimisation)
a = "hello"
b = "hello"
a is b                  # True — CPython interns short strings
a = "hello world!"      # longer/complex strings may not be interned
b = "hello world!"
a is b                  # may be False — don't rely on this

# Common string methods
s = "  Hello, World!  "
s.strip()               # "Hello, World!"  — remove leading/trailing whitespace
s.lstrip()              # "Hello, World!  " — left strip only
s.rstrip()              # "  Hello, World!" — right strip only
s.lower()               # "  hello, world!  "
s.upper()               # "  HELLO, WORLD!  "
s.title()               # "  Hello, World!  "
s.replace("World", "Python")   # "  Hello, Python!  "
s.split(",")            # ['  Hello', ' World!  ']
",".join(["a", "b", "c"])      # "a,b,c"
s.strip().startswith("Hello")  # True
s.strip().endswith("!")        # True
"42".zfill(5)           # "00042" — zero-pad
"hello".center(11, "-") # "---hello---"
"hello world".count("l")  # 3
"hello".find("ll")        # 2 — returns -1 if not found
"hello".index("ll")       # 2 — raises ValueError if not found

# str vs bytes vs bytearray
text  = "hello"             # str:       Unicode text
raw   = b"hello"            # bytes:     immutable byte sequence
buf   = bytearray(b"hello") # bytearray: mutable byte sequence

text.encode("utf-8")        # str → bytes
raw.decode("utf-8")         # bytes → str
buf[0] = ord("H")           # mutate bytearray in place
```

---

### bool and None

```python
# bool is a subclass of int
isinstance(True, int)   # True
True + True             # 2
True * 5                # 5

# Truthiness rules — what evaluates to False:
# False, None, 0, 0.0, 0j, "", b"", [], {}, set(), range(0)
# Any empty container, any zero numeric value, None, False

bool(0)         # False
bool("")        # False
bool([])        # False
bool({})        # False
bool(None)      # False
bool(42)        # True
bool("hi")      # True
bool([0])       # True — non-empty list, even with falsy element

# Short-circuit evaluation
x = None
y = x or "default"      # "default" — x is falsy, so returns right side
y = x and x.strip()     # None  — x is falsy, short-circuits (no AttributeError)

# None — singleton, use 'is' for comparison
x = None
x is None       # True  ✅
x == None       # True  ⚠️ (works but bad practice)
x is not None   # False

# None as sentinel (signal "not provided")
def connect(host, port=None):
    if port is None:            # "was port provided?"
        port = 5432             # use default
```

---

### list

```python
# Lists: ordered, mutable, allow duplicates, heterogeneous
lst = [1, 2, 3]
lst = list(range(5))     # [0, 1, 2, 3, 4]
lst = [x**2 for x in range(5)]  # [0, 1, 4, 9, 16]

# Indexing and slicing
lst = [10, 20, 30, 40, 50]
lst[0]          # 10  — first element
lst[-1]         # 50  — last element
lst[1:3]        # [20, 30] — slice [start:stop] (stop exclusive)
lst[::2]        # [10, 30, 50] — every other element
lst[::-1]       # [50, 40, 30, 20, 10] — reversed
lst[1:4:2]      # [20, 40] — [start:stop:step]

# Mutation methods
lst.append(60)           # add to end        O(1)
lst.insert(0, 5)         # insert at index   O(n)
lst.extend([70, 80])     # add multiple      O(k)
lst.pop()                # remove last       O(1)
lst.pop(0)               # remove at index   O(n)
lst.remove(30)           # remove first 30   O(n)
lst.index(40)            # find index        O(n)
lst.count(10)            # count occurrences O(n)
lst.sort()               # sort in place     O(n log n)
lst.sort(key=abs, reverse=True)  # sort by key, descending
lst.reverse()            # reverse in place  O(n)
lst.clear()              # remove all        O(n)
copy = lst.copy()        # shallow copy      O(n)

# Sorted vs sort
nums = [3, 1, 4, 1, 5]
sorted_nums = sorted(nums)    # returns new list, original unchanged
nums.sort()                   # modifies in place, returns None

# Time complexities
# append, pop() — O(1) amortized
# insert, pop(i), remove — O(n)
# index, count — O(n)
# sort — O(n log n)
# x in list — O(n)  ← use set for fast membership testing
```

---

### tuple

```python
# Tuples: ordered, immutable, allow duplicates, hashable (if elements are)
t = (1, 2, 3)
t = 1, 2, 3          # parentheses optional for packing
t = (42,)            # single-element tuple — trailing comma required!
t = ()               # empty tuple
t = tuple([1, 2, 3]) # from iterable

# Unpacking
x, y, z = (1, 2, 3)  # basic unpacking
a, *rest = (1, 2, 3, 4, 5)  # extended: a=1, rest=[2,3,4,5]
first, *middle, last = range(5)  # first=0, middle=[1,2,3], last=4

# Tuple as dict key (immutable → hashable)
coords = {(0, 0): "origin", (1, 0): "right"}
coords[(0, 0)]   # "origin"

# namedtuple — tuple with field names
from collections import namedtuple
Point = namedtuple("Point", ["x", "y"])
p = Point(3, 4)
p.x     # 3
p.y     # 4
p[0]    # 3 — still accessible by index
p._asdict()        # OrderedDict([('x', 3), ('y', 4)])
p._replace(x=10)   # Point(x=10, y=4) — returns new tuple

# typing.NamedTuple — typed version
from typing import NamedTuple
class Point3D(NamedTuple):
    x: float
    y: float
    z: float = 0.0   # default value
```

---

### dict

```python
# Dicts: ordered (3.7+), mutable, keys must be hashable
d = {"a": 1, "b": 2}
d = dict(a=1, b=2)       # keyword form
d = dict([("a", 1)])     # from iterable of (key, value) pairs
d = {k: v for k, v in zip("abc", [1, 2, 3])}  # comprehension

# Access and mutation
d["a"]                  # 1 (KeyError if missing)
d.get("x")              # None (no error if missing)
d.get("x", 0)           # 0   (default value)
d["c"] = 3              # add/update
del d["b"]              # delete key
d.pop("a")              # remove and return value
d.pop("z", None)        # safe pop — no error if missing
d.setdefault("d", [])   # set default if key absent, return value
d.update({"e": 5})      # merge another dict in
d |= {"f": 6}           # merge in place (Python 3.9+)
new = d | {"g": 7}      # merge into new dict (Python 3.9+)

# Iteration
d.keys()                # dict_keys view
d.values()              # dict_values view
d.items()               # dict_items view (key, value) pairs

for key in d:           # iterate over keys
    print(key, d[key])

for k, v in d.items():  # iterate over key-value pairs (preferred)
    print(k, v)

# Check membership (checks KEYS only)
"a" in d        # True
"z" in d        # False

# defaultdict — auto-creates missing keys
from collections import defaultdict
dd = defaultdict(list)    # missing keys default to empty list
dd["x"].append(1)         # no KeyError — creates [] then appends
dd["x"].append(2)
print(dd)                 # defaultdict(<class 'list'>, {'x': [1, 2]})

dd2 = defaultdict(int)    # missing keys default to 0
dd2["count"] += 1         # works even though "count" didn't exist

# Counter — counting hashable objects
from collections import Counter
c = Counter("abracadabra")         # count character frequencies
c                   # Counter({'a': 5, 'b': 2, 'r': 2, 'c': 1, 'd': 1})
c.most_common(3)    # [('a', 5), ('b', 2), ('r', 2)]
c["a"]              # 5
c["z"]              # 0 (no KeyError!)
c1 = Counter("abc")
c2 = Counter("bcd")
c1 + c2             # Counter({'b': 2, 'c': 2, 'a': 1, 'd': 1})

# ChainMap — search multiple dicts in order
from collections import ChainMap
defaults = {"color": "red", "size": "medium"}
overrides = {"color": "blue"}
cm = ChainMap(overrides, defaults)  # search overrides first
cm["color"]     # "blue"  — found in overrides
cm["size"]      # "medium" — falls through to defaults
```

---

### set and frozenset

```python
# Sets: unordered, mutable, unique elements, elements must be hashable
s = {1, 2, 3}
s = set([1, 2, 2, 3])      # from iterable — duplicates removed → {1, 2, 3}
s = set()                  # empty set (NOT {}, which is empty dict!)

# Mutation
s.add(4)                   # add element
s.remove(1)                # remove (KeyError if absent)
s.discard(99)              # remove (no error if absent)
s.pop()                    # remove and return arbitrary element
s.clear()                  # remove all

# Set operations
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}
a | b                      # union: {1, 2, 3, 4, 5, 6}
a & b                      # intersection: {3, 4}
a - b                      # difference: {1, 2}
a ^ b                      # symmetric difference: {1, 2, 5, 6}
a.issubset(b)              # False
a.issuperset({1, 2})       # True
a.isdisjoint({7, 8})       # True — no common elements

# frozenset — immutable, hashable, usable as dict key
fs = frozenset({1, 2, 3})
d = {fs: "a frozen set key"}   # frozenset as dict key

# Time complexity
# add, remove, discard — O(1) average
# x in set — O(1) average  ← much faster than list for membership!
# union, intersection — O(min(len(a), len(b)))

# Use set for fast membership testing
big_list = list(range(1_000_000))
big_set  = set(big_list)
999_999 in big_list    # O(n) — slow
999_999 in big_set     # O(1) — fast!
```

---

### range, bytes, bytearray, memoryview

```python
# range — lazy sequence of integers (very memory efficient)
r = range(10)           # 0 to 9
r = range(1, 11)        # 1 to 10
r = range(0, 20, 2)     # 0, 2, 4, ... 18
list(range(5))          # [0, 1, 2, 3, 4]
5 in range(1000000)     # O(1) — range supports fast membership test
len(range(100))         # 100 — O(1)
range(10)[3]            # 3 — O(1) indexing

# bytes — immutable byte sequences
b = b"hello"
b = bytes([72, 101, 108, 108, 111])  # from list of ints
b[0]                    # 72 — int (ASCII value)
b.decode("utf-8")       # "hello" — convert to str

# bytearray — mutable bytes
ba = bytearray(b"hello")
ba[0] = 72              # OK — mutable
ba.append(33)           # append byte value

# memoryview — zero-copy view of buffer objects
data = bytearray(range(10))
view = memoryview(data)
view[2:5]               # memoryview slice — no copy!
bytes(view[2:5])        # b'\x02\x03\x04'
```

---

### Type Conversion

```python
int("42")           # 42
int("0xFF", 16)     # 255 — parse hex string
int(3.9)            # 3 — truncates (does NOT round)
float("3.14")       # 3.14
float(42)           # 42.0
str(42)             # "42"
str(3.14)           # "3.14"
bool(0)             # False
bool("hello")       # True
list((1, 2, 3))     # [1, 2, 3]
tuple([1, 2, 3])    # (1, 2, 3)
set([1, 2, 2, 3])   # {1, 2, 3}
dict([("a", 1)])    # {'a': 1}
```

---

## 4. Operators & Expressions

### All Operators

```python
# --- Arithmetic ---
10 + 3      # 13   addition
10 - 3      # 7    subtraction
10 * 3      # 30   multiplication
10 / 3      # 3.333... true division (always float)
10 // 3     # 3    floor division (int result)
10 % 3      # 1    modulo
10 ** 3     # 1000 exponentiation
-10 // 3    # -4   floor division rounds toward -infinity

# --- Comparison (return bool) ---
1 == 1      # True   value equality
1 != 2      # True   value inequality
1 < 2       # True
1 <= 1      # True
2 > 1       # True
2 >= 2      # True

# Chained comparisons — Python's killer feature
1 < x < 10          # True if x is between 1 and 10 (exclusive)
0 <= i < len(lst)   # valid index check
a == b == c         # all three are equal

# --- Logical ---
True and False   # False — short-circuits: evaluates left, if falsy, returns it
True or False    # True  — short-circuits: evaluates left, if truthy, returns it
not True         # False

# Logical operators return OPERANDS, not bool
"hello" or "default"     # "hello" — truthy, returned as-is
""      or "default"     # "default" — "" is falsy
None    and "value"      # None — None is falsy, short-circuits
42      and "value"      # "value" — 42 is truthy, returns right side

# --- Bitwise ---
0b1010 & 0b1100   # 0b1000 = 8   AND
0b1010 | 0b1100   # 0b1110 = 14  OR
0b1010 ^ 0b1100   # 0b0110 = 6   XOR
~0b1010           # -11          NOT (bitwise complement)
1 << 3            # 8            left shift (multiply by 2^3)
16 >> 2           # 4            right shift (divide by 2^2)

# --- Identity and Membership ---
x is None         # True if x is the None singleton
x is not None     # True if x is not None
"a" in "banana"   # True — substring check
3 in [1, 2, 3]    # True — list membership
"k" in {"k": 1}   # True — dict key membership

# --- Augmented assignment ---
x = 10
x += 5    # x = x + 5  → 15
x -= 3    # x = x - 3  → 12
x *= 2    # x = x * 2  → 24
x //= 5   # x = x // 5 → 4
x **= 3   # x = x ** 3 → 64
x %= 10   # x = x % 10 → 4
x &= 0b11 # x = x & 0b11 → 0
```

---

### Walrus Operator `:=` (Python 3.8+)

```python
# := assigns AND returns a value in a single expression
# Useful to avoid computing the same thing twice

# ⚠️ Old way — reads file line twice (or needs extra variable)
line = f.readline()
while line:
    process(line)
    line = f.readline()

# ✅ New way with walrus operator
while line := f.readline():   # assign AND check truthiness
    process(line)

# Useful in comprehensions with filtering
# ⚠️ Without walrus — compute expensive() twice
results = [y for x in data if (y := expensive(x)) > 0]

# ✅ With walrus — compute once, use in both condition and value
results = [transformed for x in data
           if (transformed := expensive(x)) is not None]

# In if statements
import re
if m := re.match(r"(\d+)", text):  # assign match AND check if it matched
    print(m.group(1))              # use m without re-running match
```

---

### Operator Precedence (High to Low)

```python
# Highest precedence first:
# 1.  (expr), [expr], {expr}   — parentheses, brackets
# 2.  x[i], x[i:j], f(args)   — subscript, call, attribute access
# 3.  **                        — exponentiation (right-associative!)
# 4.  +x, -x, ~x               — unary positive, negative, bitwise NOT
# 5.  *, @, /, //, %            — multiplication, matrix mul, division
# 6.  +, -                      — addition, subtraction
# 7.  <<, >>                    — bitwise shifts
# 8.  &                         — bitwise AND
# 9.  ^                         — bitwise XOR
# 10. |                         — bitwise OR
# 11. == != < <= > >= is in not in  — comparisons, identity, membership
# 12. not x                     — boolean NOT
# 13. and                       — boolean AND
# 14. or                        — boolean OR
# 15. x if C else y             — conditional expression (ternary)
# 16. lambda                    — lambda expression
# 17. :=                        — walrus operator (lowest)

# ⚠️ Common gotcha: 'not in' and 'is not' are single operators
3 not in [1, 2]     # True  — reads naturally
x is not None       # True  — reads naturally (not: (x is not) None)
```

---

## 5. Control Flow

### if / elif / else

```python
score = 85

if score >= 90:
    grade = "A"
elif score >= 80:       # elif: else-if (can have multiple)
    grade = "B"
elif score >= 70:
    grade = "C"
else:                   # optional catch-all
    grade = "F"

# Ternary expression — one-liner conditional
grade = "pass" if score >= 60 else "fail"

# Nested ternary (use sparingly — readability suffers quickly)
grade = "A" if score >= 90 else ("B" if score >= 80 else "C")
```

---

### match / case — Structural Pattern Matching (Python 3.10+)

```python
# match/case is NOT a switch statement — it's full structural pattern matching

command = "quit"
match command:
    case "quit":           # literal pattern
        print("Quitting")
    case "help":
        print("Help info")
    case _:                # wildcard — matches anything (like else)
        print(f"Unknown: {command}")

# Sequence patterns
point = (1, 2)
match point:
    case (0, 0):          # exact match
        print("Origin")
    case (x, 0):          # capture x, y must be 0
        print(f"On x-axis at {x}")
    case (0, y):
        print(f"On y-axis at {y}")
    case (x, y):          # capture both
        print(f"Point at ({x}, {y})")

# Class patterns
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float

p = Point(1.0, 2.0)
match p:
    case Point(x=0, y=0):
        print("Origin")
    case Point(x=x, y=0):    # capture x, y must be 0
        print(f"X-axis: {x}")
    case Point(x=x, y=y):
        print(f"({x}, {y})")

# OR patterns
match status_code:
    case 200 | 201 | 204:  # multiple values — OR pattern
        print("Success")
    case 400 | 422:
        print("Client error")
    case 500 | 502 | 503:
        print("Server error")

# Mapping patterns (dict)
response = {"status": "ok", "data": [1, 2, 3]}
match response:
    case {"status": "ok", "data": data}:    # capture "data" value
        print(f"Got data: {data}")
    case {"status": "error", "message": msg}:
        print(f"Error: {msg}")

# Guards (if condition on a case)
match value:
    case x if x > 0:          # guard: only matches if x > 0
        print(f"Positive: {x}")
    case x if x < 0:
        print(f"Negative: {x}")
    case 0:
        print("Zero")
```

---

### for Loops

```python
# for works with any iterable (list, tuple, str, dict, set, generator, ...)
for item in [1, 2, 3]:
    print(item)

# enumerate — get index AND value
fruits = ["apple", "banana", "cherry"]
for i, fruit in enumerate(fruits):       # i starts at 0
    print(f"{i}: {fruit}")

for i, fruit in enumerate(fruits, start=1):  # i starts at 1
    print(f"{i}: {fruit}")

# zip — iterate multiple iterables in parallel (stops at shortest)
names  = ["Alice", "Bob", "Carol"]
scores = [95, 87, 91]
for name, score in zip(names, scores):
    print(f"{name}: {score}")

# zip_longest — pad shorter iterables with fillvalue
from itertools import zip_longest
for a, b in zip_longest([1, 2, 3], ["x", "y"], fillvalue=None):
    print(a, b)  # 1 x / 2 y / 3 None

# Unpack tuples in loop
pairs = [(1, "a"), (2, "b"), (3, "c")]
for num, letter in pairs:      # unpacking in loop variable
    print(num, letter)

# Loop with else — runs if loop completed WITHOUT break
for i in range(10):
    if i == 5:
        break
else:
    print("Loop completed")    # NOT printed (break was hit)

for i in range(3):
    pass
else:
    print("Loop completed")    # IS printed (no break)
```

---

### while Loops

```python
# while — repeat while condition is truthy
count = 0
while count < 5:
    print(count)
    count += 1

# break — exit loop immediately
while True:               # infinite loop — common pattern
    data = input("> ")
    if data == "quit":
        break             # exit the while loop
    process(data)

# continue — skip rest of current iteration
for i in range(10):
    if i % 2 == 0:
        continue          # skip even numbers
    print(i)              # prints 1, 3, 5, 7, 9

# while/else — else runs if loop condition became False (not if break)
attempts = 0
while attempts < 3:
    if try_login():
        break
    attempts += 1
else:
    print("Too many failed attempts")   # runs only if no break
```

---

### try / except / else / finally

```python
# Full exception handling model
try:
    result = 10 / 0             # code that might raise
except ZeroDivisionError:       # catch specific exception
    print("Division by zero!")
except (TypeError, ValueError) as e:  # catch multiple; bind to 'e'
    print(f"Type or value error: {e}")
except Exception as e:          # catch all "normal" exceptions
    print(f"Unexpected error: {e}")
    raise                       # re-raise the same exception
else:
    # Runs ONLY if no exception was raised in try block
    print(f"Result: {result}")
finally:
    # ALWAYS runs (exception or not, even with return/break)
    print("Cleanup here — always executes")

# Exception information
try:
    risky()
except Exception as e:
    print(type(e).__name__)     # exception class name
    print(str(e))               # error message
    import traceback
    traceback.print_exc()       # full traceback
```

---

### with Statement — Context Managers

```python
# Context managers handle setup and teardown automatically
# Calls __enter__ on entry, __exit__ on exit (even if exception)

# File handling — classic use case
with open("file.txt", "r") as f:
    content = f.read()
# File is automatically closed here (even on exception)

# Multiple context managers in one with statement
with open("input.txt") as fin, open("output.txt", "w") as fout:
    fout.write(fin.read())

# contextlib.contextmanager — easy custom context manager via generator
from contextlib import contextmanager
import time

@contextmanager
def timer(label):
    start = time.perf_counter()   # setup (before yield)
    try:
        yield                      # body of 'with' block runs here
    finally:
        elapsed = time.perf_counter() - start
        print(f"{label}: {elapsed:.3f}s")  # teardown (after yield)

with timer("my operation"):
    time.sleep(0.1)
# Output: my operation: 0.100s

# contextlib.suppress — suppress specific exceptions
from contextlib import suppress
with suppress(FileNotFoundError):
    os.remove("might_not_exist.txt")   # no error if file doesn't exist
```

---

## 6. Functions

### Defining Functions

```python
def greet(name: str, greeting: str = "Hello") -> str:
    """
    Return a greeting string.

    Args:
        name: The name to greet.
        greeting: The greeting to use (default: "Hello").

    Returns:
        A formatted greeting string.
    """
    return f"{greeting}, {name}!"

# Docstring access
greet.__doc__            # get docstring
help(greet)              # formatted help output

# Annotations (type hints) — not enforced at runtime
greet.__annotations__   # {'name': <class 'str'>, 'greeting': <class 'str'>, 'return': <class 'str'>}
```

---

### Parameters — All Types

```python
# Parameter types and their order:
# 1. Positional-only (before /)
# 2. Regular positional-or-keyword
# 3. *args (variadic positional)
# 4. Keyword-only (after *args or bare *)
# 5. **kwargs (variadic keyword)

def full_example(
    pos_only1,              # positional only (before /)
    pos_only2,              # positional only
    /,                      # separator: everything before here is positional-only
    regular,                # positional or keyword
    default="value",        # positional or keyword with default
    *args,                  # variadic positional → tuple
    kw_only,                # keyword-only (after *args)
    kw_default=True,        # keyword-only with default
    **kwargs                # variadic keyword → dict
):
    print(pos_only1, pos_only2, regular, default)
    print(args)     # tuple of extra positional args
    print(kw_only)
    print(kwargs)   # dict of extra keyword args

# Calling:
full_example(1, 2, "reg", "val", "e1", "e2", kw_only="kw", extra="x")

# *args collects extra positional arguments
def sum_all(*args):
    return sum(args)       # args is a tuple

sum_all(1, 2, 3, 4)        # 10

# **kwargs collects extra keyword arguments
def show(**kwargs):
    for key, value in kwargs.items():
        print(f"{key} = {value}")

show(name="Alice", age=30)  # name = Alice / age = 30

# Unpacking at call site
args = (1, 2, 3)
func(*args)                 # unpack list/tuple as positional args

kwargs = {"a": 1, "b": 2}
func(**kwargs)              # unpack dict as keyword args
```

---

### Closures

```python
# Closure: inner function that captures variables from enclosing scope

def make_multiplier(factor):
    # 'factor' is a free variable — captured by the closure
    def multiplier(x):
        return x * factor   # closes over 'factor' from outer scope
    return multiplier       # return the inner function

double = make_multiplier(2)
triple = make_multiplier(3)
double(5)   # 10
triple(5)   # 15

# nonlocal — modify enclosing (non-global) variable
def counter_factory(start=0):
    count = start
    def increment(step=1):
        nonlocal count      # modify outer function's 'count'
        count += step
        return count
    return increment

count = counter_factory(10)
count()     # 11
count()     # 12
count(5)    # 17

# ⚠️ Classic closure gotcha: late binding in loops
# Wrong:
funcs = [lambda: i for i in range(3)]
[f() for f in funcs]    # [2, 2, 2] — all capture the SAME 'i' (final value)

# Fix: capture current value with default argument
funcs = [lambda i=i: i for i in range(3)]
[f() for f in funcs]    # [0, 1, 2] — each captures its own value
```

---

### Decorators

```python
# Decorators wrap a function to add behavior before/after it runs
# @decorator is syntactic sugar for: func = decorator(func)

import functools

# Basic decorator
def log_calls(func):
    @functools.wraps(func)  # preserves func's name, docstring, etc.
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")   # before
        result = func(*args, **kwargs)       # call original
        print(f"Done: {result}")            # after
        return result
    return wrapper

@log_calls
def add(a, b):
    return a + b

add(3, 4)
# Calling add
# Done: 7

# Parameterised decorator (decorator factory)
def retry(times=3):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == times - 1:
                        raise   # re-raise on last attempt
                    print(f"Retry {attempt + 1}/{times}: {e}")
        return wrapper
    return decorator

@retry(times=3)           # retry up to 3 times on exception
def flaky_operation():
    pass

# Stacking decorators (applied bottom-up, executed top-down)
@log_calls        # applied second (outer)
@retry(times=3)   # applied first (inner)
def risky():
    pass
# risky = log_calls(retry(times=3)(risky))

# Class decorator
class Singleton:
    """Ensure only one instance of a class exists."""
    def __init__(self, cls):
        self.cls = cls
        self.instance = None
        functools.update_wrapper(self, cls)

    def __call__(self, *args, **kwargs):
        if self.instance is None:
            self.instance = self.cls(*args, **kwargs)
        return self.instance

@Singleton
class Database:
    def __init__(self):
        self.connected = False
```

---

### Generators

```python
# Generators: functions that yield values lazily, one at a time
# They maintain state between yields — memory efficient for large sequences

def countdown(n):
    print("Starting countdown")
    while n > 0:
        yield n         # pause, return value to caller, resume on next()
        n -= 1
    print("Done!")      # runs after all yields

gen = countdown(3)      # doesn't execute yet — returns generator object
next(gen)               # "Starting countdown" → 3
next(gen)               # 4 → 2
next(gen)               # 3 → 1
next(gen)               # "Done!" → raises StopIteration

for val in countdown(3):   # for loop handles StopIteration automatically
    print(val)

# Generator expressions — lazy list comprehensions
squares = (x**2 for x in range(1000))   # () not [] — no list built in memory
next(squares)                            # 0
sum(squares)                             # computed lazily

# yield from — delegate to sub-generator
def flatten(nested):
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)    # delegate recursion
        else:
            yield item

list(flatten([1, [2, [3, 4]], 5]))   # [1, 2, 3, 4, 5]

# send() — send values INTO a generator
def accumulator():
    total = 0
    while True:
        value = yield total     # yield current total, receive next value
        if value is None:
            break
        total += value

acc = accumulator()
next(acc)         # prime the generator (must call next first) → 0
acc.send(10)      # → 10
acc.send(20)      # → 30
acc.send(5)       # → 35
```

---

### functools

```python
import functools

# lru_cache — memoize function results (Least Recently Used cache)
@functools.lru_cache(maxsize=128)    # cache up to 128 results
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

fibonacci(50)               # instant after first call
fibonacci.cache_info()      # CacheInfo(hits=48, misses=51, maxsize=128, currsize=51)
fibonacci.cache_clear()     # clear the cache

# cache (Python 3.9+) — unbounded lru_cache
@functools.cache           # equivalent to @lru_cache(maxsize=None)
def expensive(n):
    return n ** 2

# partial — fix some arguments of a function
def power(base, exp):
    return base ** exp

square = functools.partial(power, exp=2)    # fix exp=2
cube   = functools.partial(power, exp=3)    # fix exp=3
square(5)   # 25
cube(3)     # 27

# reduce — fold a sequence left-to-right
functools.reduce(lambda acc, x: acc + x, [1, 2, 3, 4])  # 10
functools.reduce(lambda acc, x: acc * x, range(1, 6))    # 120 (5!)

# total_ordering — derive comparison methods from __eq__ and one other
@functools.total_ordering
class Card:
    def __init__(self, value):
        self.value = value
    def __eq__(self, other):
        return self.value == other.value
    def __lt__(self, other):
        return self.value < other.value
    # Python auto-generates __le__, __gt__, __ge__ from __eq__ and __lt__

# singledispatch — function overloading based on argument type
@functools.singledispatch
def process(data):
    raise NotImplementedError(f"Can't process {type(data)}")

@process.register(str)
def _(data):
    return data.upper()

@process.register(list)
def _(data):
    return [process(x) for x in data]

process("hello")        # "HELLO"
process(["a", "b"])     # ["A", "B"]
```

---

### itertools

```python
import itertools

# chain — concatenate iterables
list(itertools.chain([1, 2], [3, 4], [5]))  # [1, 2, 3, 4, 5]

# islice — lazy slicing of any iterable
list(itertools.islice(range(100), 5, 10))   # [5, 6, 7, 8, 9]

# product — cartesian product (nested for loops)
list(itertools.product([1, 2], ["a", "b"]))
# [(1,'a'), (1,'b'), (2,'a'), (2,'b')]

# permutations — all orderings
list(itertools.permutations("ABC", 2))
# [('A','B'), ('A','C'), ('B','A'), ('B','C'), ('C','A'), ('C','B')]

# combinations — subsets (no repeated, order doesn't matter)
list(itertools.combinations("ABCD", 2))
# [('A','B'), ('A','C'), ('A','D'), ('B','C'), ('B','D'), ('C','D')]

# combinations_with_replacement — like combinations but allows repeats
list(itertools.combinations_with_replacement("AB", 2))
# [('A','A'), ('A','B'), ('B','B')]

# groupby — group consecutive elements by key
data = [("A", 1), ("A", 2), ("B", 3), ("B", 4), ("A", 5)]
for key, group in itertools.groupby(data, key=lambda x: x[0]):
    print(key, list(group))
# A [('A',1), ('A',2)]  / B [('B',3), ('B',4)]  / A [('A',5)]
# ⚠️ Input must be sorted by key for groupby to work as expected!

# accumulate — running totals / scan
list(itertools.accumulate([1, 2, 3, 4, 5]))         # [1, 3, 6, 10, 15]
import operator
list(itertools.accumulate([1,2,3,4,5], operator.mul))# [1, 2, 6, 24, 120]

# cycle — repeat iterable indefinitely
counter = itertools.cycle(range(3))   # 0, 1, 2, 0, 1, 2, ...
[next(counter) for _ in range(7)]     # [0, 1, 2, 0, 1, 2, 0]

# repeat — repeat a value n times (or indefinitely)
list(itertools.repeat("x", 3))        # ['x', 'x', 'x']

# takewhile / dropwhile
list(itertools.takewhile(lambda x: x < 5, [1, 3, 5, 2, 4]))  # [1, 3]
list(itertools.dropwhile(lambda x: x < 5, [1, 3, 5, 2, 4]))  # [5, 2, 4]
```

---

## 7. Classes & Object-Oriented Programming

### Class Definition

```python
class BankAccount:
    """A simple bank account."""

    bank_name = "Python Bank"     # class variable — shared by all instances
    _accounts = []                # class variable — convention: internal use

    def __init__(self, owner: str, balance: float = 0.0):
        # Instance variables — unique to each instance
        self.owner = owner
        self._balance = balance   # _prefix: convention for "internal"
        self.__id = id(self)      # __prefix: name-mangled to _BankAccount__id

    def deposit(self, amount: float) -> float:
        """Instance method — receives self as first argument."""
        if amount <= 0:
            raise ValueError(f"Deposit amount must be positive, got {amount}")
        self._balance += amount
        return self._balance

    @classmethod
    def from_dict(cls, data: dict) -> "BankAccount":
        """Class method — alternative constructor. cls = the class itself."""
        return cls(data["owner"], data.get("balance", 0.0))

    @staticmethod
    def validate_amount(amount: float) -> bool:
        """Static method — no access to class or instance. Pure utility."""
        return amount > 0

    @property
    def balance(self) -> float:
        """Property — access as attribute, computed on get."""
        return self._balance

    @balance.setter
    def balance(self, value: float) -> None:
        """Property setter — validates before setting."""
        if value < 0:
            raise ValueError("Balance cannot be negative")
        self._balance = value

    @balance.deleter
    def balance(self) -> None:
        """Property deleter — called on del account.balance."""
        self._balance = 0.0

    def __repr__(self) -> str:
        return f"BankAccount(owner={self.owner!r}, balance={self._balance})"

    def __str__(self) -> str:
        return f"{self.owner}'s account: ${self._balance:.2f}"

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, BankAccount):
            return NotImplemented
        return self.owner == other.owner and self._balance == other._balance

# Usage
acc = BankAccount("Alice", 1000.0)
acc.deposit(500)          # instance method
BankAccount.from_dict({"owner": "Bob"})  # class method
BankAccount.validate_amount(100)         # static method
acc.balance               # property getter → 1500.0
acc.balance = 2000.0      # property setter
print(acc)                # __str__: "Alice's account: $2000.00"
repr(acc)                 # __repr__: "BankAccount(owner='Alice', balance=2000.0)"
```

---

### All Key Dunder Methods

```python
class Container:
    def __init__(self): self.data = []

    # String representation
    def __repr__(self): return f"Container({self.data!r})"  # unambiguous
    def __str__(self): return f"Container with {len(self.data)} items"

    # Size and boolean
    def __len__(self): return len(self.data)         # len(c)
    def __bool__(self): return bool(self.data)       # bool(c), if c:

    # Comparison (if you define __eq__, consider __hash__ too)
    def __eq__(self, other): return self.data == other.data  # c == other
    def __lt__(self, other): return len(self) < len(other)   # c < other
    def __hash__(self): return hash(tuple(self.data))        # hash(c)

    # Arithmetic
    def __add__(self, other):                        # c + other
        result = Container()
        result.data = self.data + other.data
        return result
    def __iadd__(self, other):                       # c += other
        self.data.extend(other.data)
        return self

    # Container protocol
    def __contains__(self, item): return item in self.data    # item in c
    def __getitem__(self, idx): return self.data[idx]         # c[idx]
    def __setitem__(self, idx, val): self.data[idx] = val     # c[idx] = val
    def __delitem__(self, idx): del self.data[idx]            # del c[idx]

    # Iteration protocol
    def __iter__(self): return iter(self.data)       # for x in c
    def __next__(self): ...                          # next(c)

    # Context manager protocol
    def __enter__(self): return self                 # with c as x:
    def __exit__(self, exc_type, exc_val, tb): return False  # False = don't suppress exceptions

    # Callable
    def __call__(self, *args): ...                   # c()

    # Attribute access
    def __getattr__(self, name): ...                 # c.missing_attr (only called if normal lookup fails)
    def __setattr__(self, name, val): object.__setattr__(self, name, val)  # c.attr = val
    def __getattribute__(self, name): ...            # EVERY attribute access (use carefully!)
```

---

### Inheritance and MRO

```python
class Animal:
    def __init__(self, name: str):
        self.name = name

    def speak(self) -> str:
        raise NotImplementedError

    def __repr__(self):
        return f"{type(self).__name__}({self.name!r})"

class Dog(Animal):               # single inheritance
    def speak(self) -> str:
        return f"{self.name} says Woof!"

class Cat(Animal):
    def speak(self) -> str:
        return f"{self.name} says Meow!"

class GuideDog(Dog):             # multi-level inheritance
    def guide(self):
        return f"{self.name} is guiding"

# super() — call parent class method
class Puppy(Dog):
    def __init__(self, name: str, age: int):
        super().__init__(name)   # call Dog.__init__ (which calls Animal.__init__)
        self.age = age

    def speak(self) -> str:
        return super().speak() + " (puppy)"   # extends parent behavior

# Multiple inheritance — MRO (C3 Linearisation)
class A:
    def method(self): return "A"

class B(A):
    def method(self): return "B" + super().method()

class C(A):
    def method(self): return "C" + super().method()

class D(B, C):     # inherits from B and C — both inherit from A
    pass

D.mro()            # [D, B, C, A, object] — Method Resolution Order
D().method()       # "BCA" — follows MRO, each super() calls next in chain

# Mixin pattern — add functionality via multiple inheritance
class JsonMixin:
    def to_json(self):
        import json
        return json.dumps(self.__dict__)

class LogMixin:
    def log(self, message):
        print(f"[{type(self).__name__}] {message}")

class MyModel(JsonMixin, LogMixin):
    def __init__(self, name):
        self.name = name

m = MyModel("test")
m.to_json()              # from JsonMixin
m.log("hello")           # from LogMixin
```

---

### Abstract Base Classes

```python
from abc import ABC, abstractmethod

class Shape(ABC):                    # cannot be instantiated directly
    @abstractmethod
    def area(self) -> float:         # must be implemented by subclasses
        ...

    @abstractmethod
    def perimeter(self) -> float:
        ...

    def describe(self) -> str:       # concrete method (can be inherited)
        return f"{type(self).__name__}: area={self.area():.2f}"

class Circle(Shape):
    def __init__(self, radius: float):
        self.radius = radius

    def area(self) -> float:         # must implement abstract method
        import math
        return math.pi * self.radius ** 2

    def perimeter(self) -> float:
        import math
        return 2 * math.pi * self.radius

# Shape()  # TypeError: Can't instantiate abstract class Shape
c = Circle(5)
c.area()          # 78.54...
c.describe()      # "Circle: area=78.54"
isinstance(c, Shape)  # True
```

---

### Dataclasses

```python
from dataclasses import dataclass, field, KW_ONLY

@dataclass
class Point:
    x: float
    y: float
    z: float = 0.0           # field with default

@dataclass
class Config:
    host: str = "localhost"
    port: int = 8080
    tags: list = field(default_factory=list)   # mutable default — use field()!
    metadata: dict = field(default_factory=dict)
    _cache: dict = field(default_factory=dict, repr=False, compare=False)

    def __post_init__(self):            # runs after __init__
        if self.port < 0 or self.port > 65535:
            raise ValueError(f"Invalid port: {self.port}")
        self.host = self.host.lower()   # normalise on creation

# Frozen dataclass — immutable (like a named tuple with type hints)
@dataclass(frozen=True)
class ImmutablePoint:
    x: float
    y: float
    # p.x = 10 → FrozenInstanceError

# Slots dataclass (Python 3.10+) — memory efficient
@dataclass(slots=True)
class SlottedPoint:
    x: float
    y: float

# KW_ONLY sentinel (Python 3.10+) — force keyword-only fields
@dataclass
class Server:
    host: str
    _: KW_ONLY                 # everything after this is keyword-only
    port: int = 8080
    timeout: float = 30.0

Server("localhost", port=9000)   # host is positional, rest are keyword-only

# Inheritance
@dataclass
class Point3D(Point):
    z: float = 0.0             # adds z field; x, y inherited from Point
```

---

### \_\_slots\_\_

```python
# __slots__ prevents per-instance __dict__, saving memory for many instances

class WithoutSlots:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    # Each instance has a __dict__ — flexible but uses more memory

class WithSlots:
    __slots__ = ("x", "y")    # only these attributes allowed

    def __init__(self, x, y):
        self.x = x
        self.y = y
    # No __dict__ — fixed attributes, ~40% memory savings for many instances

import sys
a = WithoutSlots(1, 2)
b = WithSlots(1, 2)
sys.getsizeof(a)    # ~48 bytes for object + dict overhead
sys.getsizeof(b)    # ~56 bytes but NO separate dict allocation

# ⚠️ Slots caveat — can't add arbitrary attributes
b.z = 3             # AttributeError: 'WithSlots' object has no attribute 'z'

# Multiple inheritance with slots — all bases must define __slots__
```

---

### Protocols — Structural Subtyping

```python
from typing import Protocol, runtime_checkable

# Protocol defines an interface by structure, not inheritance
@runtime_checkable
class Drawable(Protocol):
    def draw(self) -> None: ...         # must have a draw() method
    def get_color(self) -> str: ...     # and a get_color() method

class Circle:                           # does NOT inherit from Drawable
    def draw(self) -> None:
        print("Drawing circle")
    def get_color(self) -> str:
        return "red"

class Square:
    def draw(self) -> None:
        print("Drawing square")
    def get_color(self) -> str:
        return "blue"

def render(shape: Drawable) -> None:    # accepts anything with the right methods
    shape.draw()
    print(f"Color: {shape.get_color()}")

render(Circle())    # works — Circle has the required methods
render(Square())    # works — Square has the required methods

# isinstance check (requires @runtime_checkable)
isinstance(Circle(), Drawable)   # True
```

---

### namedtuple vs dataclass vs TypedDict

```python
# namedtuple: immutable, tuple-like, low memory, no methods
from collections import namedtuple
Point = namedtuple("Point", ["x", "y"])
p = Point(1, 2)
p[0]       # 1 — tuple indexing works
p.x        # 1 — field access works
# p.x = 3  # AttributeError — immutable

# dataclass: mutable by default, can have methods, proper class
from dataclasses import dataclass
@dataclass
class DataPoint:
    x: float
    y: float
    def distance(self): return (self.x**2 + self.y**2)**0.5

dp = DataPoint(3, 4)
dp.x = 10            # mutable
dp.distance()        # 5.0

# TypedDict: for dict-like structures (not a class, just a type hint)
from typing import TypedDict
class Movie(TypedDict):
    title: str
    year: int
    rating: float

m: Movie = {"title": "Inception", "year": 2010, "rating": 8.8}
# NOT a class instance — still a plain dict at runtime
# Only used for type checking

# When to use each:
# namedtuple  → immutable records, tuple compatibility needed, memory-critical
# dataclass   → general-purpose, needs methods/mutability/inheritance
# TypedDict   → typed dicts passed around as dicts (APIs, JSON data)
```

---

## 8. Comprehensions & Functional Patterns

```python
# --- List comprehensions ---
squares = [x**2 for x in range(10)]
# [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

# With condition (filter)
even_squares = [x**2 for x in range(10) if x % 2 == 0]
# [0, 4, 16, 36, 64]

# Nested (matrix flattening)
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flat = [x for row in matrix for x in row]
# [1, 2, 3, 4, 5, 6, 7, 8, 9]

# Nested with condition
pairs = [(x, y) for x in range(3) for y in range(3) if x != y]

# --- Dict comprehensions ---
word_lengths = {word: len(word) for word in ["cat", "elephant", "dog"]}
# {'cat': 3, 'elephant': 8, 'dog': 3}

inverted = {v: k for k, v in {"a": 1, "b": 2}.items()}
# {1: 'a', 2: 'b'}

# Conditional transformation
data = {"a": 1, "b": -2, "c": 3, "d": -4}
positives = {k: v for k, v in data.items() if v > 0}
# {'a': 1, 'c': 3}

# --- Set comprehensions ---
unique_lengths = {len(word) for word in ["cat", "dog", "elephant", "ox"]}
# {2, 3, 8}

# --- Generator expressions (lazy) ---
# Uses () instead of [] — no list built in memory
total = sum(x**2 for x in range(1_000_000))    # computed lazily, never stores all values

gen = (x**2 for x in range(5))
next(gen)    # 0 — one at a time
list(gen)    # [1, 4, 9, 16] — remaining values

# Memory comparison:
import sys
list_comp = [x**2 for x in range(1000)]
gen_exp   = (x**2 for x in range(1000))
sys.getsizeof(list_comp)  # ~8056 bytes — stores all values
sys.getsizeof(gen_exp)    # ~112  bytes — stores only generator state

# ⚠️ When NOT to use comprehensions — readability threshold
# Too complex = use a regular loop

# Hard to read:
result = [transform(x) for x in data if condition(x) and other_condition(x)]

# Better as a loop:
result = []
for x in data:
    if condition(x) and other_condition(x):
        result.append(transform(x))

# --- Functional patterns ---
nums = [3, 1, 4, 1, 5, 9, 2, 6]

# sorted with key — works on any iterable, returns new list
sorted(nums)                        # [1, 1, 2, 3, 4, 5, 6, 9]
sorted(nums, reverse=True)          # [9, 6, 5, 4, 3, 2, 1, 1]
sorted(["banana", "Apple", "cherry"], key=str.lower)   # case-insensitive sort

# min/max with key
words = ["cat", "elephant", "ox", "dog"]
min(words, key=len)     # "ox"  — shortest word
max(words, key=len)     # "elephant" — longest word

# map — apply function to each element (returns iterator)
doubled = list(map(lambda x: x * 2, nums))
# [6, 2, 8, 2, 10, 18, 4, 12]
# ✅ Prefer list comprehension: [x * 2 for x in nums]

# filter — keep elements where function returns True (returns iterator)
evens = list(filter(lambda x: x % 2 == 0, nums))
# ✅ Prefer: [x for x in nums if x % 2 == 0]

# zip — pair elements from multiple iterables
names  = ["Alice", "Bob", "Carol"]
scores = [95, 87, 91]
dict(zip(names, scores))    # {'Alice': 95, 'Bob': 87, 'Carol': 91}
```

---

## 9. Modules & Packages

### import Variants

```python
# All forms of import:
import os                          # import module — access as os.path.join()
import os.path                     # import submodule
from os import path                # import specific name
from os.path import join, exists   # import multiple names
from os import *                   # import all public names (⚠️ pollutes namespace)
import numpy as np                 # import with alias
from pathlib import Path as P      # import with alias

# __name__ guard — code only runs when file is executed directly
if __name__ == "__main__":
    print("Running as script")     # NOT printed when imported as module
    main()
```

---

### How Imports Work

```python
# 1. Python checks sys.modules cache (already imported?)
# 2. If not cached: find module in sys.path
# 3. Load and execute module code
# 4. Cache in sys.modules
# 5. Bind name in current namespace

import sys
sys.modules             # dict of all imported modules
sys.path                # list of directories to search for modules
# ['', '/usr/lib/python312.zip', '/usr/lib/python3.12', ...]

# Inspect the cache
"os" in sys.modules     # True (if already imported)
sys.modules["os"]       # <module 'os' from ...>

# .pyc files: compiled bytecode cached in __pycache__/
# module.cpython-312.pyc — auto-regenerated when source changes

# importlib — dynamic/programmatic imports
import importlib
mod = importlib.import_module("json")   # import by string name
importlib.reload(mod)                   # reload a module (rare — use with caution)
```

---

### Packages

```python
# Package = directory with __init__.py
# mypackage/
# ├── __init__.py        ← makes it a package
# ├── core.py
# └── utils/
#     ├── __init__.py
#     └── helpers.py

# __init__.py can:
# 1. Be empty (minimal package)
# 2. Import sub-modules for convenience
# 3. Define __all__ to control "from package import *"

# mypackage/__init__.py
from .core import MyClass           # relative import — . means same package
from .utils.helpers import helper   # relative import into subpackage
__all__ = ["MyClass", "helper"]     # public API

# Relative imports (only work inside packages)
from . import sibling_module        # import from same package
from .. import parent_module        # import from parent package
from ..other import something       # import from sibling package

# Namespace packages (Python 3.3+) — packages WITHOUT __init__.py
# Allows splitting a package across multiple directories
```

---

### Circular Imports — Causes and Solutions

```python
# Problem: module A imports B, module B imports A → circular dependency

# a.py:
from b import func_b    # imports b.py which...
def func_a(): return "A"

# b.py:
from a import func_a    # ...imports a.py — circular!
def func_b(): return func_a()

# Solution 1: import inside function (deferred import)
# b.py:
def func_b():
    from a import func_a   # import at call time, not module load time
    return func_a()

# Solution 2: import module, not name
# b.py:
import a                   # import module reference
def func_b():
    return a.func_a()      # access name at call time

# Solution 3: restructure — extract shared code to a third module
# common.py: (new)
def shared_logic(): ...

# a.py: from common import shared_logic
# b.py: from common import shared_logic
```

---

### Key stdlib Modules

```python
import os               # OS interface: file/dir ops, env vars, process
import sys              # system-specific params: argv, path, exit()
from pathlib import Path # OOP file system paths (prefer over os.path)
import shutil           # high-level file ops: copy, move, rmtree
import glob             # Unix-style pathname pattern expansion
import re               # regular expressions
import json             # JSON serialisation/deserialisation
import csv              # CSV file reading/writing
from datetime import datetime, date, timedelta  # dates and times
import collections      # Counter, defaultdict, OrderedDict, deque, namedtuple
import itertools        # infinite iterators, combinatorics
import functools        # higher-order functions: lru_cache, partial, reduce
import math             # mathematical functions: sqrt, floor, ceil, pi, e
import random           # random numbers, choices, shuffle, sample
import hashlib          # cryptographic hash functions: md5, sha256
import logging          # flexible logging framework
import argparse         # command-line argument parsing
import subprocess       # spawn and manage subprocesses
import threading        # thread-based parallelism
import multiprocessing  # process-based parallelism
import asyncio          # async I/O, event loop
from typing import ...  # type hint support
from dataclasses import dataclass, field  # data classes
from enum import Enum, auto             # enumerations
import contextlib       # context manager utilities
import inspect          # runtime introspection: signature, source, members
import copy             # shallow and deep copy
import pickle           # Python object serialisation
import struct           # pack/unpack binary data
import io               # in-memory file-like objects
import time             # time access: sleep, perf_counter, time
import tempfile         # temporary files and directories
import textwrap         # text wrapping and filling
import string           # string constants and Template
import unicodedata      # Unicode database
```

---

## 10. Error Handling & Exceptions

### Exception Hierarchy

```
BaseException
├── SystemExit              ← sys.exit() raises this
├── KeyboardInterrupt       ← Ctrl+C
├── GeneratorExit           ← generator.close() called
└── Exception               ← all "normal" exceptions inherit from here
    ├── StopIteration       ← iterator exhausted
    ├── StopAsyncIteration
    ├── ArithmeticError
    │   ├── ZeroDivisionError
    │   ├── OverflowError
    │   └── FloatingPointError
    ├── LookupError
    │   ├── IndexError      ← list[99] out of range
    │   └── KeyError        ← dict["missing"]
    ├── AttributeError      ← obj.no_such_attr
    ├── NameError           ← undefined_variable
    ├── TypeError           ← wrong type for operation
    ├── ValueError          ← right type, wrong value
    ├── OSError (IOError)   ← file/system errors
    │   ├── FileNotFoundError
    │   ├── PermissionError
    │   └── TimeoutError
    ├── RuntimeError
    │   ├── RecursionError
    │   └── NotImplementedError
    ├── ImportError
    │   └── ModuleNotFoundError
    ├── UnicodeError
    │   └── UnicodeDecodeError, UnicodeEncodeError
    └── AssertionError      ← assert statement failed
```

---

### try/except Patterns

```python
# Pattern 1: catch specific exceptions (always prefer specific over broad)
try:
    value = int(user_input)
except ValueError:
    print("Not a valid integer")

# Pattern 2: catch multiple exceptions
try:
    data = json.loads(text)["key"]
except json.JSONDecodeError:
    print("Invalid JSON")
except KeyError:
    print("Key not found")

# Pattern 3: catch multiple in one clause
try:
    risky()
except (TypeError, ValueError) as e:
    print(f"Input error: {e}")

# Pattern 4: else (runs only if NO exception was raised)
try:
    result = compute()
except Exception as e:
    print(f"Error: {e}")
else:
    print(f"Success: {result}")   # only runs on success
finally:
    cleanup()                      # ALWAYS runs

# Pattern 5: re-raise
try:
    dangerous()
except ValueError as e:
    log_error(e)
    raise           # re-raise the same exception with original traceback

# Pattern 6: transform exception
try:
    connect_to_db()
except ConnectionError as e:
    raise RuntimeError("Database unavailable") from e   # chain exceptions

# Pattern 7: suppress traceback chain
try:
    value = d["key"]
except KeyError:
    raise ValueError("key not found") from None   # hides original KeyError

# Pattern 8: examine exception info
import sys, traceback
try:
    risky()
except Exception:
    exc_type, exc_value, exc_tb = sys.exc_info()
    traceback.print_exception(exc_type, exc_value, exc_tb)
```

---

### Custom Exceptions

```python
# Always inherit from Exception (or a subclass), not BaseException
class AppError(Exception):
    """Base exception for this application."""

class ValidationError(AppError):
    """Raised when input validation fails."""
    def __init__(self, field: str, message: str):
        self.field = field
        self.message = message
        super().__init__(f"{field}: {message}")   # set human-readable message

class NotFoundError(AppError):
    """Raised when a resource is not found."""
    def __init__(self, resource: str, id: int):
        self.resource = resource
        self.id = id
        super().__init__(f"{resource} with id={id} not found")

# Usage
try:
    raise ValidationError("email", "must contain @")
except ValidationError as e:
    print(e.field)    # "email"
    print(str(e))     # "email: must contain @"

# ExceptionGroup (Python 3.11+) — group multiple exceptions
try:
    raise ExceptionGroup("multiple errors", [
        ValueError("bad value"),
        TypeError("bad type"),
    ])
except* ValueError as eg:    # except* handles ExceptionGroup
    print(f"ValueError(s): {eg.exceptions}")
except* TypeError as eg:
    print(f"TypeError(s): {eg.exceptions}")
```

---

### contextlib Utilities

```python
from contextlib import suppress, contextmanager, ExitStack

# suppress — silently ignore specific exceptions
with suppress(FileNotFoundError, PermissionError):
    os.remove("/tmp/maybe_exists.txt")   # no error if file doesn't exist or no permission

# contextmanager — create context manager from a generator
@contextmanager
def managed_resource(name):
    print(f"Acquiring {name}")
    resource = acquire(name)
    try:
        yield resource                 # provide resource to 'with' block
    except Exception as e:
        print(f"Error during {name}: {e}")
        raise
    finally:
        print(f"Releasing {name}")
        release(resource)              # always release

with managed_resource("database") as db:
    db.query("SELECT 1")

# ExitStack — dynamic stack of context managers
with ExitStack() as stack:
    files = [stack.enter_context(open(f)) for f in file_list]
    # All files automatically closed on exit, even if one fails

# Warnings
import warnings
warnings.warn("This is deprecated", DeprecationWarning, stacklevel=2)
warnings.warn("Performance issue", RuntimeWarning)

# Filter warnings
warnings.filterwarnings("ignore", category=DeprecationWarning)
warnings.filterwarnings("error",  category=RuntimeWarning)   # treat as error
```

---

## 11. File I/O & Pathlib

### open() and File Operations

```python
# open(file, mode='r', encoding=None, newline=None)
# Modes: r=read, w=write, a=append, x=exclusive create, b=binary, +=update

# Always use context manager — guarantees file is closed
with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()           # read entire file as string
    # OR:
    lines = f.readlines()        # read all lines into list
    # OR:
    for line in f:               # iterate line by line (memory efficient!)
        print(line.rstrip())     # strip trailing newline

with open("data.txt", "r") as f:
    first_line = f.readline()    # read single line

# Writing
with open("output.txt", "w", encoding="utf-8") as f:
    f.write("Hello, World!\n")          # write string (no auto-newline)
    f.writelines(["line1\n", "line2\n"]) # write list of strings
    print("Hello", file=f)              # print to file

# Appending
with open("log.txt", "a") as f:
    f.write(f"{datetime.now()}: event\n")

# Binary mode
with open("image.png", "rb") as f:
    data = f.read()                # read as bytes

with open("image.png", "wb") as f:
    f.write(binary_data)           # write bytes
```

---

### pathlib.Path

```python
from pathlib import Path

# Construction
p = Path("/home/alice/projects")
p = Path.home() / "projects"           # ~ expansion
p = Path.cwd()                          # current working directory
p = Path("relative/path")              # relative path

# Navigation — / operator joins paths
config = Path("/etc") / "nginx" / "nginx.conf"
src = Path("src") / "mypackage" / "core.py"

# Path properties
p = Path("/home/alice/projects/app.py")
p.name          # "app.py"           — filename with extension
p.stem          # "app"              — filename without extension
p.suffix        # ".py"              — extension
p.suffixes      # [".tar", ".gz"] for "archive.tar.gz"
p.parent        # Path("/home/alice/projects")
p.parents       # [Path("/home/alice/projects"), Path("/home/alice"), ...]
p.parts         # ('/', 'home', 'alice', 'projects', 'app.py')
p.is_absolute() # True
p.is_relative_to("/home")  # True

# Checking existence
p.exists()      # True if path exists
p.is_file()     # True if it's a file
p.is_dir()      # True if it's a directory
p.is_symlink()  # True if it's a symlink

# File system operations
p.mkdir(parents=True, exist_ok=True)   # create directory (+ parents)
p.touch()                              # create empty file / update timestamp
p.rename(new_path)                     # rename/move
p.unlink()                             # delete file
p.unlink(missing_ok=True)             # delete, no error if missing (3.8+)
p.rmdir()                              # remove EMPTY directory
import shutil
shutil.rmtree(p)                       # remove directory tree

# Reading and writing (no open() needed!)
text = p.read_text(encoding="utf-8")   # read entire file as str
data = p.read_bytes()                  # read entire file as bytes
p.write_text("content", encoding="utf-8")  # write string (overwrites)
p.write_bytes(b"binary data")              # write bytes (overwrites)

# Globbing
list(Path(".").glob("*.py"))           # all .py files in current dir
list(Path(".").glob("**/*.py"))        # all .py files recursively
list(Path(".").rglob("*.py"))          # rglob = recursive glob (shorter)

# Iterating directory
for item in Path("/etc").iterdir():    # list directory contents
    print(item.name, "DIR" if item.is_dir() else "FILE")

# File info
stat = p.stat()
stat.st_size      # file size in bytes
stat.st_mtime     # modification time (Unix timestamp)

# Resolve — get absolute, canonical path
Path("../config").resolve()    # /abs/path/to/config

# Convert
str(p)            # "/home/alice/projects/app.py"
p.as_uri()        # "file:///home/alice/projects/app.py"
```

---

### CSV and JSON

```python
import csv, json

# --- CSV ---
# Write CSV
with open("data.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.writer(f)
    writer.writerow(["name", "age", "city"])      # header
    writer.writerows([
        ["Alice", 30, "NYC"],
        ["Bob",   25, "LA"],
    ])

# Read CSV
with open("data.csv", "r", encoding="utf-8") as f:
    reader = csv.reader(f)
    header = next(reader)                          # skip header row
    for row in reader:
        name, age, city = row                      # unpack

# DictReader/DictWriter — rows as dicts
with open("data.csv") as f:
    reader = csv.DictReader(f)                     # uses first row as fieldnames
    for row in reader:
        print(row["name"], row["age"])             # access by column name

with open("out.csv", "w", newline="") as f:
    fieldnames = ["name", "age"]
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerow({"name": "Alice", "age": 30})

# --- JSON ---
import json

# Serialise Python → JSON string
data = {"name": "Alice", "scores": [95, 87], "active": True, "meta": None}
json_str = json.dumps(data)                # compact string
json_str = json.dumps(data, indent=2)      # pretty-printed
json_str = json.dumps(data, sort_keys=True) # sorted keys

# Deserialise JSON string → Python
parsed = json.loads(json_str)              # string → Python dict/list/...

# File I/O
with open("data.json", "w") as f:
    json.dump(data, f, indent=2)           # write to file

with open("data.json") as f:
    loaded = json.load(f)                  # read from file

# Custom encoder/decoder
from datetime import datetime

class DateTimeEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()          # convert to ISO string
        return super().default(obj)         # let base class handle others

json.dumps({"time": datetime.now()}, cls=DateTimeEncoder)

# Custom decoder
def date_hook(d):
    for key, value in d.items():
        try:
            d[key] = datetime.fromisoformat(value)
        except (ValueError, TypeError):
            pass
    return d

json.loads(json_str, object_hook=date_hook)

# shutil — high-level file operations
import shutil
shutil.copy("src.txt", "dst.txt")          # copy file
shutil.copy2("src.txt", "dst.txt")         # copy + preserve metadata
shutil.copytree("src_dir", "dst_dir")      # copy directory tree
shutil.move("src", "dst")                  # move file or directory
shutil.rmtree("directory")                 # delete directory tree ⚠️
shutil.make_archive("archive", "zip", "directory")  # create zip
shutil.unpack_archive("archive.zip", "dest/")

# tempfile — temporary files
import tempfile
with tempfile.TemporaryFile(mode="w+") as f:   # auto-deleted on close
    f.write("temp data")
    f.seek(0)
    f.read()

with tempfile.NamedTemporaryFile(suffix=".json", delete=False) as f:
    f.write(b'{"key": "value"}')
    tmp_path = f.name    # get the path

with tempfile.TemporaryDirectory() as tmpdir:  # auto-deleted on exit
    tmp = Path(tmpdir) / "work.txt"
    tmp.write_text("working")
```

---

## 12. Concurrency & Parallelism

### Conceptual Model

```
Concurrency vs Parallelism vs Async:

Concurrency:  multiple tasks make progress (not necessarily simultaneously)
Parallelism:  multiple tasks run simultaneously (multiple CPU cores)
Async:        cooperative multitasking — tasks yield control voluntarily

                  ┌──────────────────────────────────────┐
                  │     When to use what?                │
                  │                                      │
  I/O-bound tasks │──▶ threading  OR  asyncio           │
  CPU-bound tasks │──▶ multiprocessing                  │
  High concurrency│──▶ asyncio (thousands of connections)│
  Simple parallel │──▶ concurrent.futures               │
                  └──────────────────────────────────────┘
```

---

### Threading

```python
import threading
import time

# Basic thread
def worker(name, delay):
    print(f"{name} starting")
    time.sleep(delay)         # GIL released during sleep — real concurrency
    print(f"{name} done")

t1 = threading.Thread(target=worker, args=("T1", 1), daemon=True)
t2 = threading.Thread(target=worker, args=("T2", 2))
t1.start()
t2.start()
t1.join()   # wait for t1 to finish
t2.join()   # wait for t2 to finish

# Lock — prevent race conditions
counter = 0
lock = threading.Lock()

def increment():
    global counter
    with lock:                # acquire lock, release on exit (even on exception)
        counter += 1

# RLock — reentrant lock (same thread can acquire multiple times)
rlock = threading.RLock()
with rlock:
    with rlock:               # OK — same thread can acquire again
        pass

# Event — signal between threads
event = threading.Event()

def waiter():
    print("Waiting for event...")
    event.wait()              # blocks until event.set() is called
    print("Event received!")

def setter():
    time.sleep(2)
    event.set()               # unblocks all waiters

# Queue — thread-safe producer/consumer
from queue import Queue
q = Queue(maxsize=10)

def producer():
    for i in range(5):
        q.put(i)              # blocks if queue is full
    q.put(None)               # sentinel to signal done

def consumer():
    while True:
        item = q.get()        # blocks until item available
        if item is None:
            break
        print(f"Processing: {item}")
        q.task_done()         # mark item as processed

# Semaphore — limit concurrent access
sem = threading.Semaphore(3)  # max 3 concurrent holders

def limited_resource():
    with sem:                 # acquire (count - 1), release (count + 1)
        time.sleep(1)         # at most 3 threads here simultaneously
```

---

### multiprocessing

```python
import multiprocessing as mp
from multiprocessing import Process, Pool, Queue, Pipe

# Basic Process
def cpu_bound(n):
    return sum(i**2 for i in range(n))  # pure Python — benefits from multiprocessing

if __name__ == "__main__":   # ⚠️ REQUIRED on Windows/macOS (fork safety)
    p = Process(target=cpu_bound, args=(1_000_000,))
    p.start()
    p.join()

    # Pool — manage a pool of worker processes
    with Pool(processes=4) as pool:           # 4 worker processes
        results = pool.map(cpu_bound, [1_000_000] * 8)   # map over args
        # All 8 tasks distributed across 4 processes

        # starmap — map with multiple args per call
        results = pool.starmap(pow, [(2, 10), (3, 5), (4, 4)])

        # apply_async — non-blocking submission
        async_result = pool.apply_async(cpu_bound, (500_000,))
        value = async_result.get(timeout=10)  # get result (blocks until ready)

    # Queue — share data between processes (not the same as threading.Queue)
    q = mp.Queue()

    def producer(q):
        q.put("data")

    def consumer(q):
        item = q.get()
        print(item)

    # Pipe — fast 2-way communication between 2 processes
    parent_conn, child_conn = Pipe()

    def child_process(conn):
        data = conn.recv()      # receive from parent
        conn.send(data * 2)     # send back

    p = Process(target=child_process, args=(child_conn,))
    p.start()
    parent_conn.send(42)        # send to child
    result = parent_conn.recv() # receive from child → 84
    p.join()

    # shared_memory (Python 3.8+) — zero-copy shared data
    from multiprocessing import shared_memory
    shm = shared_memory.SharedMemory(create=True, size=1024)
    # ... access via shm.buf (memoryview)
    shm.close()
    shm.unlink()   # delete shared memory segment
```

---

### concurrent.futures

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
from concurrent.futures import as_completed, wait, FIRST_COMPLETED
import urllib.request

# ThreadPoolExecutor — I/O-bound tasks
def fetch(url):
    with urllib.request.urlopen(url) as r:
        return r.read()

urls = ["http://example.com"] * 10

with ThreadPoolExecutor(max_workers=5) as executor:
    # submit() — non-blocking, returns Future
    futures = [executor.submit(fetch, url) for url in urls]

    # map() — like built-in map but concurrent, results in order
    results = list(executor.map(fetch, urls))

    # as_completed — process results as they finish (out of order)
    for future in as_completed(futures):
        try:
            result = future.result()    # get result (or raise exception)
            print(f"Got {len(result)} bytes")
        except Exception as e:
            print(f"Failed: {e}")

# ProcessPoolExecutor — CPU-bound tasks
def crunch(n):
    return sum(i**2 for i in range(n))

with ProcessPoolExecutor(max_workers=4) as executor:
    futures = {executor.submit(crunch, n): n for n in range(4)}
    for future in as_completed(futures):
        n = futures[future]             # which input produced this result
        print(f"crunch({n}) = {future.result()}")
```

---

### asyncio

```python
import asyncio

# async def defines a coroutine — doesn't run until awaited
async def fetch_data(url: str) -> str:
    print(f"Fetching {url}...")
    await asyncio.sleep(1)      # non-blocking sleep — yields control
    return f"Data from {url}"

async def main():
    # Sequential — each awaits before next starts
    r1 = await fetch_data("url1")   # 1 second
    r2 = await fetch_data("url2")   # another 1 second → 2s total

    # Concurrent — all start together with gather
    results = await asyncio.gather(
        fetch_data("url1"),
        fetch_data("url2"),
        fetch_data("url3"),
    )                               # all 3 run concurrently → ~1s total
    print(results)

    # Tasks — schedule coroutines without immediately awaiting
    task1 = asyncio.create_task(fetch_data("url1"))
    task2 = asyncio.create_task(fetch_data("url2"))
    # Both start immediately
    result1 = await task1
    result2 = await task2

    # Timeout
    try:
        result = await asyncio.wait_for(fetch_data("slow"), timeout=0.5)
    except asyncio.TimeoutError:
        print("Timed out!")

    # asyncio.wait — more control over completion
    done, pending = await asyncio.wait(
        {task1, task2},
        return_when=asyncio.FIRST_COMPLETED    # return when first finishes
    )

# Run the event loop
asyncio.run(main())    # creates event loop, runs main(), closes loop

# Async context manager
class AsyncDB:
    async def __aenter__(self):
        await asyncio.sleep(0.1)    # async connection setup
        return self

    async def __aexit__(self, *args):
        await asyncio.sleep(0)      # async cleanup
        pass

async def use_db():
    async with AsyncDB() as db:     # async with
        pass

# Async iterator / async for
class AsyncRange:
    def __init__(self, stop):
        self.stop = stop
        self.current = 0

    def __aiter__(self):
        return self

    async def __anext__(self):
        if self.current >= self.stop:
            raise StopAsyncIteration
        await asyncio.sleep(0)      # simulate async work
        value = self.current
        self.current += 1
        return value

async def iterate():
    async for i in AsyncRange(5):   # async for
        print(i)

# asyncio primitives
async def producer_consumer():
    queue = asyncio.Queue(maxsize=5)
    lock  = asyncio.Lock()
    event = asyncio.Event()

    async def producer():
        for i in range(10):
            await queue.put(i)      # blocks if queue full
        await queue.put(None)       # sentinel

    async def consumer():
        while True:
            item = await queue.get()
            if item is None:
                break
            async with lock:        # async lock
                process(item)

    await asyncio.gather(producer(), consumer())
```

---

## 13. Type Hints & Static Analysis

### Basic Annotations

```python
# Variable annotations
name: str = "Alice"
count: int = 0
ratio: float = 3.14
active: bool = True
data: list[str] = []             # Python 3.9+ built-in generic syntax
mapping: dict[str, int] = {}

# Function annotations
def greet(name: str, times: int = 1) -> str:
    return (f"Hello, {name}!\n" * times).strip()

# Unannotated — type: ignore
def mystery(x):          # no annotations — x has implicit type 'Any'
    return x
```

---

### typing Module

```python
from typing import (
    Optional, Union, Any, Callable,
    List, Dict, Tuple, Set,          # legacy (3.5-3.8) — prefer list[int] in 3.9+
    Iterator, Generator, Iterable,
    Sequence, Mapping, MutableMapping,
    TypeVar, Generic,
    Literal, Final, ClassVar,
    TypedDict, NamedTuple,
    overload, TYPE_CHECKING,
    NewType, TypeAlias,
)

# Optional — value can be T or None
def find(items: list[str], target: str) -> Optional[str]:  # str | None
    for item in items:
        if item == target:
            return item
    return None

# Union — value can be any of these types (use X | Y in Python 3.10+)
def process(value: Union[int, str]) -> str:   # old
def process(value: int | str) -> str: ...     # Python 3.10+ (preferred)

# Any — opts out of type checking (use sparingly)
def flexible(x: Any) -> Any:
    return x

# Callable — callable with specific signature
from typing import Callable
Handler = Callable[[str, int], bool]    # takes str and int, returns bool

def register(handler: Callable[[str], None]) -> None:
    pass

# TypeVar — generic functions
T = TypeVar("T")

def first(items: list[T]) -> T:         # preserves the type
    return items[0]

first([1, 2, 3])      # → int
first(["a", "b"])     # → str

# TypeVar with bound
Comparable = TypeVar("Comparable", bound="SupportsLessThan")

# Generic class
from typing import Generic
class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()

s: Stack[int] = Stack()
s.push(1)
s.push(2)

# Literal — specific values only
from typing import Literal
Mode = Literal["r", "w", "a", "rb", "wb"]
def open_file(path: str, mode: Mode) -> None: ...

# Final — cannot be reassigned
from typing import Final
MAX_SIZE: Final = 100
# MAX_SIZE = 200  # mypy error

# ClassVar — class variable, not instance variable
class Config:
    DEBUG: ClassVar[bool] = False    # shared across all instances

# TypedDict — dict with specific key types
class Movie(TypedDict):
    title: str
    year: int
    rating: float

# total=False — all keys optional
class PartialMovie(TypedDict, total=False):
    title: str
    year: int

# TYPE_CHECKING — avoid circular imports, import only for type checking
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from mymodule import MyClass     # only imported during type checking, not runtime

def func(x: "MyClass") -> None:     # string annotation (forward reference)
    pass

# NewType — create distinct types
UserId = NewType("UserId", int)
user_id = UserId(42)                # UserId, not plain int
user_id + 1                         # OK — UserId is still an int at runtime

# TypeAlias (Python 3.10+)
from typing import TypeAlias
Vector: TypeAlias = list[float]
ConnectionPool: TypeAlias = dict[str, list["Connection"]]

# Modern type alias (Python 3.12+)
type Vector = list[float]           # new syntax

# @overload — multiple type signatures for one function
from typing import overload

@overload
def stringify(x: int) -> str: ...
@overload
def stringify(x: float) -> str: ...
@overload
def stringify(x: list[int]) -> list[str]: ...

def stringify(x):           # actual implementation (no @overload)
    if isinstance(x, list):
        return [str(i) for i in x]
    return str(x)

# Protocol — structural typing
from typing import Protocol
class Sized(Protocol):
    def __len__(self) -> int: ...

def get_length(x: Sized) -> int:    # any object with __len__ qualifies
    return len(x)
```

---

### Static Analysis Tools

```bash
# mypy — type checker (PEP 484)
pip install mypy
mypy mypackage/           # type-check all files
mypy --strict mypackage/  # enable all strict checks
mypy --ignore-missing-imports mypackage/

# pyright — Microsoft's type checker (fast, used by Pylance)
pip install pyright
pyright mypackage/

# ruff — extremely fast linter + formatter (replaces flake8, isort, many others)
pip install ruff
ruff check .              # lint
ruff check . --fix        # auto-fix fixable issues
ruff format .             # format (like black)
```

---

## 14. Testing

### pytest

```python
# tests/test_math.py
import pytest

# Basic test function — pytest discovers test_* functions
def test_addition():
    assert 1 + 1 == 2             # pytest rewrites assert for better output

def test_raises():
    with pytest.raises(ValueError) as exc_info:
        int("not a number")
    assert "invalid literal" in str(exc_info.value)

def test_raises_specific():
    with pytest.raises(ZeroDivisionError, match="division by zero"):
        1 / 0

# Parametrize — run test with multiple inputs
@pytest.mark.parametrize("input,expected", [
    (1, 1),
    (2, 4),
    (3, 9),
    (-1, 1),
    (0, 0),
])
def test_square(input, expected):
    assert input ** 2 == expected   # runs 5 separate tests

# Fixtures — reusable setup/teardown
@pytest.fixture
def db_connection():
    conn = create_test_db()        # setup
    yield conn                     # provide to test
    conn.close()                   # teardown (after yield)

@pytest.fixture(scope="module")    # shared across all tests in module
def expensive_resource():
    resource = setup_expensive()
    yield resource
    teardown_expensive(resource)

def test_query(db_connection):     # fixture injected by name
    result = db_connection.query("SELECT 1")
    assert result == [(1,)]

# conftest.py — shared fixtures available to all tests in directory
# Put fixtures used by multiple test files here

# Marks — categorise tests
@pytest.mark.slow
@pytest.mark.integration
def test_slow_operation():
    pass

# Skip tests
@pytest.mark.skip(reason="Not implemented yet")
def test_future():
    pass

@pytest.mark.skipif(sys.platform == "win32", reason="Unix only")
def test_unix_feature():
    pass

# xfail — expected to fail
@pytest.mark.xfail(reason="Known bug #123")
def test_broken():
    assert broken_function() == expected
```

---

### Mocking

```python
from unittest.mock import Mock, MagicMock, patch, call

# Mock — basic mock object
mock = Mock()
mock.method()           # any call works — returns Mock
mock.method(1, 2, key="val")
mock.method.assert_called_once_with(1, 2, key="val")
mock.method.call_count  # 2
mock.method.call_args   # call(1, 2, key='val')

# Return values
mock.compute.return_value = 42
mock.compute()          # 42

# Side effects
mock.fetch.side_effect = ConnectionError("timeout")
# mock.fetch() → raises ConnectionError

mock.counter.side_effect = [1, 2, 3]  # returns different values each call

# patch — replace object in a module during test
with patch("mymodule.requests.get") as mock_get:
    mock_get.return_value.status_code = 200
    mock_get.return_value.json.return_value = {"data": "value"}
    result = mymodule.fetch_data("http://example.com")
    mock_get.assert_called_once_with("http://example.com")

# patch as decorator
@patch("mymodule.os.path.exists", return_value=True)
@patch("mymodule.open", MagicMock(read_data="file content"))
def test_file_read(mock_exists, mock_open):
    result = mymodule.read_config()
    assert result is not None

# patch.object — patch attribute on an object
with patch.object(MyClass, "method", return_value="mocked"):
    obj = MyClass()
    assert obj.method() == "mocked"

# MagicMock — like Mock but supports magic methods
magic = MagicMock()
len(magic)          # 0 (magic.__len__ is pre-configured)
magic["key"]        # returns MagicMock (magic.__getitem__ works)
```

---

### doctest and Coverage

```python
# doctest — runnable examples in docstrings
def add(a, b):
    """
    Add two numbers together.

    >>> add(2, 3)
    5
    >>> add(-1, 1)
    0
    >>> add(0.1, 0.2)  # doctest: +ELLIPSIS
    0.30000...
    """
    return a + b

# Run doctests:
# python -m doctest module.py
# pytest --doctest-modules
```

```bash
# Coverage
pip install pytest-cov

pytest --cov=mypackage --cov-report=html   # HTML report
pytest --cov=mypackage --cov-report=term-missing  # show missed lines
pytest --cov=mypackage --cov-fail-under=80  # fail if coverage < 80%

# .coveragerc
# [run]
# source = mypackage
# omit = tests/*

# hypothesis — property-based testing
pip install hypothesis
from hypothesis import given, strategies as st

@given(st.integers(), st.integers())
def test_commutative(a, b):
    assert a + b == b + a    # hypothesis generates 100s of random inputs

@given(st.lists(st.integers(), min_size=1))
def test_max_in_list(lst):
    assert max(lst) in lst
    assert all(x <= max(lst) for x in lst)

# pytest-asyncio — test async code
pip install pytest-asyncio

import pytest
@pytest.mark.asyncio
async def test_async_function():
    result = await my_async_function()
    assert result == expected
```

---

## 15. Performance & Profiling

### Benchmarking and Profiling

```python
# timeit — benchmark expressions
import timeit

# Quick benchmark
timeit.timeit('"-".join(str(i) for i in range(100))', number=10000)
timeit.timeit('"-".join([str(i) for i in range(100)])', number=10000)

# In code
timer = timeit.Timer(
    stmt="[x**2 for x in range(1000)]",
    setup="pass"
)
print(timer.timeit(number=1000))    # run 1000 times, report total

# cProfile — CPU profiling
import cProfile, pstats
profiler = cProfile.Profile()
profiler.enable()
your_function()
profiler.disable()
stats = pstats.Stats(profiler)
stats.sort_stats("cumulative")      # sort by cumulative time
stats.print_stats(20)               # top 20 functions

# Command line
# python -m cProfile -s cumulative my_script.py

# memory_profiler (install: pip install memory-profiler)
from memory_profiler import profile

@profile
def memory_heavy():
    big_list = [0] * 1_000_000     # allocate 1M integers
    return sum(big_list)

# line_profiler (install: pip install line-profiler)
from line_profiler import LineProfiler

lp = LineProfiler()
lp_wrapper = lp(my_function)
lp_wrapper()
lp.print_stats()
```

---

### Performance Patterns

```python
# ⚠️ Anti-pattern: string concatenation in loops
def slow_join(items):
    result = ""
    for item in items:
        result += str(item)   # creates new string each iteration — O(n²)
    return result

# ✅ Fast: use join
def fast_join(items):
    return "".join(str(item) for item in items)   # O(n)

# ⚠️ Anti-pattern: global lookup in tight loop
import math
def slow_sqrt(nums):
    return [math.sqrt(x) for x in nums]   # looks up math, then sqrt each time

# ✅ Fast: cache local reference
def fast_sqrt(nums):
    sqrt = math.sqrt    # one lookup, reuse
    return [sqrt(x) for x in nums]

# ✅ List comprehension vs loop (comprehensions are faster)
slow = []
for x in range(1000):
    slow.append(x**2)         # append called 1000 times

fast = [x**2 for x in range(1000)]  # ~2x faster — optimised in C

# ✅ Generator vs list when you don't need all values at once
total = sum(x**2 for x in range(1_000_000))  # never stores the list

# ✅ set for membership testing
lookup_list = list(range(100_000))
lookup_set  = set(lookup_list)
99_999 in lookup_list   # O(n) — slow
99_999 in lookup_set    # O(1) — fast

# ✅ dict.get() instead of try/except for simple defaults
d = {"a": 1}
# Slow (exception overhead when key missing):
try:
    v = d["b"]
except KeyError:
    v = 0

# Fast:
v = d.get("b", 0)     # O(1), no exception

# NumPy — vectorised operations for numeric data
import numpy as np

# ⚠️ Python loop (slow for large arrays)
result = [x * 2 for x in range(1_000_000)]

# ✅ NumPy vectorised (C-speed, no Python loop)
arr = np.arange(1_000_000)
result = arr * 2    # ~100x faster

# __slots__ for memory-efficient classes (see Section 7)

# array module — compact storage for homogeneous numeric data
import array
arr = array.array("i", range(1000))   # 'i' = signed int — much less memory than list
```

---

## 16. Pythonic Idioms & Best Practices

### PEP 8 — Style Guide

```python
# Naming conventions
snake_case_function = True         # functions, variables, modules
CamelCase = True                   # classes
SCREAMING_SNAKE_CASE = True        # constants
_single_leading_underscore = True  # internal/private
__double_leading = True            # name mangling (strong "private")
__dunder__ = True                  # magic methods (don't invent new ones)

# Imports — at top of file, in this order:
import os                          # 1. standard library
import sys                         # 1. standard library

import requests                    # 2. third-party (blank line before)
import numpy as np

from mypackage import utils        # 3. local/project imports (blank line before)

# Line length: 88 characters (Black default), 79 (PEP 8 strict)
# Blank lines: 2 between top-level definitions, 1 between methods
```

---

### PEP 20 — The Zen of Python

```python
import this   # prints The Zen of Python

# Key principles:
# Beautiful is better than ugly.
# Explicit is better than implicit.
# Simple is better than complex.
# Readability counts.
# Errors should never pass silently — unless explicitly silenced.
# There should be one obvious way to do it.
# If the implementation is hard to explain, it's a bad idea.
```

---

### EAFP vs LBYL

```python
# LBYL — Look Before You Leap (C-style)
# Check for conditions before performing action
if "key" in d:          # check first
    value = d["key"]    # then access

# EAFP — Easier to Ask Forgiveness than Permission (Pythonic)
# Try the action and handle failure
try:
    value = d["key"]    # try directly
except KeyError:
    value = default     # handle failure

# EAFP is preferred in Python — more readable, handles race conditions better

# Duck typing — don't check type, check behavior
# ⚠️ Not Pythonic:
def process(data):
    if type(data) == list:      # overly strict
        return len(data)

# ✅ Pythonic (duck typing):
def process(data):
    try:
        return len(data)        # works for list, tuple, str, dict, any Sized
    except TypeError:
        return None

# ✅ Or use isinstance for legitimate type checks (accepts subclasses):
if isinstance(data, (list, tuple)):
    pass
```

---

### Unpacking Patterns

```python
# Basic unpacking
a, b, c = [1, 2, 3]
x, y = y, x                         # swap without temp variable

# Extended unpacking (Python 3)
first, *rest = [1, 2, 3, 4, 5]      # first=1, rest=[2,3,4,5]
*init, last = [1, 2, 3, 4, 5]       # init=[1,2,3,4], last=5
a, *b, c = [1, 2, 3, 4, 5]         # a=1, b=[2,3,4], c=5

# Nested unpacking
(a, b), c = (1, 2), 3               # a=1, b=2, c=3
[[a, b], [c, d]] = [[1, 2], [3, 4]] # a=1, b=2, c=3, d=4

# Unpacking in for loops
pairs = [(1, "a"), (2, "b"), (3, "c")]
for num, letter in pairs:           # unpack each tuple in loop
    print(num, letter)

# Ignore values with _
x, _, z = (1, 2, 3)                 # ignore middle value
a, *_, b = range(10)                # a=0, b=9 (ignore middle)

# Function call unpacking
def func(a, b, c):
    pass
args = (1, 2, 3)
func(*args)                          # unpack as positional
kwargs = {"a": 1, "b": 2, "c": 3}
func(**kwargs)                       # unpack as keyword

# Merge dicts with unpacking (Python 3.5+)
d1 = {"a": 1}
d2 = {"b": 2}
merged = {**d1, **d2}               # {"a": 1, "b": 2}
merged = d1 | d2                    # Python 3.9+ — cleaner!
```

---

### Common Anti-Patterns to Avoid

```python
# ⚠️ 1. Mutable default argument — shared across all calls!
def bad_append(item, lst=[]):   # lst is created ONCE at definition time
    lst.append(item)
    return lst

bad_append(1)   # [1]
bad_append(2)   # [1, 2] — NOT [2]! Same list reused!

# ✅ Fix: use None as default
def good_append(item, lst=None):
    if lst is None:
        lst = []            # create new list each call
    lst.append(item)
    return lst

# ⚠️ 2. Bare except — catches EVERYTHING including KeyboardInterrupt, SystemExit
try:
    risky()
except:             # catches Ctrl+C, sys.exit(), everything — BAD
    pass

# ✅ Fix: catch specific exceptions
try:
    risky()
except Exception:   # catches all "normal" exceptions, not SystemExit/KeyboardInterrupt
    pass

# ⚠️ 3. type() instead of isinstance() — doesn't handle subclasses
if type(x) == int:          # breaks if x is bool (bool is subclass of int)
    pass

# ✅ Fix: use isinstance
if isinstance(x, int):      # accepts int and any subclass
    pass

# ⚠️ 4. String concatenation in loops
result = ""
for word in words:
    result += word          # O(n²) — creates new string each iteration

# ✅ Fix: join
result = "".join(words)     # O(n)

# ⚠️ 5. Comparing to True/False/None with ==
if x == True:  pass         # wrong
if x == None:  pass         # wrong

# ✅ Fix
if x:          pass         # truthiness check
if x is None:  pass         # identity check

# ⚠️ 6. Using range(len(x)) when you need values
for i in range(len(items)):
    print(items[i])         # needlessly clunky

# ✅ Fix
for item in items:          # direct iteration
    print(item)

for i, item in enumerate(items):  # when you need the index too
    print(i, item)
```

---

### Dictionary Patterns

```python
d = {"a": 1, "b": 2}

# get() with default — avoid KeyError
value = d.get("missing", "default")    # "default" (no exception)

# setdefault — add key only if absent
d.setdefault("c", []).append(3)        # creates d["c"] = [] if absent, then appends

# Merge dicts (Python 3.9+)
combined = d | {"c": 3, "d": 4}       # new dict
d |= {"c": 3}                          # merge in place

# Build dict from two lists
keys   = ["a", "b", "c"]
values = [1, 2, 3]
d = dict(zip(keys, values))            # {"a": 1, "b": 2, "c": 3}

# Invert a dict
inverted = {v: k for k, v in d.items()}  # {1: "a", 2: "b", 3: "c"}

# Group items by key
from collections import defaultdict
words = ["apple", "ant", "banana", "bear", "cherry"]
by_letter = defaultdict(list)
for word in words:
    by_letter[word[0]].append(word)
# {"a": ["apple", "ant"], "b": ["banana", "bear"], "c": ["cherry"]}
```

---

## 17. Modern Python Features (3.8–3.13)

### Python 3.8

```python
# Walrus operator :=
while chunk := f.read(8192):
    process(chunk)

# f-string = specifier (debug format)
x = 42
print(f"{x=}")           # "x=42" — variable name + value
print(f"{x * 2 = }")    # "x * 2 = 84"

# Positional-only parameters with /
def greet(name, /, greeting="Hello"):  # name MUST be positional
    return f"{greeting}, {name}"

greet("Alice")                         # OK
greet(name="Alice")                    # TypeError: name is positional-only
```

---

### Python 3.9

```python
# Dict merge/update operators
d1 = {"a": 1, "b": 2}
d2 = {"b": 3, "c": 4}
d1 | d2              # {"a": 1, "b": 3, "c": 4} — new dict, d2 wins
d1 |= d2             # d1 updated in place

# Built-in generics (no more from typing import List, Dict, etc.)
def func(items: list[int], mapping: dict[str, float]) -> tuple[int, ...]:
    pass

# Type hints with built-in types
x: list[str]         = []
y: dict[str, int]    = {}
z: tuple[int, ...]   = ()
w: set[str]          = set()

# zoneinfo — IANA timezone database
from zoneinfo import ZoneInfo
from datetime import datetime
dt = datetime.now(ZoneInfo("America/New_York"))
dt = datetime.now(ZoneInfo("Europe/London"))
```

---

### Python 3.10

```python
# Structural pattern matching (see Section 5 for full coverage)
match command:
    case {"action": "move", "direction": d}:
        move(d)
    case {"action": "quit"}:
        quit()

# X | Y union syntax (no more Optional[X] or Union[X, Y])
def func(x: int | str | None) -> int | None:
    pass

# Better error messages
# Python 3.10+ gives helpful suggestions:
# NameError: name 'pritn' is not defined. Did you mean: 'print'?
# AttributeError: 'list' has no attribute 'apend'. Did you mean: 'append'?

# parenthesized context managers
with (open("a") as fa, open("b") as fb):  # parentheses allow line wrapping
    pass
```

---

### Python 3.11

```python
# ExceptionGroup and except*
try:
    raise ExceptionGroup("oops", [ValueError("v"), TypeError("t")])
except* ValueError as eg:    # except* handles ExceptionGroup
    print(f"ValueErrors: {eg.exceptions}")
except* TypeError as eg:
    print(f"TypeErrors: {eg.exceptions}")

# TaskGroup in asyncio (structured concurrency)
import asyncio
async def main():
    async with asyncio.TaskGroup() as tg:  # Python 3.11+
        task1 = tg.create_task(fetch(url1))
        task2 = tg.create_task(fetch(url2))
    # All tasks done here; exceptions propagated via ExceptionGroup

# tomllib — read TOML files (built-in)
import tomllib
with open("pyproject.toml", "rb") as f:
    config = tomllib.load(f)

# Self type — refer to current class in annotations
from typing import Self

class Builder:
    def set_name(self, name: str) -> Self:  # returns same class (including subclasses)
        self.name = name
        return self

# Python 3.11 is ~25% faster than 3.10 (specialising adaptive interpreter)
```

---

### Python 3.12

```python
# Type parameter syntax (PEP 695) — cleaner generics
# Old:
from typing import TypeVar
T = TypeVar("T")
def first(items: list[T]) -> T: ...

# New (Python 3.12+):
def first[T](items: list[T]) -> T: ...    # [T] is the type parameter

class Stack[T]:
    def push(self, item: T) -> None: ...
    def pop(self) -> T: ...

# Type alias (PEP 695)
# Old: from typing import TypeAlias; Vector: TypeAlias = list[float]
type Vector = list[float]       # new syntax — clearly a type alias

# @override decorator
from typing import override

class Parent:
    def method(self) -> int: ...

class Child(Parent):
    @override               # signals intent to override — mypy checks it exists in parent
    def method(self) -> int: ...

# Improved f-strings — allow any valid Python expression
data = {"key": "value"}
print(f"{data["key"]}")    # nested quotes now allowed (was syntax error in 3.11)
print(f"{'x'!r}")          # conversion specifiers work anywhere
```

---

### Python 3.13

```python
# Free-threaded mode (experimental — PEP 703) — no GIL!
# Build Python with: ./configure --disable-gil
# Run with:  python -X gil=0 script.py
# Or set:    PYTHON_GIL=0

# True thread-based parallelism for CPU-bound code (experimental)
import threading
results = []
lock = threading.Lock()

def compute(n):
    value = sum(i**2 for i in range(n))
    with lock:
        results.append(value)

threads = [threading.Thread(target=compute, args=(500_000,)) for _ in range(4)]
for t in threads: t.start()
for t in threads: t.join()
# In free-threaded mode: runs in parallel on 4 cores

# JIT compiler (experimental — PEP 744)
# python -X jit script.py
# Automatically optimises hot paths — no code changes needed

# Improved locals() — behaviour now defined as returning a fresh snapshot
def demo():
    x = 1
    l = locals()    # captures snapshot
    x = 2
    # In 3.13: l["x"] == 1 (snapshot), not 2

# Better interactive interpreter (REPL)
# python  → new REPL with multi-line editing, colours, history search
```

---

## 18. Quick Reference Tables

### Built-in Functions

| Function | Description | Example |
|----------|-------------|---------|
| `abs(x)` | Absolute value | `abs(-5)` → `5` |
| `all(it)` | True if all truthy | `all([1,2,3])` → `True` |
| `any(it)` | True if any truthy | `any([0,1,0])` → `True` |
| `bin(x)` | Binary string | `bin(10)` → `'0b1010'` |
| `bool(x)` | Convert to bool | `bool(0)` → `False` |
| `breakpoint()` | Drop into debugger | - |
| `callable(x)` | True if callable | `callable(len)` → `True` |
| `chr(i)` | Int → Unicode char | `chr(65)` → `'A'` |
| `compile()` | Compile source to code | - |
| `delattr(o,n)` | Delete attribute | `delattr(obj, 'x')` |
| `dir(x)` | List attributes | `dir([])` |
| `divmod(a,b)` | `(a//b, a%b)` | `divmod(7,3)` → `(2,1)` |
| `enumerate(it)` | Index-value pairs | `enumerate(['a'])` |
| `eval(s)` | Evaluate expression | `eval('2+2')` → `4` |
| `filter(f,it)` | Filter iterable | `filter(None,[0,1,2])` |
| `float(x)` | Convert to float | `float('3.14')` |
| `format(v,s)` | Format value | `format(3.14,'.1f')` → `'3.1'` |
| `frozenset(it)` | Immutable set | `frozenset([1,2,2])` |
| `getattr(o,n)` | Get attribute | `getattr(obj,'x',default)` |
| `globals()` | Global namespace dict | - |
| `hasattr(o,n)` | Has attribute? | `hasattr(obj, 'x')` |
| `hash(x)` | Hash value | `hash('hello')` |
| `help(x)` | Interactive help | `help(str)` |
| `hex(x)` | Hex string | `hex(255)` → `'0xff'` |
| `id(x)` | Object identity | `id(obj)` |
| `input(s)` | Read stdin | `input('Name: ')` |
| `int(x)` | Convert to int | `int('42')` → `42` |
| `isinstance(o,t)` | Instance check | `isinstance(1, int)` |
| `issubclass(c,t)` | Subclass check | `issubclass(bool, int)` |
| `iter(x)` | Get iterator | `iter([1,2,3])` |
| `len(x)` | Length | `len([1,2,3])` → `3` |
| `list(it)` | Convert to list | `list(range(3))` |
| `locals()` | Local namespace dict | - |
| `map(f,it)` | Map function | `map(str, [1,2])` |
| `max(it)` | Maximum | `max([1,3,2])` → `3` |
| `min(it)` | Minimum | `min([1,3,2])` → `1` |
| `next(it)` | Next item | `next(iter([1,2]))` → `1` |
| `object()` | Base object | - |
| `oct(x)` | Octal string | `oct(8)` → `'0o10'` |
| `open(f)` | Open file | `open('f.txt')` |
| `ord(c)` | Char → int | `ord('A')` → `65` |
| `pow(b,e)` | Power | `pow(2,10)` → `1024` |
| `print(*a)` | Print to stdout | `print('hi', end='')` |
| `property()` | Property descriptor | `@property` |
| `range(n)` | Integer range | `range(1,10,2)` |
| `repr(x)` | Repr string | `repr([1,2])` → `'[1, 2]'` |
| `reversed(it)` | Reversed iterator | `reversed([1,2,3])` |
| `round(x,n)` | Round number | `round(3.14159,2)` → `3.14` |
| `set(it)` | Convert to set | `set([1,1,2])` |
| `setattr(o,n,v)` | Set attribute | `setattr(obj,'x',1)` |
| `slice(s,e)` | Slice object | `slice(1,5,2)` |
| `sorted(it)` | Sorted list | `sorted([3,1,2])` |
| `staticmethod` | Static method | `@staticmethod` |
| `str(x)` | Convert to string | `str(42)` → `'42'` |
| `sum(it)` | Sum iterable | `sum([1,2,3])` → `6` |
| `super()` | Parent class proxy | `super().__init__()` |
| `tuple(it)` | Convert to tuple | `tuple([1,2])` |
| `type(x)` | Get type | `type(42)` → `<class 'int'>` |
| `vars(x)` | `__dict__` of object | `vars(obj)` |
| `zip(*its)` | Zip iterables | `zip([1,2],['a','b'])` |
| `__import__` | Import machinery | - |

---

### String Methods Reference

| Method | Description | Example |
|--------|-------------|---------|
| `s.upper()` | All uppercase | `"hi".upper()` → `"HI"` |
| `s.lower()` | All lowercase | `"HI".lower()` → `"hi"` |
| `s.title()` | Title case | `"hello world".title()` → `"Hello World"` |
| `s.capitalize()` | First char upper | `"hello".capitalize()` → `"Hello"` |
| `s.strip(c)` | Strip chars | `"  hi  ".strip()` → `"hi"` |
| `s.lstrip(c)` | Left strip | `"  hi".lstrip()` → `"hi"` |
| `s.rstrip(c)` | Right strip | `"hi  ".rstrip()` → `"hi"` |
| `s.split(sep)` | Split string | `"a,b".split(",")` → `["a","b"]` |
| `s.rsplit(sep,n)` | Split from right | `"a.b.c".rsplit(".",1)` |
| `s.splitlines()` | Split on newlines | `"a\nb".splitlines()` |
| `sep.join(it)` | Join iterable | `",".join(["a","b"])` → `"a,b"` |
| `s.replace(o,n)` | Replace substring | `"hi".replace("h","H")` |
| `s.find(sub)` | Find index (or -1) | `"hello".find("ll")` → `2` |
| `s.index(sub)` | Find index (raises) | `"hello".index("ll")` → `2` |
| `s.count(sub)` | Count occurrences | `"banana".count("a")` → `3` |
| `s.startswith(p)` | Starts with? | `"hello".startswith("he")` |
| `s.endswith(s)` | Ends with? | `"hello".endswith("lo")` |
| `s.isdigit()` | All digits? | `"123".isdigit()` → `True` |
| `s.isalpha()` | All letters? | `"abc".isalpha()` → `True` |
| `s.isalnum()` | Letters or digits? | `"abc1".isalnum()` → `True` |
| `s.isspace()` | All whitespace? | `"  ".isspace()` → `True` |
| `s.isupper()` | All uppercase? | `"HI".isupper()` → `True` |
| `s.islower()` | All lowercase? | `"hi".islower()` → `True` |
| `s.zfill(w)` | Zero-pad | `"42".zfill(5)` → `"00042"` |
| `s.center(w,c)` | Center-pad | `"hi".center(6,"-")` → `"--hi--"` |
| `s.ljust(w,c)` | Left-justify | `"hi".ljust(5)` → `"hi   "` |
| `s.rjust(w,c)` | Right-justify | `"hi".rjust(5)` → `"   hi"` |
| `s.encode(e)` | str → bytes | `"hi".encode("utf-8")` |
| `s.format(...)` | Format string | `"{} {}".format(1,2)` |
| `s.expandtabs(n)` | Replace tabs | `"a\tb".expandtabs(4)` |
| `s.removeprefix(p)` | Remove prefix (3.9+) | `"Hello".removeprefix("He")` |
| `s.removesuffix(s)` | Remove suffix (3.9+) | `"Hello".removesuffix("lo")` |
| `s.maketrans(x)` | Make trans table | `str.maketrans("aeiou","*****")` |
| `s.translate(t)` | Translate chars | `"hello".translate(t)` |

---

### List Methods with Time Complexity

| Method | Description | Time |
|--------|-------------|------|
| `l.append(x)` | Add to end | O(1) amortised |
| `l.extend(it)` | Add all from iterable | O(k) |
| `l.insert(i, x)` | Insert at index | O(n) |
| `l.pop()` | Remove last | O(1) |
| `l.pop(i)` | Remove at index | O(n) |
| `l.remove(x)` | Remove first x | O(n) |
| `l.index(x)` | Find index of x | O(n) |
| `l.count(x)` | Count occurrences | O(n) |
| `l.sort()` | Sort in place | O(n log n) |
| `l.reverse()` | Reverse in place | O(n) |
| `l.copy()` | Shallow copy | O(n) |
| `l.clear()` | Remove all | O(n) |
| `x in l` | Membership test | O(n) |
| `l[i]` | Index access | O(1) |
| `l[i:j]` | Slicing | O(k) |
| `len(l)` | Length | O(1) |
| `sorted(l)` | New sorted list | O(n log n) |

---

### Dict Methods Reference

| Method | Description | Example |
|--------|-------------|---------|
| `d[k]` | Get value (KeyError if missing) | `d["a"]` |
| `d.get(k, default)` | Get with default | `d.get("x", 0)` |
| `d[k] = v` | Set value | `d["a"] = 1` |
| `del d[k]` | Delete key | `del d["a"]` |
| `d.pop(k)` | Remove and return | `d.pop("a")` |
| `d.pop(k, default)` | Safe pop | `d.pop("x", None)` |
| `d.popitem()` | Remove last item (3.7+) | `d.popitem()` |
| `d.setdefault(k, v)` | Set if absent | `d.setdefault("x", [])` |
| `d.update(other)` | Merge dict in | `d.update({"b": 2})` |
| `d.keys()` | Keys view | `list(d.keys())` |
| `d.values()` | Values view | `list(d.values())` |
| `d.items()` | Key-value pairs view | `for k,v in d.items()` |
| `d.copy()` | Shallow copy | `copy = d.copy()` |
| `d.clear()` | Remove all | `d.clear()` |
| `k in d` | Key membership | `"a" in d` |
| `d \| other` | Merge (new dict, 3.9+) | `d \| {"c": 3}` |
| `d \|= other` | Merge in place (3.9+) | `d \|= {"c": 3}` |
| `len(d)` | Number of keys | `len(d)` |

---

### Operator Precedence (Highest to Lowest)

| Precedence | Operators | Notes |
|-----------|-----------|-------|
| 1 | `()`, `[]`, `{}` | Parentheses, brackets |
| 2 | `x[i]`, `x.attr`, `f()` | Subscript, attribute, call |
| 3 | `**` | Exponentiation (right-associative) |
| 4 | `+x`, `-x`, `~x` | Unary |
| 5 | `*`, `@`, `/`, `//`, `%` | Mul, matmul, div, floordiv, mod |
| 6 | `+`, `-` | Add, sub |
| 7 | `<<`, `>>` | Bitwise shifts |
| 8 | `&` | Bitwise AND |
| 9 | `^` | Bitwise XOR |
| 10 | `\|` | Bitwise OR |
| 11 | `in`, `not in`, `is`, `is not`, `<`, `<=`, `>`, `>=`, `==`, `!=` | Comparisons |
| 12 | `not x` | Boolean NOT |
| 13 | `and` | Boolean AND |
| 14 | `or` | Boolean OR |
| 15 | `x if C else y` | Conditional |
| 16 | `lambda` | Lambda |
| 17 | `:=` | Walrus |

---

### Common itertools Functions

| Function | Description | Example |
|----------|-------------|---------|
| `chain(*its)` | Concatenate iterables | `chain([1,2],[3,4])` → `1,2,3,4` |
| `chain.from_iterable(it)` | Chain from nested | `chain.from_iterable([[1,2],[3]])` |
| `islice(it,n)` | Lazy slice | `islice(range(100),5,10)` |
| `product(*its)` | Cartesian product | `product('AB', [1,2])` |
| `permutations(it,r)` | All r-length orderings | `permutations('ABC',2)` |
| `combinations(it,r)` | r-length combos (no repeat) | `combinations('ABC',2)` |
| `combinations_with_replacement` | combos with repeats | - |
| `groupby(it, key)` | Group consecutive | `groupby(data, key=lambda x:x[0])` |
| `accumulate(it)` | Running totals | `accumulate([1,2,3])` → `1,3,6` |
| `cycle(it)` | Infinite cycling | `cycle('AB')` → `A,B,A,B,...` |
| `repeat(x, n)` | Repeat value | `repeat(0, 3)` → `0,0,0` |
| `takewhile(f, it)` | Take while True | `takewhile(lambda x:x<5, [1,3,5])` |
| `dropwhile(f, it)` | Drop while True | `dropwhile(lambda x:x<5, [1,3,5,2])` |
| `filterfalse(f, it)` | Filter where False | `filterfalse(None, [0,1,0,2])` |
| `starmap(f, it)` | Map with unpacking | `starmap(pow, [(2,3),(3,2)])` |
| `zip_longest(*its)` | Zip, pad shorter | `zip_longest('AB','xyz',fillvalue='-')` |
| `count(start, step)` | Infinite counter | `count(10, 2)` → `10,12,14,...` |
| `pairwise(it)` | Overlapping pairs (3.10+) | `pairwise('ABCD')` → `AB,BC,CD` |
| `batched(it, n)` | Batches of n (3.12+) | `batched('ABCDE',2)` → `AB,CD,E` |

---

### datetime Format Codes

| Code | Meaning | Example |
|------|---------|---------|
| `%Y` | 4-digit year | `2024` |
| `%y` | 2-digit year | `24` |
| `%m` | Month (01-12) | `03` |
| `%B` | Full month name | `March` |
| `%b` | Short month name | `Mar` |
| `%d` | Day (01-31) | `15` |
| `%H` | Hour 24h (00-23) | `14` |
| `%I` | Hour 12h (01-12) | `02` |
| `%M` | Minute (00-59) | `30` |
| `%S` | Second (00-59) | `45` |
| `%f` | Microseconds | `000000` |
| `%p` | AM/PM | `PM` |
| `%A` | Full weekday | `Friday` |
| `%a` | Short weekday | `Fri` |
| `%j` | Day of year | `074` |
| `%Z` | Timezone name | `UTC` |
| `%z` | UTC offset | `+0000` |
| `%c` | Locale datetime | `Fri Mar 15 14:30:45 2024` |
| `%x` | Locale date | `03/15/24` |
| `%X` | Locale time | `14:30:45` |
| `%s` | Unix timestamp | `1710509445` |

```python
from datetime import datetime
now = datetime.now()
now.strftime("%Y-%m-%d %H:%M:%S")   # "2024-03-15 14:30:45"
datetime.strptime("2024-03-15", "%Y-%m-%d")  # parse string to datetime
datetime.fromisoformat("2024-03-15T14:30:45")  # ISO 8601 parse
now.isoformat()                      # "2024-03-15T14:30:45.123456"
```

---

### Type Hints: Old vs New Syntax

| Old (3.5-3.8) | New (3.9+/3.10+) | Description |
|---------------|------------------|-------------|
| `List[int]` | `list[int]` | List of ints |
| `Dict[str, int]` | `dict[str, int]` | Dict |
| `Tuple[int, str]` | `tuple[int, str]` | Tuple |
| `Set[int]` | `set[int]` | Set |
| `FrozenSet[int]` | `frozenset[int]` | Frozenset |
| `Type[MyClass]` | `type[MyClass]` | Class itself |
| `Optional[str]` | `str \| None` | Optional |
| `Union[int, str]` | `int \| str` | Union |
| `Callable[[int], str]` | `Callable[[int], str]` | Callable (unchanged) |
| `TypeVar("T")` | `[T]` syntax (3.12+) | Type variable |
| `TypeAlias` | `type X = ...` (3.12+) | Type alias |
| `from __future__ import annotations` | Not needed in 3.10+ | Postponed evaluation |

---

*Generated with ❤️ — For Python 3.12+ (notes for 3.8+ compatibility throughout)*
*Official docs: [docs.python.org](https://docs.python.org) | PEPs: [peps.python.org](https://peps.python.org)*
