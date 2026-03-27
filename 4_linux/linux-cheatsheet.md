# 🐧 Linux Comprehensive Cheatsheet

> A complete reference for Linux — covering theory, filesystems, shell scripting, permissions, processes, networking, security, containers, and kernel internals.

---

## 📚 Table of Contents

1. [Core Concepts & Theory](#1-core-concepts--theory)
2. [Filesystem Hierarchy & Navigation](#2-filesystem-hierarchy--navigation)
3. [File Operations](#3-file-operations)
4. [Text Processing](#4-text-processing)
5. [Shell & Scripting](#5-shell--scripting)
6. [Permissions & Ownership](#6-permissions--ownership)
7. [Users & Groups](#7-users--groups)
8. [Processes & Jobs](#8-processes--jobs)
9. [System Monitoring & Performance](#9-system-monitoring--performance)
10. [Networking](#10-networking)
11. [Storage & Filesystems](#11-storage--filesystems)
12. [Package Management](#12-package-management)
13. [Services & systemd](#13-services--systemd)
14. [SSH & Remote Access](#14-ssh--remote-access)
15. [Security & Hardening](#15-security--hardening)
16. [Containers & Virtualisation](#16-containers--virtualisation)
17. [Kernel & Boot](#17-kernel--boot)
18. [Quick Reference Tables](#18-quick-reference-tables)

---

## 1. Core Concepts & Theory

### What is Linux?

```
Linux (the word) is overloaded — it means different things in different contexts:

┌─────────────────────────────────────────────────────┐
│                  Linux Distribution                 │
│  (Ubuntu, Fedora, Arch, Alpine...)                  │
├─────────────────────────────────────────────────────┤
│            User Space (GNU tools, apps,             │
│            libraries, init system, shell)           │
├─────────────────────────────────────────────────────┤
│         System Call Interface (syscalls)            │
├─────────────────────────────────────────────────────┤
│         Linux Kernel (the actual "Linux")           │
│  Process mgmt | Memory | Drivers | FS | Network    │
├─────────────────────────────────────────────────────┤
│               Hardware (CPU, RAM, Disk)             │
└─────────────────────────────────────────────────────┘

Kernel:       The core program managing hardware and resources
OS:           Kernel + GNU tools + libraries + init system
Distribution: OS + package manager + configuration + desktop/server apps
```

**The Linux Kernel** manages five core subsystems:
- **Process management** — scheduling, creating/terminating processes
- **Memory management** — virtual memory, paging, swapping, the OOM killer
- **Device drivers** — abstraction layer between hardware and software
- **Virtual filesystem (VFS)** — unified interface over ext4, xfs, btrfs, etc.
- **Networking** — TCP/IP stack, sockets, firewall (netfilter)

---

### Monolithic vs Microkernel

```
Monolithic kernel (Linux):          Microkernel (Minix, GNU Hurd):
┌──────────────────────┐            ┌──────────────────────┐
│  KERNEL SPACE        │            │  USER SPACE          │
│  ┌────────────────┐  │            │  FS server           │
│  │ Scheduler      │  │            │  Driver servers      │
│  │ Memory manager │  │            │  Network server      │
│  │ Drivers        │  │            └──────────────────────┘
│  │ Filesystem     │  │            ┌──────────────────────┐
│  │ Network stack  │  │            │  KERNEL SPACE        │
│  └────────────────┘  │            │  (IPC only)          │
└──────────────────────┘            └──────────────────────┘
  All runs in one space               Services in user space
  Fast (direct calls)                 Safer (isolated crashes)
  A driver bug can crash kernel        Slower (IPC overhead)
```

Linux is monolithic but uses **loadable kernel modules** (LKMs) to load drivers at runtime — getting some flexibility without full microkernel overhead.

---

### Linux Distributions

| Distro Family | When to use |
|---------------|-------------|
| **Debian/Ubuntu** | General servers, desktops, beginners — huge package repo, long LTS cycles |
| **RHEL/CentOS/AlmaLinux/Rocky** | Enterprise production — certified software, 10-year support, regulated industries |
| **Fedora** | Cutting-edge RHEL preview, developers wanting latest packages |
| **Arch Linux** | Rolling release, full control, learn-everything approach, `pacman` |
| **Alpine Linux** | Minimal containers (5MB base), security-focused, musl libc instead of glibc |
| **NixOS** | Reproducible builds, declarative config, atomic upgrades/rollbacks |
| **Gentoo** | Source-based, extreme customisation, performance tuning |

---

### The Unix Philosophy

> **"Write programs that do one thing and do it well. Write programs to work together. Write programs to handle text streams, because that is a universal interface."** — Doug McIlroy

```bash
# Composability in action — chain small tools into powerful pipelines
cat /var/log/auth.log \           # read the log
  | grep "Failed password" \      # filter failures
  | awk '{print $11}' \           # extract IP addresses
  | sort \                        # sort them
  | uniq -c \                     # count unique IPs
  | sort -rn \                    # sort by count descending
  | head -10                      # show top 10 attackers
```

---

### Everything is a File

Linux exposes almost everything through the filesystem:

```bash
/dev/sda          # block device (hard disk)
/dev/null         # discard all writes, return EOF on reads
/dev/zero         # infinite stream of null bytes
/dev/urandom      # cryptographically secure random bytes
/dev/stdin        # standard input (fd 0)
/dev/stdout       # standard output (fd 1)
/dev/stderr       # standard error (fd 2)
/proc/1/cmdline   # command line of PID 1 (init/systemd)
/proc/meminfo     # memory statistics as a text file
/sys/class/net/   # network interface parameters
/sys/block/sda/   # disk device attributes
```

---

### The Boot Process

```
Power On
   │
   ▼
BIOS/UEFI ────── POST (hardware self-test)
   │              Finds boot device (disk, USB, network)
   ▼
Bootloader ────── GRUB2 loaded from MBR/ESP
   │              Displays boot menu
   │              Loads kernel + initramfs into RAM
   ▼
Linux Kernel ──── Decompresses itself
   │              Detects hardware
   │              Mounts initramfs (tmpfs root)
   ▼
initramfs ─────── Minimal root filesystem in RAM
   │              Loads drivers needed to access real root
   │              Mounts real root filesystem
   ▼
init / systemd ── PID 1 starts
   │              Reads unit files / inittab
   │              Starts services in dependency order
   ▼
Login prompt / Display Manager
```

---

### init Systems

| Feature | SysV init | systemd | OpenRC |
|---------|-----------|---------|--------|
| Config format | Shell scripts | Unit files (INI) | Shell scripts |
| Parallelism | Limited | Full (dependency graph) | Limited |
| Service supervision | No | Yes | No |
| Logging | Syslog | journald (binary) | Syslog |
| Used by | Older distros | Ubuntu, Fedora, RHEL, Arch | Gentoo, Alpine |
| Complexity | Low | High | Medium |

---

### Kernel Space vs User Space

```
Ring 3: User Space  ──── Applications run here (low privilege)
                          Cannot directly access hardware
                          Must request services via syscall
                    ──── System Call Interface (boundary)
Ring 0: Kernel Space ─── Full hardware access
                          Manages all resources
                          One bug can crash the whole system

Syscall flow:
  app calls read()
    → C library wraps in syscall instruction
      → CPU switches to ring 0
        → kernel handles request
          → returns result
            → CPU switches back to ring 3
```

```bash
# Trace syscalls made by a command
strace ls /tmp 2>&1 | head -20    # show first 20 syscalls
strace -c ls /tmp                  # summarise syscall counts and times
```

---

### POSIX Standard

POSIX (Portable Operating System Interface) defines:
- Standard C library functions (`read`, `write`, `fork`, `exec`)
- Shell and utilities behaviour (`sh`, `ls`, `grep`, `awk`)
- File permissions model, process model, signals

```bash
# Check if your system is POSIX compliant for a command
man 1p ls        # POSIX spec for ls
man 3p read      # POSIX spec for read() C function

# Write portable shell scripts using POSIX sh (not bash-specific)
#!/bin/sh        # portable — runs on any POSIX system
#!/bin/bash      # bash-specific — not portable to Alpine (uses dash)
```

---

## 2. Filesystem Hierarchy & Navigation

### The Filesystem Hierarchy Standard (FHS)

```
/                   Root — the top of the entire filesystem tree
├── bin/            Essential user binaries (ls, cp, bash) — symlink to /usr/bin on modern systems
├── sbin/           Essential system binaries (fsck, mount) — symlink to /usr/sbin on modern
├── usr/            Unix System Resources — the bulk of installed software
│   ├── bin/        User commands (/usr/bin/python3, /usr/bin/git)
│   ├── sbin/       System admin commands
│   ├── lib/        Shared libraries for /usr/bin and /usr/sbin
│   ├── local/      Locally compiled software (not managed by package manager)
│   │   ├── bin/
│   │   ├── lib/
│   │   └── share/
│   └── share/      Architecture-independent data (man pages, icons, locale)
├── etc/            System-wide configuration files (text, editable)
├── var/            Variable data that changes at runtime
│   ├── log/        Log files
│   ├── lib/        Persistent application data
│   ├── spool/      Print/mail queues
│   ├── tmp/        Temp files (preserved between reboots, unlike /tmp)
│   └── cache/      Application cache data
├── tmp/            Temporary files (cleared on reboot, world-writable)
├── home/           User home directories (/home/alice, /home/bob)
├── root/           Home directory for the root user
├── opt/            Optional/third-party software (Oracle, Chrome)
├── srv/            Service data (web files /srv/www, FTP /srv/ftp)
├── dev/            Device files (block/char devices, virtual devices)
├── proc/           Virtual FS — kernel/process info as files (not on disk)
├── sys/            Virtual FS — kernel hardware/driver info
├── run/            Runtime data (PIDs, sockets) — tmpfs, cleared on reboot
├── lib/            Shared libraries and kernel modules — symlink to /usr/lib
├── lib64/          64-bit shared libraries
├── boot/           Kernel image, initramfs, GRUB files
├── mnt/            Temporary manual mount point
└── media/          Auto-mounted removable media (USB, DVD)
```

---

### Navigation

```bash
pwd                 # print working directory — where you are right now
cd /etc/nginx       # change to absolute path
cd nginx            # change to relative path (relative to current dir)
cd ..               # go up one directory
cd ../..            # go up two directories
cd ~                # go to your home directory
cd -                # go to the PREVIOUS directory (toggle back and forth)
cd                  # cd with no args = same as cd ~

# ls — list directory contents
ls                  # basic listing
ls -l               # long format: permissions, owner, size, date
ls -a               # show hidden files (starting with .)
ls -la              # long + hidden (most common combination)
ls -lh              # long + human-readable sizes (KB, MB, GB)
ls -lt              # long + sort by modification time (newest first)
ls -ltr             # long + time + reverse (oldest first — useful for logs)
ls -lS              # long + sort by size (largest first)
ls -R               # recursive listing of all subdirectories
ls -i               # show inode numbers
ls -1               # one file per line (for piping)
ls --color=always   # force colour output (even when piped)
ls -d */            # list only directories in current dir

# tree — visual directory tree
tree                # show full tree from current dir
tree -L 2           # limit depth to 2 levels
tree -a             # include hidden files
tree -d             # directories only
tree -h             # human-readable sizes
tree -I "*.pyc"     # ignore patterns
tree --du           # show disk usage per directory
```

---

### Directory Stack

```bash
pushd /etc          # push /etc onto stack AND cd to it; shows stack
pushd /var/log      # push /var/log; now stack is: /var/log /etc ~
pushd /tmp          # stack: /tmp /var/log /etc ~
dirs                # show directory stack
dirs -v             # show stack with indices
popd                # pop top entry, cd back to it (/var/log)
popd                # cd to /etc
pushd +2            # rotate stack, cd to index 2
```

---

### Hard Links vs Symbolic Links

```bash
# Inodes: every file = inode (metadata + data pointers) + directory entry (name)
# Hard link: another directory entry pointing to the SAME inode
# Symbolic link: a file whose content IS a path to another file

ls -i file.txt        # show inode number
stat file.txt         # show full inode info including link count

# Hard links
ln file.txt hardlink.txt       # create hard link — same inode, different name
# Both names are equally "real" — deleting one leaves data intact
# Hard links CANNOT cross filesystems or link to directories

# Symbolic links
ln -s /etc/nginx/nginx.conf nginx.conf    # create symlink
ln -s ../relative/path link               # relative symlink (more portable)
ls -la nginx.conf    # shows: nginx.conf -> /etc/nginx/nginx.conf
readlink nginx.conf  # print symlink target
readlink -f nginx.conf  # resolve ALL symlinks, print absolute real path

# Find broken symlinks
find /etc -xtype l   # find symlinks whose target doesn't exist
```

---

### Mounting

```bash
# Show all mounted filesystems
mount                           # verbose list
df -h                           # disk usage + mount points
lsblk                          # block device tree
lsblk -f                       # include filesystem type and UUID
findmnt                        # tree view of mounts
findmnt /home                  # show what's mounted at /home
blkid                          # show UUIDs and filesystem types for all devices
blkid /dev/sda1               # specific device

# Mount/unmount
mount /dev/sdb1 /mnt/data              # mount by device
mount UUID=abc-123 /mnt/data           # mount by UUID (preferred in fstab)
mount -t ext4 /dev/sdb1 /mnt/data      # specify filesystem type
mount -o ro /dev/sdb1 /mnt/data        # mount read-only
mount -o remount,rw /                  # remount root as read-write
umount /mnt/data                       # unmount
umount -l /mnt/data                    # lazy unmount (detach when idle)

# /etc/fstab — permanent mounts
# <device>  <mountpoint>  <fstype>  <options>  <dump>  <pass>
# dump: 0=skip, 1=include in dump backup
# pass: 0=skip fsck, 1=check first (root), 2=check after root
```

```ini
# /etc/fstab examples
UUID=abc-123-def   /           ext4    defaults,noatime   0 1
UUID=xyz-456       /home       xfs     defaults,noatime   0 2
UUID=aaa-789       /boot/efi   vfat    umask=0077         0 1
tmpfs              /tmp        tmpfs   defaults,size=2G   0 0
/dev/sdb1          /data       ext4    defaults,nofail    0 2
```

---

### df and du

```bash
df -h               # disk space: human-readable, all filesystems
df -hT              # include filesystem type
df -i               # inode usage (useful when disk "full" but df shows space)
df -h /home         # only the filesystem containing /home

du -sh /var/log     # total size of /var/log in human-readable
du -h --max-depth=1 /var  # sizes of each direct child of /var
du -sh * | sort -rh # size of everything in current dir, sorted largest first
du -ah /etc | sort -rh | head -20  # top 20 largest files in /etc
```

---

## 3. File Operations

### Basic File Commands

```bash
# cp — copy
cp file.txt dest/              # copy file into directory
cp file.txt newname.txt        # copy with new name
cp -r src/ dest/               # copy directory recursively
cp -p file.txt dest/           # preserve permissions, timestamps, ownership
cp -a src/ dest/               # archive mode: recursive + preserve all attributes
cp -u src dest                 # copy only if src is newer than dest
cp -v file.txt dest/           # verbose — show what's being copied
cp -i file.txt dest/           # interactive — prompt before overwriting

# mv — move / rename
mv file.txt newname.txt        # rename in place
mv file.txt /tmp/              # move to directory
mv -i src dest                 # prompt before overwriting
mv -n src dest                 # never overwrite existing file
mv -v src dest                 # verbose

# rm — remove
# ⚠️ Warning: rm is permanent — no Trash. Always verify before running.
rm file.txt                    # delete file
rm -r directory/               # delete directory recursively
rm -f file.txt                 # force: no error if not exists, no prompt
rm -rf directory/              # force + recursive — VERY DANGEROUS
rm -i file.txt                 # interactive — confirm each deletion
rm -v file.txt                 # verbose

# Safer rm -rf pattern:
ls -la /path/to/delete/        # ALWAYS verify the path first
rm -rf /path/to/delete/        # then delete

# mkdir / rmdir / touch
mkdir mydir                    # create directory
mkdir -p a/b/c/d               # create full path including parents
mkdir -m 750 secure/           # create with specific permissions
rmdir emptydir/                # remove EMPTY directory only
touch file.txt                 # create empty file or update timestamps
touch -t 202401151030 file.txt # set specific timestamp
```

---

### find — Complete Guide

```bash
# Syntax: find [path] [expression]
find /etc -name "nginx.conf"          # find by exact name
find /etc -name "*.conf"              # find by wildcard
find /etc -iname "*.CONF"             # case-insensitive name search
find / -type f -name "*.sh"           # type f=file, d=dir, l=symlink, b=block, c=char
find /var -type f -size +100M         # files larger than 100MB
find /var -type f -size +10k -size -1M # between 10KB and 1MB
find /tmp -type f -mtime +7           # modified more than 7 days ago
find /tmp -type f -mtime -1           # modified in the last 24 hours
find /tmp -type f -newer /etc/passwd  # modified more recently than /etc/passwd
find /etc -maxdepth 1 -name "*.conf"  # only in /etc, not subdirs
find /home -perm 777                  # files with exact permissions 777
find /home -perm -u+x                 # files with user execute bit set
find / -perm /4000                    # find all setuid files (security audit)
find . -empty                         # find empty files and directories
find . -user alice                    # owned by user alice
find . -group developers              # owned by group developers

# -exec: run a command on each found file
find /tmp -mtime +30 -exec rm {} \;   # delete files older than 30 days
find . -name "*.py" -exec chmod 644 {} \;  # fix permissions
find . -type f -exec grep -l "TODO" {} \;  # find files containing "TODO"

# Faster: -exec with + batches all results into one command call
find . -name "*.log" -exec gzip {} +  # compress all logs in one gzip call

# -delete: even faster than -exec rm for deletion
find /tmp -mtime +30 -delete          # delete files older than 30 days

# Combining with logical operators
find . -name "*.log" -and -size +10M      # AND (implicit between expressions)
find . -name "*.tmp" -or -name "*.bak"   # OR
find . -not -name "*.py"                  # NOT
find . \( -name "*.jpg" -o -name "*.png" \) -size +1M  # grouped OR
```

> 💡 **Tip:** Use `find . -print0 | xargs -0 command` for filenames with spaces — `-print0` uses null byte as separator, `-0` tells xargs to split on null.

---

### stat, file, locate

```bash
stat file.txt              # full metadata: size, inode, permissions, all timestamps
# Output includes:
#   File: file.txt
#   Size: 1234       Inode: 567890    Links: 1
#   Access: (0644/-rw-r--r--)  Uid: (1000/alice)  Gid: (1000/alice)
#   Access: 2024-01-15 10:30:00   ← atime: last read
#   Modify: 2024-01-14 09:00:00   ← mtime: last content change
#   Change: 2024-01-14 09:00:00   ← ctime: last metadata change (permissions, owner)

file /bin/ls               # determine file type from magic bytes
file *.txt                 # check multiple files
file -i document.bin       # show MIME type

# locate — fast filename search using a database
locate nginx.conf          # find files by name (nearly instant)
locate -i "*.conf"         # case-insensitive
locate -c nginx            # count matches
updatedb                   # update the locate database (usually runs daily via cron)
```

---

### rsync — Sync Files

```bash
# rsync is the gold standard for syncing files locally and over SSH
rsync -av src/ dest/                    # archive (recursive + preserve attrs) + verbose
rsync -avz src/ user@host:/dest/        # sync to remote over SSH, with compression
rsync -avz --delete src/ dest/         # delete files in dest not in src (mirror)
rsync -avzn src/ dest/                 # dry run — shows what WOULD happen (safe!)
rsync -avz --exclude="*.log" src/ dest/ # exclude pattern
rsync -avz --exclude-from=exclude.txt src/ dest/  # exclusions from file
rsync -avz --progress src/ dest/       # show per-file progress
rsync -avz --bwlimit=5000 src/ dest/   # limit bandwidth to ~5MB/s
rsync -avz -e "ssh -p 2222" src/ user@host:/dest/  # SSH on custom port
rsync --checksum src/ dest/            # compare by checksum not timestamp/size

# ⚠️ Note the trailing slash difference:
rsync -av src/  dest/    # copies CONTENTS of src into dest
rsync -av src   dest/    # copies src DIRECTORY itself into dest → dest/src/
```

---

### tar — Archives

```bash
# Create archives
tar -czf archive.tar.gz  directory/    # create gzip-compressed archive
tar -cjf archive.tar.bz2 directory/   # create bzip2-compressed archive
tar -cJf archive.tar.xz  directory/   # create xz-compressed archive
tar -cf  archive.tar     directory/   # create uncompressed archive
tar -czf backup.tar.gz --exclude="*.log" /var/www/  # exclude files

# Extract archives
tar -xzf archive.tar.gz               # extract gzip archive
tar -xjf archive.tar.bz2              # extract bzip2 archive
tar -xJf archive.tar.xz               # extract xz archive
tar -xf  archive.tar                  # extract (auto-detect format)
tar -xzf archive.tar.gz -C /tmp/      # extract to specific directory
tar -xzf archive.tar.gz file.txt      # extract only one file

# List contents without extracting
tar -tzf archive.tar.gz               # list contents of gzip archive
tar -tf  archive.tar                  # list uncompressed

# Flag reference:
# -c create   -x extract   -t list
# -z gzip     -j bzip2     -J xz
# -f file     -v verbose   -C destination dir

# Other compression tools
gzip file.txt            # compress → file.txt.gz (replaces original)
gzip -k file.txt         # keep original
gzip -d file.txt.gz      # decompress
gunzip file.txt.gz        # same as gzip -d

zstd file.txt            # modern fast compression → file.txt.zst
zstd -d file.txt.zst     # decompress
zstd -19 file.txt        # max compression level (1-19)
```

---

### dd — Low-Level Copy

> ⚠️ **Warning:** `dd` writes directly to devices — a wrong `of=` can wipe your disk instantly. Triple-check your paths.

```bash
# ALWAYS verify paths before running dd
lsblk                               # verify which device is which

# Disk imaging
dd if=/dev/sda of=/backup/disk.img bs=4M status=progress  # image entire disk
dd if=/backup/disk.img of=/dev/sdb bs=4M status=progress  # restore image to disk

# Create a file of specific size
dd if=/dev/zero of=testfile.bin bs=1M count=100  # create 100MB file of zeros
dd if=/dev/urandom of=testfile.bin bs=1M count=1 # 1MB of random data

# Wipe a disk
dd if=/dev/zero of=/dev/sdb bs=4M status=progress  # overwrite with zeros ⚠️

# Create bootable USB (safer alternatives: cp or cat)
dd if=ubuntu.iso of=/dev/sdb bs=4M conv=fsync status=progress  # write ISO to USB

# Options:
# if=  input file   of=  output file
# bs=  block size   count=  number of blocks
# status=progress   show transfer rate and ETA
# conv=fsync        flush write buffer before exit (safer)
```

---

### Globbing and Wildcards

```bash
ls *.txt            # * matches any string (including empty)
ls file?.txt        # ? matches exactly one character
ls file[123].txt    # [abc] matches one character from set
ls file[1-9].txt    # [1-9] matches one character in range
ls file[!abc].txt   # [!abc] matches one char NOT in set
ls {*.txt,*.md}     # brace expansion: matches *.txt OR *.md
ls {file1,file2,file3}.conf  # expand specific names

# ** (globstar) — matches any path depth
shopt -s globstar    # enable globstar in bash
ls **/*.py           # all .py files in current dir and all subdirs

# Brace expansion (not globbing — always expands)
echo {a,b,c}         # a b c
echo {1..5}          # 1 2 3 4 5
echo {01..10}        # 01 02 03 04 05 06 07 08 09 10
mkdir -p project/{src,tests,docs,bin}  # create multiple dirs at once
cp file.txt{,.bak}   # copy file.txt to file.txt.bak (neat trick!)
```

---

## 4. Text Processing

### Viewing Files

```bash
cat file.txt              # print entire file to stdout
cat -n file.txt           # show line numbers
cat -A file.txt           # show non-printing chars ($ for newlines, ^I for tabs)
tac file.txt              # print file in reverse order (last line first)
head file.txt             # first 10 lines
head -n 20 file.txt       # first 20 lines
head -c 1024 file.txt     # first 1024 bytes
tail file.txt             # last 10 lines
tail -n 20 file.txt       # last 20 lines
tail -f /var/log/syslog   # follow: print new lines as they're added (live log watching)
tail -F /var/log/nginx/access.log  # follow even if file is rotated (F = follow name)
less file.txt             # page through file (q to quit, / to search, n/N next/prev)
more file.txt             # simpler pager (less is better)

# less key bindings:
# /pattern  search forward   ?pattern  search backward
# n         next match       N         previous match
# g         go to start      G         go to end
# q         quit             Space     page down
# F         follow mode (like tail -f)
```

---

### grep — Complete Guide

```bash
# Basic usage
grep "pattern" file.txt          # find lines containing pattern
grep "error" /var/log/syslog     # search log file

# Key flags
grep -i "error" file.txt         # case-insensitive search
grep -r "TODO" /src/             # recursive: search all files in directory
grep -rl "TODO" /src/            # recursive, list only FILENAMES with matches
grep -n "error" file.txt         # show line numbers
grep -v "debug" file.txt         # invert: show lines NOT matching
grep -c "error" file.txt         # count matching lines
grep -o "192\.168\.[0-9]*\.[0-9]*" file.txt  # print ONLY the matched text

# Context lines
grep -A 3 "error" file.txt       # show 3 lines After match
grep -B 3 "error" file.txt       # show 3 lines Before match
grep -C 3 "error" file.txt       # show 3 lines Context (before AND after)

# Regex modes
grep -E "err(or|no)" file.txt    # extended regex (ERE): |, +, ?, (), {}
grep -P "\d{3}-\d{4}" file.txt   # PCRE: Perl-compatible (lookahead, etc.)
grep -F "exact.string" file.txt  # fixed string: no regex interpretation (fast)

# Multiple patterns
grep -e "error" -e "warn" file.txt     # match either pattern
grep -f patterns.txt file.txt          # read patterns from file

# File filtering
grep -r --include="*.py" "import" /src/    # only search .py files
grep -r --exclude="*.log" "error" /var/    # skip .log files
grep -r --exclude-dir=".git" "TODO" .     # skip .git directory

# Useful combinations
grep -rn "TODO\|FIXME\|HACK" /src/        # find code annotations
grep -v "^#\|^$" /etc/nginx/nginx.conf    # strip comments and blank lines
grep -c "" file.txt                        # count total lines (matches empty string)
ls -la | grep "^d"                         # list only directories
```

---

### sed — Stream Editor

```bash
# Substitution: s/pattern/replacement/flags
sed 's/old/new/' file.txt          # replace FIRST occurrence per line
sed 's/old/new/g' file.txt         # replace ALL occurrences per line (global)
sed 's/old/new/gi' file.txt        # global + case-insensitive
sed 's/old/new/2' file.txt         # replace 2nd occurrence only
sed 's|/old/path|/new/path|g' file.txt  # use | as delimiter (avoids escaping /)

# In-place editing
sed -i 's/old/new/g' file.txt          # edit file in place ⚠️
sed -i.bak 's/old/new/g' file.txt      # edit in place, keep backup as file.txt.bak

# Address ranges — specify which lines to operate on
sed '5s/old/new/' file.txt         # only line 5
sed '2,8s/old/new/g' file.txt      # lines 2 through 8
sed '/pattern/s/old/new/g' file.txt # only lines matching /pattern/
sed '/start/,/end/s/old/new/g' file.txt  # lines between /start/ and /end/

# Deletion
sed '/^#/d' file.txt               # delete lines starting with #
sed '/^$/d' file.txt               # delete empty lines
sed '1d' file.txt                  # delete line 1
sed '1,5d' file.txt                # delete lines 1 through 5

# Print (with -n to suppress default output)
sed -n '5,10p' file.txt            # print only lines 5-10
sed -n '/error/p' file.txt         # print only lines matching /error/

# Insert / Append / Change
sed '3i\New line before line 3' file.txt   # insert before line 3
sed '3a\New line after line 3' file.txt    # append after line 3
sed '3c\Replace line 3 entirely' file.txt  # change (replace) line 3

# Multiple expressions
sed -e 's/foo/bar/g' -e 's/baz/qux/g' file.txt   # apply multiple seds
sed 's/foo/bar/g; s/baz/qux/g' file.txt           # semicolon separator

# Hold buffer (advanced — two-line patterns)
sed -n '/start/{h; d}; /end/{H; g; p}' file.txt  # collect block between start/end
```

---

### awk — Complete Guide

```bash
# Syntax: awk 'pattern { action }' file
# awk reads line by line, splits into fields ($1, $2, ..., $NF = last field)

# Basic field access
awk '{print $1}' file.txt           # print first field of each line
awk '{print $NF}' file.txt          # print last field
awk '{print $1, $3}' file.txt       # print fields 1 and 3 (OFS-separated)
awk '{print $0}' file.txt           # print entire line ($0 = whole record)

# Field separator
awk -F: '{print $1}' /etc/passwd    # use : as field separator
awk -F',' '{print $2}' data.csv     # use , as separator (CSV)
awk 'BEGIN{FS=":"} {print $1}' /etc/passwd  # set FS in BEGIN

# Pattern filtering
awk '/error/ {print}' file.txt      # print lines matching /error/
awk '!/debug/ {print}' file.txt     # print lines NOT matching /debug/
awk '$3 > 100 {print}' data.txt     # print if 3rd field > 100

# BEGIN and END blocks
awk 'BEGIN{print "Header"} {print} END{print "Footer"}' file.txt
awk 'BEGIN{sum=0} {sum += $2} END{print "Total:", sum}' data.txt  # sum column 2

# Built-in variables
# NR   current record number (line number)
# NF   number of fields in current record
# FS   input field separator (default: whitespace)
# OFS  output field separator (default: space)
# RS   input record separator (default: newline)
# ORS  output record separator
awk 'NR==5' file.txt                # print only line 5
awk 'NR>=5 && NR<=10' file.txt      # print lines 5-10
awk '{print NR, $0}' file.txt       # add line numbers
awk 'END{print NR}' file.txt        # count lines (like wc -l)

# Arithmetic
awk '{sum+=$1; count++} END{print "avg:", sum/count}' nums.txt
awk '{if($1>max) max=$1} END{print "max:", max}' nums.txt

# Arrays (associative)
awk '{count[$1]++} END{for(k in count) print k, count[k]}' file.txt  # frequency count
awk -F: '{users[$3]=$1} END{for(uid in users) print uid, users[uid]}' /etc/passwd

# String functions
awk '{print length($0)}' file.txt           # line length
awk '{print toupper($1)}' file.txt          # uppercase
awk '{gsub(/old/, "new"); print}' file.txt  # global substitution
awk '{split($0, a, ":"); print a[1]}' file.txt  # split field into array

# printf formatting
awk '{printf "%-20s %5d\n", $1, $2}' file.txt  # aligned output
awk '{printf "%08.2f\n", $1}' nums.txt          # zero-padded float

# Multiline/piped
ps aux | awk 'NR>1 {print $1, $2, $11}'    # process list: user, PID, command
df -h | awk 'NR>1 {print $6, $5}'          # disk: mount point and usage %
awk -F: '$3 >= 1000 {print $1}' /etc/passwd  # list all regular users (UID >= 1000)
```

---

### sort, uniq, wc, cut

```bash
# sort
sort file.txt               # alphabetical sort
sort -n file.txt            # numeric sort
sort -rn file.txt           # reverse numeric sort (largest first)
sort -u file.txt            # sort and remove duplicates
sort -k2 file.txt           # sort by field 2 (whitespace-delimited)
sort -k2,2n file.txt        # sort by field 2 numerically
sort -t: -k3 -n /etc/passwd # use : as delimiter, sort by field 3 (UID)
sort -h file.txt            # human sort: 1K 2M 10G sorted correctly
sort -V versions.txt        # version sort: 1.9 < 1.10 < 2.0

# uniq (input must be sorted!)
sort file.txt | uniq        # remove duplicate lines
sort file.txt | uniq -c    # count occurrences of each unique line
sort file.txt | uniq -d    # show only DUPLICATE lines
sort file.txt | uniq -u    # show only UNIQUE (non-duplicate) lines

# wc
wc file.txt                 # lines, words, characters
wc -l file.txt              # count lines only
wc -w file.txt              # count words only
wc -c file.txt              # count bytes
wc -m file.txt              # count characters (differs from bytes for UTF-8)
wc -l *.py | sort -rn       # count lines in all .py files, sorted

# cut
cut -d: -f1 /etc/passwd     # field 1 using : delimiter (usernames)
cut -d: -f1,3 /etc/passwd   # fields 1 and 3
cut -c1-10 file.txt         # characters 1-10 from each line
cut -c-5 file.txt           # first 5 characters

# tr — translate or delete characters
tr 'a-z' 'A-Z' < file.txt    # lowercase to uppercase
tr -d '\r' < windows.txt     # delete carriage returns (fix Windows line endings)
tr -s ' ' < file.txt         # squeeze multiple spaces into one
tr -d '[:digit:]' < file.txt # delete all digits
echo "hello world" | tr ' ' '\n'  # replace spaces with newlines
```

---

### diff, jq, Column Formatting

```bash
# diff — compare files
diff file1.txt file2.txt        # show differences
diff -u file1.txt file2.txt     # unified format (used in patches)
diff -y file1.txt file2.txt     # side-by-side
diff -r dir1/ dir2/             # compare directories recursively
diff --color file1.txt file2.txt # coloured output

# patch — apply diff to file
diff -u original.txt modified.txt > changes.patch   # create patch
patch original.txt < changes.patch                   # apply patch

# jq — JSON processor (must-have tool)
cat data.json | jq '.'                          # pretty-print JSON
jq '.name' data.json                            # extract .name field
jq '.users[0]' data.json                        # first element of users array
jq '.users[].name' data.json                    # name field of every user
jq 'select(.age > 18)' users.json              # filter by condition
jq '.[] | {name: .name, age: .age}' users.json # reshape objects
jq -r '.name' data.json                         # raw output (no quotes)
jq 'length' data.json                           # count elements
jq 'keys' data.json                             # list keys of object
jq '.users | map(select(.active == true))' d.json  # filter array
curl -s https://api.github.com/users/torvalds | jq '{name,company,followers}'

# column — tabular formatting
column -t file.txt              # auto-align columns (whitespace-separated)
column -t -s: /etc/passwd       # use : as column separator
mount | column -t               # nicely format mount output

# nl — line numbering
nl file.txt                     # number lines (skips empty lines)
nl -ba file.txt                 # number ALL lines including empty
```

---

## 5. Shell & Scripting

### Shell Types

| Shell | Use case |
|-------|----------|
| `sh` (POSIX) | Portable scripts — runs everywhere |
| `bash` | Default on most Linux distros — rich scripting, interactive |
| `zsh` | Power users — better completion, plugins (Oh My Zsh) |
| `fish` | Beginner-friendly — syntax highlighting, no POSIX |
| `dash` | Fast POSIX shell — Ubuntu uses as `/bin/sh` for scripts |

```bash
echo $SHELL           # current shell
cat /etc/shells       # available shells
chsh -s /bin/zsh      # change login shell (takes effect on next login)
bash --version        # bash version
```

---

### Shell Startup Files

```
Login shell (ssh, su -, console login):
  /etc/profile → /etc/profile.d/*.sh → ~/.bash_profile → ~/.bashrc

Interactive non-login shell (terminal emulator, new bash):
  /etc/bash.bashrc → ~/.bashrc

Non-interactive shell (scripts):
  Only $BASH_ENV if set

Zsh:
  /etc/zshenv → ~/.zshenv → /etc/zprofile → ~/.zprofile →
  /etc/zshrc → ~/.zshrc → /etc/zlogin → ~/.zlogin
```

```bash
# Source a file (execute in current shell, not a subshell)
source ~/.bashrc       # reload bashrc
. ~/.bashrc            # same thing (. is POSIX alias for source)

# Common ~/.bashrc additions
export PATH="$HOME/.local/bin:$PATH"    # add to PATH
alias ll='ls -la'                        # create alias
alias grep='grep --color=auto'           # always colour grep
export EDITOR=vim                        # set default editor
export HISTSIZE=10000                    # larger history
export HISTFILESIZE=20000
export HISTCONTROL=ignoredups:erasedups  # no duplicate history entries
```

---

### Variables

```bash
# Assignment — NO spaces around =
name="Alice"              # string
count=42                  # integer (still stored as string in bash)
readonly PI=3.14159       # constant — cannot be changed
unset name                # delete variable

# Expansion
echo $name                # basic expansion
echo ${name}              # explicit braces (required before text: ${name}s)
echo "${name} Smith"      # quote to preserve spaces

# Parameter expansion
${var:-default}     # use default if var is unset or empty
${var:=default}     # set var to default if unset, then expand
${var:+value}       # use value if var IS set (else empty)
${var:?error msg}   # expand or print error and exit if unset

# String operations
${#var}             # length of string
${var%suffix}       # remove shortest matching suffix
${var%%suffix}      # remove longest matching suffix
${var#prefix}       # remove shortest matching prefix
${var##prefix}      # remove longest matching prefix
${var/old/new}      # replace first occurrence
${var//old/new}     # replace ALL occurrences
${var^}             # uppercase first character
${var^^}            # uppercase all characters
${var,}             # lowercase first character
${var,,}            # lowercase all characters

# Examples
filename="archive.tar.gz"
echo ${filename##*.}      # gz (extension)
echo ${filename%.*}       # archive.tar (remove last extension)
echo ${filename%%.*}      # archive (remove all extensions)

path="/home/alice/docs"
echo ${path##*/}          # docs (basename)
echo ${path%/*}           # /home/alice (dirname)
```

---

### Special Variables

```bash
$0          # script name (or shell name if interactive)
$1 .. $9    # positional parameters (arguments)
${10}       # tenth argument (braces needed for 2+ digits)
$@          # all positional parameters as separate words → "$1" "$2" "$3"
$*          # all positional parameters as one word → "$1 $2 $3"
$#          # number of positional parameters
$?          # exit status of last command (0=success, non-zero=error)
$$          # PID of current shell process
$!          # PID of last background command
$-          # current shell option flags (himBH...)
$_          # last argument of previous command

# $@ vs $* — critical difference in double quotes:
for arg in "$@"; do  # ✅ each arg as separate word — handles spaces in args
    echo "$arg"
done
for arg in "$*"; do  # ⚠️ all args as ONE word — breaks with spaces
    echo "$arg"
done
```

---

### Arithmetic

```bash
# $(( )) — preferred bash arithmetic (integers only)
echo $((5 + 3))         # 8
echo $((10 / 3))        # 3 (integer division — truncates)
echo $((10 % 3))        # 1 (modulo)
echo $((2 ** 10))       # 1024 (exponentiation)
x=5; ((x += 3)); echo $x  # 8 — arithmetic in place
(( x > 5 )) && echo "big"  # arithmetic conditional

# let — alternative syntax
let "x = 5 * 3"
let x++

# bc — arbitrary precision (handles floats!)
echo "scale=2; 10/3" | bc      # 3.33
echo "scale=4; sqrt(2)" | bc   # 1.4142
echo "obase=16; 255" | bc      # FF (decimal to hex)
```

---

### Arrays

```bash
# Indexed arrays
fruits=("apple" "banana" "cherry")   # declare and initialise
fruits[3]="date"                      # add element at index 3
echo ${fruits[0]}                     # apple — access by index
echo ${fruits[-1]}                    # last element (bash 4.3+)
echo ${fruits[@]}                     # all elements
echo ${#fruits[@]}                    # number of elements
echo ${!fruits[@]}                    # all indices

# Iterate array
for fruit in "${fruits[@]}"; do
    echo "$fruit"
done

# Array operations
fruits+=("elderberry")                # append element
unset 'fruits[1]'                     # delete element at index 1

# Associative arrays (hash maps)
declare -A person
person[name]="Alice"
person[age]=30
person[city]="London"
echo ${person[name]}                  # Alice
echo ${!person[@]}                    # all keys
echo ${person[@]}                     # all values
for key in "${!person[@]}"; do
    echo "$key: ${person[$key]}"
done
```

---

### Control Flow

```bash
# if / elif / else
if [ "$1" = "hello" ]; then       # [ ] = test command
    echo "Hello!"
elif [[ "$1" =~ ^[0-9]+$ ]]; then # [[ ]] = bash conditional (regex with =~)
    echo "A number"
else
    echo "Something else"
fi

# case / esac
case "$1" in
    start)   systemctl start nginx ;;
    stop)    systemctl stop nginx ;;
    restart) systemctl restart nginx ;;
    *)       echo "Usage: $0 {start|stop|restart}" ;;
esac

# for — foreach style
for file in *.txt; do
    echo "Processing: $file"
done

# for — C style
for ((i=0; i<10; i++)); do
    echo $i
done

# for — with range
for i in {1..10}; do echo $i; done
for i in $(seq 1 2 20); do echo $i; done  # 1 3 5 7 ... 19

# while
while IFS= read -r line; do      # read file line by line (IFS= preserves leading spaces)
    echo "$line"
done < file.txt

count=0
while [[ $count -lt 5 ]]; do
    echo $count
    ((count++))
done

# until (opposite of while)
until ping -c1 server &>/dev/null; do
    echo "Waiting for server..."
    sleep 5
done
```

---

### Functions

```bash
# Function definition (two syntaxes — identical)
greet() {
    local greeting="Hello"         # local: scoped to function only
    echo "$greeting, $1!"          # $1 is first arg to function
}

function goodbye {
    echo "Goodbye, $1!"
}

# Calling
greet "Alice"      # Hello, Alice!
goodbye "Bob"      # Goodbye, Bob!

# Return values
# ⚠️ return only returns integer exit codes (0-255)
# Use command substitution to return strings

get_hostname() {
    hostname -f          # output goes to stdout
}

result=$(get_hostname)   # capture stdout into variable
echo "Hostname: $result"

# Pass arrays to functions
process_array() {
    local -n arr=$1     # nameref — reference to caller's array (bash 4.3+)
    for item in "${arr[@]}"; do
        echo "$item"
    done
}
my_list=("a" "b" "c")
process_array my_list
```

---

### test, [, [[

```bash
# test, [, [[ — all evaluate conditions
# [ is POSIX (available everywhere)
# [[ is bash-specific (safer, more features)

# String tests
[ -z "$str" ]           # true if string is empty
[ -n "$str" ]           # true if string is non-empty
[ "$a" = "$b" ]         # string equality (use = not == in [ ])
[[ "$a" == "$b" ]]      # string equality in [[
[[ "$str" =~ ^[0-9]+$ ]] # regex match (only in [[)

# Numeric tests (use inside [ ] or [[ ]])
[ "$a" -eq "$b" ]       # equal
[ "$a" -ne "$b" ]       # not equal
[ "$a" -lt "$b" ]       # less than
[ "$a" -le "$b" ]       # less than or equal
[ "$a" -gt "$b" ]       # greater than
[ "$a" -ge "$b" ]       # greater than or equal

# File tests
[ -e "$f" ]             # file or directory exists
[ -f "$f" ]             # exists and is a regular file
[ -d "$f" ]             # exists and is a directory
[ -L "$f" ]             # exists and is a symlink
[ -r "$f" ]             # exists and is readable
[ -w "$f" ]             # exists and is writable
[ -x "$f" ]             # exists and is executable
[ -s "$f" ]             # exists and has size > 0
[ "$f1" -nt "$f2" ]     # f1 is newer than f2
[ "$f1" -ot "$f2" ]     # f1 is older than f2

# Combining conditions
[ -f "$f" ] && [ -r "$f" ]    # AND with [ ]
[[ -f "$f" && -r "$f" ]]      # AND with [[
[[ -f "$f" || -d "$f" ]]      # OR with [[
[[ ! -e "$f" ]]               # NOT
```

---

### I/O Redirection

```bash
command > file.txt        # redirect stdout to file (overwrite)
command >> file.txt       # redirect stdout to file (append)
command < file.txt        # redirect file to stdin
command 2> errors.txt     # redirect stderr to file
command 2>&1              # redirect stderr to wherever stdout is going
command > out.txt 2>&1    # both stdout and stderr to file
command &> out.txt        # same thing (bash shorthand)
command > /dev/null 2>&1  # discard all output (silence command)
command 2>/dev/null       # discard only errors

# Heredoc — multi-line stdin
cat << 'EOF'              # single-quoted: NO variable expansion inside
This is literal $var text
EOF

cat << EOF                # unquoted: variables ARE expanded
Hello, $USER
Today is $(date)
EOF

# Herestring — single-line stdin
grep "pattern" <<< "search in this string"

# tee — write to file AND stdout simultaneously
command | tee output.txt             # tee to file, also print to screen
command | tee -a output.txt          # append instead of overwrite
command | tee file1 file2            # write to multiple files
command | tee output.txt | grep err  # tee and continue pipeline
```

---

### set Options for Safer Scripts

```bash
#!/bin/bash
set -e              # exit immediately if any command fails
set -u              # treat unset variables as errors
set -x              # print each command before executing (debugging)
set -o pipefail     # pipeline fails if ANY command in it fails
                    # (without this, only the last command's exit code matters)

# Combined (best practice for scripts)
set -euo pipefail

# Example of why pipefail matters:
set -e
false | true        # without pipefail: exit code is 0 (true succeeded)
                    # with pipefail:    exit code is 1 (false failed)

# trap — cleanup on exit or error
cleanup() {
    rm -f /tmp/tempfile      # always clean up temp files
    echo "Cleanup complete"
}
trap cleanup EXIT            # run cleanup() when script exits (any reason)
trap 'echo "Error on line $LINENO"' ERR  # run on any error
trap 'echo "Interrupted"; exit 1' INT   # run on Ctrl+C

# Full robust script template
#!/bin/bash
set -euo pipefail
IFS=$'\n\t'         # safer word splitting (only newlines and tabs, not spaces)

readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

cleanup() {
    # cleanup code here
    true
}
trap cleanup EXIT
```

---

### getopts — Option Parsing

```bash
#!/bin/bash

usage() {
    echo "Usage: $0 [-v] [-o output] [-n count] file..."
    exit 1
}

verbose=false
output=""
count=1

while getopts ":vo:n:" opt; do    # : at start = silent error mode
    case $opt in
        v) verbose=true ;;         # flag: no argument
        o) output="$OPTARG" ;;     # option with required argument
        n) count="$OPTARG" ;;      # option with required argument
        :) echo "Option -$OPTARG requires an argument."; usage ;;
        \?) echo "Invalid option: -$OPTARG"; usage ;;
    esac
done

shift $((OPTIND - 1))              # remove parsed options, leave positional args
remaining_args=("$@")              # remaining non-option arguments
```

---

## 6. Permissions & Ownership

### The Permission Model

```
Permission string format:
   drwxr-xr-x
   │└─┬─┘└─┬─┘└─┬─┘
   │  │    │    └─ Other:  r-x = 5
   │  │    └─────── Group:  r-x = 5
   │  └──────────── User:  rwx = 7
   └─────────────── Type: d=dir, -=file, l=link, b=block, c=char, p=pipe

Each permission set: r=read(4) w=write(2) x=execute(1)

For FILES:   r=read content,   w=modify content, x=run as program
For DIRS:    r=list contents,  w=create/delete files inside,  x=enter (cd into)

Octal quick reference:
  7 = rwx    6 = rw-    5 = r-x    4 = r--
  3 = -wx    2 = -w-    1 = --x    0 = ---

Common permission patterns:
  755 = rwxr-xr-x   directories, executables (everyone can enter/run)
  644 = rw-r--r--   regular files (owner writes, others read)
  600 = rw-------   private files (SSH keys, passwords)
  700 = rwx------   private directories
  777 = rwxrwxrwx   world-writable (AVOID — security risk)
  664 = rw-rw-r--   shared group files
```

---

### chmod, chown, chgrp

```bash
# chmod — change permissions
chmod 755 script.sh          # octal mode
chmod u+x script.sh          # symbolic: add execute for user
chmod go-w file.txt          # symbolic: remove write for group and other
chmod u=rw,go=r file.txt     # symbolic: set exact permissions
chmod -R 755 /var/www/html/  # recursive — apply to all files and dirs
chmod a+x script.sh          # a = all (user+group+other)

# chown — change owner (and optionally group)
chown alice file.txt          # change owner to alice
chown alice:devs file.txt     # change owner AND group
chown :devs file.txt          # change only group (same as chgrp)
chown -R www-data:www-data /var/www/  # recursive ownership change
chown --reference=src.txt dest.txt    # copy ownership from another file

# chgrp — change group only
chgrp developers project/     # change group ownership
chgrp -R developers project/  # recursive
```

---

### Special Permission Bits

```bash
# Setuid (SUID) — u+s — 4xxx
# On executable: runs as file OWNER, not the user who executes it
# Security risk: never set on shell scripts
ls -l /usr/bin/passwd         # -rwsr-xr-x — 's' in user execute = setuid
chmod u+s /usr/bin/special    # set SUID
chmod 4755 /usr/bin/special   # set SUID in octal
find / -perm /4000 -type f    # audit: find all setuid files

# Setgid (SGID) — g+s — 2xxx
# On executable: runs as file GROUP
# On directory: new files inherit directory's GROUP (useful for shared dirs)
chmod g+s /shared/directory/  # set SGID on directory
chmod 2775 /shared/directory/ # SGID + rwxrwxr-x

# Sticky bit — o+t — 1xxx
# On directory: only file OWNER can delete their files (even if others have write)
# Classic use: /tmp
ls -ld /tmp                   # drwxrwxrwt — 't' in other execute = sticky
chmod +t /shared/uploads/     # set sticky bit
chmod 1777 /tmp               # sticky + world-writable
```

---

### umask

```bash
# umask defines which permissions are REMOVED from new files/dirs
# Default file permissions: 666 (rw-rw-rw-)
# Default dir permissions:  777 (rwxrwxrwx)
# umask 022: removes w from group and other
#   Files: 666 - 022 = 644 (rw-r--r--)
#   Dirs:  777 - 022 = 755 (rwxr-xr-x)

umask                  # show current umask (e.g., 0022)
umask 022              # set umask for current session
umask 027              # private: group read-only, others nothing

# Calculation:
# new_permissions = default_permissions AND NOT umask
# umask 027 for files: 666 & ~027 = 640 (rw-r-----)
```

---

### ACLs

```bash
# Standard Unix permissions are per-user-or-group
# ACLs allow fine-grained control for multiple users/groups

# View ACL
getfacl /shared/file.txt     # show ACL entries

# Set ACL
setfacl -m u:alice:rwx /shared/file.txt    # give alice rwx
setfacl -m u:bob:r-- /shared/file.txt      # give bob read-only
setfacl -m g:dev:rw- /shared/file.txt      # give dev group rw
setfacl -m o::--- /shared/file.txt         # restrict others

# Default ACL (inherited by new files in directory)
setfacl -d -m u:alice:rw /shared/dir/     # new files get alice:rw

# Remove ACL entries
setfacl -x u:alice /shared/file.txt       # remove alice's ACL entry
setfacl -b /shared/file.txt               # remove ALL ACL entries

# The mask limits effective permissions for group + named users
setfacl -m mask::r-- /shared/file.txt     # cap all non-owner permissions to read
```

---

### sudo and Capabilities

```bash
# sudo — execute as another user (usually root)
sudo command                    # run as root
sudo -u postgres psql           # run as specific user
sudo -l                         # list what YOU can sudo
sudo -l -U alice                # list what alice can sudo
sudo -i                         # interactive root login shell
sudo -s                         # root shell without full login env
sudo !!                         # re-run last command with sudo

# visudo — ALWAYS use this to edit /etc/sudoers (validates syntax)
visudo

# /etc/sudoers syntax
alice ALL=(ALL:ALL) ALL              # alice can run anything as anyone
bob ALL=(ALL) NOPASSWD: /bin/systemctl restart nginx  # specific cmd, no password
%devs ALL=(ALL) /usr/bin/apt        # group devs can run apt
Defaults timestamp_timeout=15       # sudo password timeout in minutes

# su — switch user
su alice              # switch to alice (requires alice's password)
su - alice            # switch to alice WITH full login environment
su -                  # switch to root (requires root password)
sudo su -             # switch to root via sudo (uses YOUR password)

# Linux capabilities — fine-grained root privileges
# Instead of full root (setuid), give specific capabilities
getcap /usr/bin/ping              # show capabilities of a file
setcap cap_net_bind_service=+ep /usr/bin/node  # allow binding port <1024
setcap -r /usr/bin/node           # remove capabilities
# cap_net_bind_service  bind to ports < 1024 without root
# cap_net_raw           send raw packets (ping)
# cap_sys_time          set system clock
# cap_kill              send signals to any process
```

---

### /etc/passwd, /etc/shadow, /etc/group

```bash
# /etc/passwd — user account info (readable by all)
# Format: username:x:UID:GID:GECOS:home:shell
cat /etc/passwd
# root:x:0:0:root:/root:/bin/bash
# alice:x:1000:1000:Alice Smith:/home/alice:/bin/bash
# nginx:x:101:101::/var/lib/nginx:/usr/sbin/nologin   ← service account

# /etc/shadow — password hashes (readable only by root)
# Format: username:hash:lastchange:min:max:warn:inactive:expire:reserved
sudo cat /etc/shadow
# alice:$6$salt$hashhash...:19000:0:99999:7:::

# /etc/group — group memberships
# Format: groupname:x:GID:member1,member2
cat /etc/group
# sudo:x:27:alice,bob      ← alice and bob are in sudo group

# Check current user info
id                   # uid, gid, and all groups
id alice             # info for another user
who                  # who is logged in right now
w                    # who is logged in and what they're doing
last                 # login history from /var/log/wtmp
last -n 20           # last 20 logins
lastlog              # most recent login for all users
```

## 7. Users & Groups

### Creating and Managing Users

```bash
# useradd — create a new user
useradd alice                            # create user (minimal setup)
useradd -m alice                         # create with home directory
useradd -m -s /bin/bash alice            # with home dir and bash shell
useradd -m -g users -G sudo,docker alice # primary group + supplementary groups
useradd -u 1500 alice                    # specify UID
useradd -r nginx                         # system user (low UID, no home, no login)
useradd -d /opt/service svc             # custom home directory

# usermod — modify existing user
usermod -aG sudo alice         # add alice to sudo group (-a = append, REQUIRED)
usermod -aG docker,www-data alice  # add to multiple groups
usermod -s /bin/zsh alice      # change shell
usermod -l newname oldname     # rename user account
usermod -L alice               # lock account (prepends ! to password hash)
usermod -U alice               # unlock account
usermod -e 2025-12-31 alice    # set account expiry date

# userdel — delete user
userdel alice                  # delete user (keep home dir)
userdel -r alice               # delete user AND home directory + mail spool

# Groups
groupadd developers            # create group
groupadd -g 1500 developers    # with specific GID
groupmod -n newname oldname    # rename group
groupdel developers            # delete group (fails if it's a user's primary group)

# passwd — password management
passwd alice              # set/change alice's password (root: no current needed)
passwd                    # change your own password
passwd -l alice           # lock account
passwd -u alice           # unlock account
passwd -e alice           # expire password (force change at next login)
passwd -d alice           # delete password (empty password — dangerous!)

# chage — password aging
chage -l alice                    # show password aging info
chage -M 90 alice                 # max password age: 90 days
chage -m 1 alice                  # min days between changes: 1
chage -W 14 alice                 # warn 14 days before expiry
chage -E 2025-12-31 alice         # account expiry date
chage -I 30 alice                 # 30 days grace period after expiry
```

---

### /etc/skel and PAM

```bash
# /etc/skel — template for new user home directories
ls /etc/skel          # .bashrc .profile .bash_logout (and anything you add)
# Files here are COPIED to new home dirs when useradd -m is used
cp custom_bashrc /etc/skel/.bashrc   # all new users get this bashrc

# newgrp — switch active group (affects file creation)
id                     # shows all groups
newgrp developers      # start new shell with developers as primary group
touch file.txt         # this file will be owned by group developers

# PAM — Pluggable Authentication Modules
# /etc/pam.d/ contains per-service PAM config
ls /etc/pam.d/         # sshd, sudo, login, passwd, etc.
cat /etc/pam.d/sshd    # PAM rules for SSH authentication

# PAM module types:
# auth       authentication (verify identity)
# account    authorization (can this user log in now?)
# session    session setup/teardown
# password   password changing rules

# Common PAM modules:
# pam_unix.so      — standard Unix password/shadow auth
# pam_limits.so    — enforce ulimit settings from /etc/security/limits.conf
# pam_tally2.so    — account lockout after failed attempts
# pam_google_authenticator.so  — 2FA

# /etc/nsswitch.conf — Name Service Switch
cat /etc/nsswitch.conf
# passwd:   files systemd     ← check /etc/passwd, then systemd
# group:    files systemd
# hosts:    files dns          ← check /etc/hosts, then DNS
# shadow:   files             ← check /etc/shadow only
```

---

## 8. Processes & Jobs

### Process Theory

```
Process states:
  R  Running or runnable (on CPU or in run queue)
  S  Sleeping (interruptible — waiting for event)
  D  Disk sleep (uninterruptible — waiting for I/O, CANNOT be killed)
  Z  Zombie (finished, waiting for parent to collect exit status)
  T  Stopped (by signal SIGSTOP or debugger)
  X  Dead (should never be seen)
  I  Idle (kernel thread)

Process creation (fork-exec model):
  Parent process calls fork()  → creates identical copy (child)
  Child calls exec()           → replaces itself with new program
  Parent calls wait()          → collects child's exit status

Zombie: child has exited but parent hasn't called wait() yet
        Shows as Z in ps — harmless if temporary, leak if permanent
        Fix: kill the PARENT (not the zombie itself)

Orphan: parent died before child
        init/systemd (PID 1) automatically adopts orphans
```

---

### ps — Process Status

```bash
ps                     # processes in current terminal
ps aux                 # all processes (BSD style) — most common
                       # a=all users, u=user-oriented format, x=no TTY required
ps -ef                 # all processes (System V style)
ps -p 1234             # specific PID
ps -u alice            # processes owned by alice
ps --forest            # tree view showing parent/child relationships
ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu | head  # custom columns, sorted by CPU

# ps aux output columns:
# USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND
# VSZ = virtual size, RSS = resident (physical) memory
```

---

### top, htop, btop

```bash
top                    # interactive process monitor
top -u alice           # show only alice's processes
top -p 1234            # monitor specific PID
top -b -n 1            # batch mode, 1 iteration (good for scripting)

# top key bindings:
# k     kill a process      r     renice a process
# M     sort by memory       P     sort by CPU
# u     filter by user       q     quit
# 1     toggle per-CPU view  H     toggle threads
# f     field management

htop                   # ncurses-based, easier to use (apt install htop)
# F2=Setup  F3=Search  F4=Filter  F5=Tree  F6=SortBy  F9=Kill  F10=Quit

btop                   # beautiful resource monitor (apt install btop)
```

---

### pgrep, pkill, pstree

```bash
pgrep nginx            # find PIDs of processes named nginx
pgrep -u alice bash    # find bash processes owned by alice
pgrep -la nginx        # list with process names
pgrep -P 1234          # find children of PID 1234

pkill nginx            # send SIGTERM to all processes named nginx
pkill -9 nginx         # send SIGKILL (force kill)
pkill -u alice         # kill all processes owned by alice
pkill -f "python script.py"  # match against full command line

pidof nginx            # show PIDs of all nginx processes
pstree                 # display process tree
pstree -p              # include PIDs
pstree -u              # include usernames
pstree alice           # only alice's processes
```

---

### Signals

```bash
# Common signals
# SIGTERM (15) — polite termination request — process can ignore or clean up
# SIGKILL (9)  — forcible immediate termination — cannot be caught or ignored
# SIGHUP  (1)  — hangup: reload config (many daemons respond to this)
# SIGINT  (2)  — interrupt from keyboard (Ctrl+C)
# SIGQUIT (3)  — quit from keyboard (Ctrl+\) — creates core dump
# SIGSTOP (19) — pause process — cannot be caught or ignored
# SIGCONT (18) — resume paused process
# SIGUSR1 (10) — user-defined signal 1 (app-specific meaning)
# SIGUSR2 (12) — user-defined signal 2

kill -15 1234          # SIGTERM (graceful stop) — default if no signal given
kill 1234              # same as kill -15
kill -9 1234           # SIGKILL (force kill immediately)
kill -HUP 1234         # SIGHUP (reload config)
kill -0 1234           # check if process exists (no signal sent)
kill -l                # list all signal names and numbers

killall nginx          # kill all processes named nginx
killall -HUP nginx     # send HUP to all nginx processes
```

---

### Job Control

```bash
command &              # run in background; shows [job_num] PID
jobs                   # list current jobs
jobs -l                # include PIDs

fg                     # bring most recent background job to foreground
fg %2                  # bring job 2 to foreground
bg                     # resume stopped job in background
bg %1                  # resume job 1 in background

# Ctrl+Z  — suspend (stop) current foreground job
# Ctrl+C  — send SIGINT (terminate) to foreground job

disown %1              # remove job from shell's job table (won't be killed when shell exits)
disown -h %1           # SIGHUP won't be sent but job stays in table

nohup command &        # run immune to SIGHUP, output to nohup.out
nohup command > out.log 2>&1 &  # redirect nohup output

wait                   # wait for all background jobs to finish
wait 1234              # wait for specific PID
wait %1                # wait for job 1
```

---

### nice, renice, ionice, ulimit

```bash
# nice — start process with adjusted priority (-20=highest, 19=lowest)
nice command                    # start at priority 10 (default)
nice -n 19 heavy_job            # start at lowest priority (nice to others)
nice -n -5 important_task       # higher priority (requires root for negative)

# renice — change priority of RUNNING process
renice -n 10 -p 1234            # lower priority of PID 1234
renice -n -5 -p 1234            # raise priority (requires root)
renice -n 15 -u alice           # lower priority of ALL alice's processes

# ionice — I/O scheduling class
ionice -c 3 command             # idle class: only runs I/O when nothing else needs disk
ionice -c 2 -n 0 command        # best-effort class, highest priority
ionice -c 1 -n 0 command        # real-time class (use with care!)
ionice -p 1234                  # show I/O class of process

# ulimit — resource limits for current shell and children
ulimit -a                       # show all limits
ulimit -n 65535                 # max open file descriptors
ulimit -u 1000                  # max user processes
ulimit -m 512000                # max memory (kbytes)
ulimit -c unlimited             # allow core dumps (for debugging)
ulimit -v 1048576               # virtual memory limit (kbytes)

# Permanent limits: /etc/security/limits.conf
# alice hard nofile 65535       # hard limit: max open files for alice
# @devs soft nproc 2048         # soft limit: max processes for dev group
```

---

### /proc Filesystem

```bash
ls /proc/                      # one directory per PID + virtual files
ls /proc/1234/                 # files for PID 1234

cat /proc/1234/cmdline | tr '\0' ' '  # command line (null-separated → space-separated)
cat /proc/1234/status          # process status: name, PID, PPID, state, memory
ls -la /proc/1234/fd/          # open file descriptors → symlinks to files/sockets
cat /proc/1234/maps            # memory map (virtual address space)
cat /proc/1234/net/tcp         # TCP connections (in hex — use ss instead)
cat /proc/1234/io              # I/O stats: read_bytes, write_bytes
cat /proc/1234/environ | tr '\0' '\n'  # environment variables

# System-wide /proc files
cat /proc/cpuinfo              # CPU details per core
cat /proc/meminfo              # memory statistics
cat /proc/loadavg              # 1/5/15 min load avg, running/total processes, last PID
cat /proc/uptime               # seconds since boot
cat /proc/version              # kernel version string
cat /proc/filesystems          # supported filesystem types
cat /proc/mounts               # currently mounted filesystems
cat /proc/net/dev              # network interface statistics
```

---

### lsof, strace, ltrace

```bash
# lsof — list open files (files, sockets, pipes, devices)
lsof                           # ALL open files (massive output)
lsof -p 1234                   # open files for PID 1234
lsof -u alice                  # open files for user alice
lsof /var/log/syslog           # which processes have this file open
lsof -i                        # all network connections
lsof -i :80                    # processes using port 80
lsof -i :80 -i :443            # processes using ports 80 or 443
lsof -i tcp                    # only TCP connections
lsof -i @1.2.3.4               # connections to specific host
lsof -nP -i tcp -i udp         # -n=no DNS lookup, -P=no port name resolution

# strace — trace system calls
strace command                  # trace all syscalls of a command
strace -p 1234                  # attach to running process
strace -e trace=open,read,write command  # trace only specific syscalls
strace -e trace=network command  # trace only network calls
strace -f command               # follow forks (trace children too)
strace -c command               # summary: count and time each syscall
strace -o output.txt command    # write trace to file

# ltrace — trace library calls
ltrace command                  # trace dynamic library calls
ltrace -p 1234                  # attach to running process
```

---

## 9. System Monitoring & Performance

### Load Averages and CPU

```bash
uptime                         # current time, uptime, load averages
# Output: 14:30:45 up 5 days, 3:22, 2 users, load average: 1.23, 0.89, 0.75
#                                                              ├──────┴─────┘
#                                                           1 min  5 min  15 min

# Load average interpretation:
# = number of CPUs   → fully utilised
# > number of CPUs   → overloaded (queue building up)
# < number of CPUs   → underutilised
# Check CPU count: nproc  or  grep -c processor /proc/cpuinfo

# CPU monitoring tools
mpstat 1 5              # per-CPU stats every 1 sec, 5 times
mpstat -P ALL 1         # all CPUs simultaneously
sar -u 1 10             # CPU utilisation: 1s interval, 10 samples
sar -u -f /var/log/sysstat/sa20  # historical data from specific date
vmstat 1 5              # virtual memory, CPU, I/O stats every 1s

# vmstat output columns:
# procs: r=running, b=blocked
# memory: swpd, free, buff, cache
# swap: si=swap in, so=swap out
# io: bi=blocks in, bo=blocks out
# system: in=interrupts/s, cs=context switches/s
# cpu: us=user, sy=system, id=idle, wa=I/O wait, st=stolen
```

---

### Memory

```bash
free                    # memory usage
free -h                 # human-readable (KB/MB/GB)
free -m                 # megabytes
free -s 2               # update every 2 seconds

# free output explained:
#               total   used    free    shared  buff/cache  available
# Mem:          16G     8G      1G      500M    7G          8G
# Swap:         4G      100M    3.9G

# "available" is what programs can actually USE
# buff/cache is used but can be freed if needed
# DO NOT monitor "free" — monitor "available"

cat /proc/meminfo       # detailed memory breakdown
grep MemAvailable /proc/meminfo  # available memory
grep SwapUsed /proc/meminfo      # swap usage
```

---

### Disk I/O and Network Monitoring

```bash
# iostat — disk I/O statistics
iostat                 # summary since boot
iostat -x 1 5          # extended stats, 1s interval, 5 samples
iostat -h              # human-readable sizes
# Key columns: %util (disk busy %), await (avg wait ms), r/s w/s (reads/writes per sec)

iotop                  # top-like view of disk I/O per process
iotop -o               # only show processes actually doing I/O
iotop -a               # accumulated I/O instead of rates

# Network monitoring
ip -s link             # network interface statistics (errors, packets)
ip -s link show eth0   # stats for specific interface
sar -n DEV 1 5         # network device stats from sar
nload eth0             # visual bandwidth meter per interface
iftop -i eth0          # top-like view of bandwidth per connection
nethogs eth0           # bandwidth usage per PROCESS

# ss — socket statistics (modern replacement for netstat)
ss -tulpn              # tcp+udp, listening, process names, don't resolve names
ss -ta                 # all TCP sockets
ss -tn                 # TCP, numeric (no DNS/port resolution)
ss -tp                 # TCP with process info
ss -s                  # summary statistics
ss -tulpn | grep :80   # what's listening on port 80
ss state established   # only established connections

# netstat (legacy — use ss instead)
netstat -tulpn         # same as ss -tulpn
netstat -rn            # routing table
```

---

### sysctl, dmesg, journalctl

```bash
# sysctl — read/write kernel parameters at runtime
sysctl -a                               # list ALL kernel parameters
sysctl vm.swappiness                    # read specific parameter
sysctl -w vm.swappiness=10             # write parameter (temporary, lost on reboot)
sysctl -p                              # load settings from /etc/sysctl.conf
sysctl -p /etc/sysctl.d/99-custom.conf # load specific file

# Persistent: /etc/sysctl.conf or /etc/sysctl.d/99-custom.conf
# vm.swappiness = 10             ← reduce swap tendency
# net.ipv4.ip_forward = 1       ← enable IP routing
# fs.file-max = 2097152         ← max open file descriptors
# kernel.pid_max = 4194304      ← max PID number

# dmesg — kernel ring buffer messages
dmesg                          # all kernel messages
dmesg | tail -20               # most recent messages
dmesg -H                       # human-readable timestamps
dmesg -T                       # human-readable timestamps (absolute time)
dmesg -l err,warn              # only errors and warnings
dmesg -f kern                  # only kernel facility
dmesg -w                       # follow: show new messages in real-time
dmesg | grep -i "usb\|sda\|eth"  # grep for hardware events

# journalctl — systemd journal
journalctl                     # all journal entries (oldest first)
journalctl -r                  # reverse: newest first
journalctl -f                  # follow: tail -f for the journal
journalctl -n 50               # last 50 lines
journalctl -u nginx            # only nginx service logs
journalctl -u nginx -f         # follow nginx logs live
journalctl --since "1 hour ago"
journalctl --since "2024-01-15 10:00:00" --until "2024-01-15 11:00:00"
journalctl -p err              # only priority: emerg/alert/crit/err/warning/notice/info/debug
journalctl -p err..warning     # range of priorities
journalctl -b                  # this boot only
journalctl -b -1               # previous boot
journalctl -b -2               # two boots ago
journalctl --disk-usage        # how much disk journal is using
journalctl --vacuum-size=500M  # reduce journal to 500MB
journalctl -o json-pretty      # output as JSON
journalctl -k                  # only kernel messages (like dmesg)
```

---

### Performance Methodology

```
USE Method (Brendan Gregg):
For every resource (CPU, Memory, Disk, Network):
  Utilisation — how busy is the resource? (0-100%)
  Saturation  — is there a queue? (queue depth, wait time)
  Errors      — are there errors? (dropped packets, disk errors)

Quick diagnostic checklist:
  uptime            → check load average vs nproc
  dmesg | tail      → hardware errors?
  vmstat 1          → CPU iowait, swap activity?
  mpstat -P ALL 1   → CPU imbalance?
  free -h           → memory pressure?
  iostat -x 1       → disk saturation (%util, await)?
  ss -s             → network connections?
  top               → which processes consuming resources?
```

---

## 10. Networking

### OSI Model — Linux Context

```
Layer 7  Application   HTTP, SSH, DNS, SMTP          ← curl, wget, ssh
Layer 6  Presentation  TLS/SSL encryption             ← openssl
Layer 5  Session       TCP session management         ← 
Layer 4  Transport     TCP/UDP, ports                 ← ss, iptables, nftables
Layer 3  Network       IP addresses, routing          ← ip route, ping, traceroute
Layer 2  Data Link     MAC addresses, Ethernet frames ← ip link, arp
Layer 1  Physical      Cables, signals                ← hardware
```

---

### Network Interfaces and Routing

```bash
# ip addr — interface addresses
ip addr                        # show all interfaces
ip addr show eth0              # specific interface
ip addr add 192.168.1.100/24 dev eth0  # add address
ip addr del 192.168.1.100/24 dev eth0  # remove address

# ip link — interface state
ip link                        # show all interfaces with state
ip link show eth0              # specific interface
ip link set eth0 up            # bring interface up
ip link set eth0 down          # bring interface down
ip link set eth0 mtu 9000      # set MTU (jumbo frames)

# ip route — routing table
ip route                       # show routing table
ip route show table all        # all routing tables
ip route add default via 192.168.1.1       # set default gateway
ip route add 10.0.0.0/8 via 192.168.1.254  # add static route
ip route del 10.0.0.0/8                    # delete route
ip route get 8.8.8.8           # which route would be used to reach 8.8.8.8

# ip neigh — ARP/neighbour table
ip neigh                       # show ARP table
ip neigh flush dev eth0        # flush ARP cache for interface

# DNS
dig google.com                 # DNS lookup (A record)
dig google.com MX              # mail exchange records
dig google.com NS              # nameserver records
dig google.com AAAA            # IPv6 records
dig +short google.com          # just the answer
dig @8.8.8.8 google.com        # use specific DNS server
dig -x 8.8.8.8                 # reverse DNS lookup
nslookup google.com            # alternative (older)
host google.com                # simple DNS lookup
resolvectl status              # show systemd-resolved status
resolvectl query google.com    # resolve via systemd-resolved
cat /etc/resolv.conf           # DNS server config
```

---

### curl — Complete Guide

```bash
# Basic requests
curl https://example.com                   # GET request, print body
curl -s https://example.com                # silent: suppress progress meter
curl -o output.html https://example.com    # save to file
curl -O https://example.com/file.tar.gz   # save with remote filename
curl -L https://example.com               # follow redirects (-L = location)

# POST requests
curl -X POST https://api.example.com/data \
     -H "Content-Type: application/json" \      # set request header
     -d '{"key": "value"}'                      # request body

curl -X POST https://api.example.com/upload \
     -F "file=@/path/to/file.txt"              # multipart form upload

# Authentication
curl -u username:password https://api.example.com   # Basic auth
curl -H "Authorization: Bearer TOKEN" https://api.example.com  # Bearer token

# TLS/SSL
curl -k https://self-signed.example.com    # skip certificate verification ⚠️
curl --cacert /path/to/ca.crt https://...  # use custom CA cert
curl --cert /path/to/client.crt https://...  # client certificate

# Verbose and debugging
curl -v https://example.com                # verbose: show headers, TLS info
curl -I https://example.com               # HEAD request: only headers

# Timeouts
curl --connect-timeout 10 https://...     # connection timeout in seconds
curl --max-time 30 https://...            # max total time in seconds
curl --retry 3 --retry-delay 2 https://...  # retry on failure

# DNS override (useful for testing)
curl --resolve example.com:443:1.2.3.4 https://example.com

# Download with progress
curl -# -O https://example.com/largefile.iso  # progress bar

# Rate limiting
curl --limit-rate 1M https://...          # limit to 1MB/s
```

---

### nc (netcat) — Network Swiss Army Knife

```bash
# TCP/UDP testing
nc -zv example.com 80         # test if port 80 is open (-z=scan, -v=verbose)
nc -zv example.com 80-100     # scan port range
nc -zu example.com 53         # test UDP port 53

# Simple server and client
nc -l 1234                    # listen on TCP port 1234
nc example.com 1234           # connect to port 1234

# File transfer
nc -l 1234 > received.txt     # receiver: listen and save
nc host 1234 < file.txt       # sender: send file

# Chat (both sides type, both sides see)
nc -l 1234                    # side A
nc host 1234                  # side B

# Port scanning
nc -zv host 22 80 443 8080    # check multiple specific ports
for port in 80 443 8080; do nc -zv host $port 2>&1; done

# Banner grabbing
echo "" | nc -w1 host 80      # grab HTTP banner
echo "GET / HTTP/1.0\r\n\r\n" | nc host 80
```

---

### iptables and nftables

```bash
# iptables — Linux firewall (based on netfilter)
# Tables: filter (default), nat, mangle, raw, security
# Chains: INPUT (incoming to host), OUTPUT (outgoing from host), FORWARD (routed traffic)

# View rules
iptables -L -n -v              # list all rules in filter table
iptables -L INPUT -n -v        # list INPUT chain rules
iptables -t nat -L -n          # list nat table rules
iptables -S                    # show rules in command format (easy to save)

# Basic rules
iptables -A INPUT -p tcp --dport 22 -j ACCEPT      # allow SSH inbound
iptables -A INPUT -p tcp --dport 80 -j ACCEPT      # allow HTTP
iptables -A INPUT -p tcp --dport 443 -j ACCEPT     # allow HTTPS
iptables -A INPUT -i lo -j ACCEPT                  # allow loopback
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT  # allow established
iptables -P INPUT DROP                              # default policy: drop all

# Delete rules
iptables -D INPUT -p tcp --dport 80 -j ACCEPT      # delete specific rule
iptables -F INPUT                                   # flush (delete all) INPUT rules
iptables -F                                         # flush all rules

# NAT (masquerading — for router/NAT setup)
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE  # NAT outbound traffic
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT          # allow forwarding

# Save and restore rules
iptables-save > /etc/iptables/rules.v4              # save current rules
iptables-restore < /etc/iptables/rules.v4           # restore rules

# nftables — modern replacement for iptables (default on Debian 10+, RHEL 8+)
nft list ruleset               # show all rules
nft list tables                # list tables
nft add table inet filter      # create table
nft add chain inet filter input { type filter hook input priority 0 \; }
nft add rule inet filter input tcp dport 22 accept

# ufw — Uncomplicated Firewall (Ubuntu default frontend)
ufw status                     # show status and rules
ufw enable                     # enable firewall
ufw disable                    # disable firewall
ufw allow 22/tcp               # allow SSH
ufw allow 'Nginx Full'         # allow by app profile
ufw deny 23                    # deny telnet
ufw delete allow 80/tcp        # delete rule
ufw status numbered            # show rules with numbers
ufw delete 3                   # delete rule number 3
ufw reset                      # reset all rules
```

---

### tcpdump — Packet Capture

```bash
# Always use sudo for packet capture
tcpdump -i eth0                     # capture on eth0
tcpdump -i any                      # capture on all interfaces
tcpdump -i eth0 -w capture.pcap     # save to file (open in Wireshark)
tcpdump -r capture.pcap             # read from file
tcpdump -n                          # don't resolve hostnames
tcpdump -nn                         # don't resolve names or port names
tcpdump -v                          # verbose (more packet details)
tcpdump -vvv                        # very verbose

# BPF filters
tcpdump -i eth0 port 80             # only port 80 traffic
tcpdump -i eth0 host 192.168.1.100  # only traffic to/from specific host
tcpdump -i eth0 src 192.168.1.100   # only from specific source
tcpdump -i eth0 tcp                 # only TCP
tcpdump -i eth0 udp port 53        # DNS traffic
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn|tcp-rst) != 0'  # SYN and RST flags
tcpdump -i eth0 port 80 and host api.example.com
tcpdump -i eth0 not port 22        # everything except SSH

# Useful combinations
tcpdump -i eth0 -nn -c 100         # capture exactly 100 packets
tcpdump -i eth0 -A port 80         # show ASCII payload (HTTP requests)
tcpdump -i eth0 -X port 80         # show hex + ASCII
```

---

### WireGuard VPN Basics

```bash
# Install
apt install wireguard       # Debian/Ubuntu
dnf install wireguard-tools  # Fedora/RHEL

# Generate keys
wg genkey | tee privatekey | wg pubkey > publickey
cat privatekey              # server private key
cat publickey               # server public key

# /etc/wireguard/wg0.conf (server)
# [Interface]
# Address = 10.0.0.1/24
# ListenPort = 51820
# PrivateKey = <server_private_key>
#
# [Peer]
# PublicKey = <client_public_key>
# AllowedIPs = 10.0.0.2/32

wg-quick up wg0             # start WireGuard interface
wg-quick down wg0           # stop
systemctl enable wg-quick@wg0  # enable on boot
wg show                     # show interface and peer status
wg show wg0 latest-handshakes   # when each peer last connected
```

---

## 11. Storage & Filesystems

### Block Device Stack

```
┌─────────────────────────────────────────────────────┐
│               Applications / Filesystem             │
├─────────────────────────────────────────────────────┤
│         VFS (Virtual Filesystem Switch)             │
├─────────────────────────────────────────────────────┤
│      Filesystem (ext4, xfs, btrfs, ...)             │
├─────────────────────────────────────────────────────┤
│         LVM (Logical Volume Manager)                │  ← optional
├─────────────────────────────────────────────────────┤
│       MD (Software RAID — mdadm)                    │  ← optional
├─────────────────────────────────────────────────────┤
│              Block Device (sda, nvme0n1)            │
│              Partitions (sda1, sda2, ...)           │
└─────────────────────────────────────────────────────┘
```

---

### Partitioning

```bash
# View partition layout
lsblk                          # tree view of block devices
lsblk -f                       # include filesystem, UUID, label
fdisk -l                       # list all partition tables
fdisk -l /dev/sda              # specific disk
parted -l                      # parted version (shows GPT + MBR)

# MBR (Master Boot Record) vs GPT (GUID Partition Table)
# MBR: max 4 primary partitions, max disk size 2TB, BIOS boot
# GPT: 128 partitions, 8ZB limit, UEFI, stores backup at end of disk

# fdisk — MBR partitioning (interactive)
fdisk /dev/sdb                 # open partitioning menu ⚠️
# n = new partition  d = delete  p = print  w = write  q = quit  t = type

# gdisk — GPT partitioning (like fdisk but for GPT) ⚠️
gdisk /dev/sdb

# parted — both MBR and GPT ⚠️
parted /dev/sdb
# (parted) mklabel gpt         # create new GPT partition table
# (parted) mkpart primary ext4 1MiB 100%
# (parted) quit

# ALWAYS verify the device before any write operation:
lsblk
fdisk -l /dev/sdb              # confirm it's the right disk
```

---

### LVM — Logical Volume Manager

```
Physical Volumes (PVs) — actual disks/partitions
         ↓
Volume Group (VG) — pool of storage
         ↓
Logical Volumes (LVs) — flexible "virtual partitions"

LVM benefits: resize LVs without downtime, snapshots, spanning multiple disks
```

```bash
# Full LVM setup workflow
# Step 1: Create Physical Volumes
pvcreate /dev/sdb /dev/sdc     # initialise disks as PVs
pvdisplay                      # show PV info
pvs                            # summary

# Step 2: Create Volume Group
vgcreate data-vg /dev/sdb /dev/sdc   # create VG from PVs
vgdisplay                      # show VG info
vgs                            # summary

# Step 3: Create Logical Volumes
lvcreate -L 50G -n web-lv data-vg    # create 50GB LV named web-lv
lvcreate -l 100%FREE -n data-lv data-vg  # use all remaining space
lvdisplay                      # show LV info
lvs                            # summary

# Step 4: Create filesystem on LV
mkfs.ext4 /dev/data-vg/web-lv  # create ext4 filesystem
mount /dev/data-vg/web-lv /var/www

# Extending an LV
lvextend -L +20G /dev/data-vg/web-lv      # add 20GB to LV
lvextend -l +100%FREE /dev/data-vg/web-lv  # use all remaining VG space
resize2fs /dev/data-vg/web-lv              # expand ext4 to fill LV (online!)
xfs_growfs /var/www                        # expand XFS (use mount point)

# Snapshots
lvcreate -s -n web-snap -L 5G /dev/data-vg/web-lv  # create snapshot
lvremove /dev/data-vg/web-snap                       # delete snapshot
```

---

### RAID with mdadm

```bash
# RAID levels:
# RAID 0 — striping: max performance, NO redundancy (one disk fails = total loss)
# RAID 1 — mirroring: full redundancy, 50% capacity
# RAID 5 — striping + parity: N-1 capacity, 1 disk can fail (min 3 disks)
# RAID 6 — striping + double parity: N-2 capacity, 2 disks can fail (min 4)
# RAID 10 — mirror + stripe: high performance + redundancy (min 4 disks)

# Create RAID 1 (mirror)
mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc
# Create RAID 5
mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd

# Monitor
cat /proc/mdstat               # RAID status and sync progress
mdadm --detail /dev/md0        # detailed status

# Save config
mdadm --detail --scan >> /etc/mdadm/mdadm.conf
update-initramfs -u            # rebuild initramfs with RAID config

# Add spare
mdadm /dev/md0 --add /dev/sde  # add hot spare
```

---

### Filesystem Creation and Management

```bash
# Create filesystems ⚠️ DESTROYS existing data
mkfs.ext4 /dev/sdb1            # ext4 (most common for Linux)
mkfs.ext4 -L "data" /dev/sdb1  # with volume label
mkfs.xfs /dev/sdb1             # xfs (default for RHEL)
mkfs.btrfs /dev/sdb1           # btrfs (advanced features)
mkfs.btrfs -d raid1 /dev/sdb /dev/sdc  # btrfs RAID1

# ext4 tools
e2fsck -f /dev/sdb1            # force filesystem check (unmounted!)
tune2fs -L "newlabel" /dev/sdb1  # set label
tune2fs -m 1 /dev/sdb1         # reduce reserved blocks to 1%
tune2fs -l /dev/sdb1           # list filesystem parameters
dumpe2fs /dev/sdb1             # dump full filesystem info

# fsck — filesystem check (run on UNMOUNTED filesystems!)
fsck /dev/sdb1                 # check and repair
fsck -y /dev/sdb1              # auto-answer yes to all prompts
fsck -n /dev/sdb1              # dry run: check only, no repairs

# Btrfs
btrfs subvolume create /data/subvol     # create subvolume
btrfs subvolume snapshot /data /backup/snap  # create snapshot
btrfs subvolume list /data              # list subvolumes
btrfs scrub start /data                 # verify data integrity
btrfs balance start /data               # rebalance data across devices
btrfs send /backup/snap | btrfs receive /remote/  # send snapshot

# Swap
mkswap /dev/sdb2               # create swap on partition
swapon /dev/sdb2               # enable swap
swapoff /dev/sdb2              # disable swap
swapon --show                  # show swap spaces
cat /proc/swaps                # swap status

sysctl vm.swappiness=10        # set swappiness (0=avoid swap, 100=swap aggressively)
echo "vm.swappiness=10" >> /etc/sysctl.conf  # make permanent
```

---

### NFS and Samba

```bash
# NFS Server setup
apt install nfs-kernel-server

# /etc/exports
# /shared/data  192.168.1.0/24(rw,sync,no_subtree_check)
# /readonly     *(ro,sync,no_root_squash)

exportfs -a                    # apply /etc/exports changes
exportfs -v                    # show current exports
systemctl restart nfs-server

# NFS Client
showmount -e 192.168.1.10      # list available exports from server
mount -t nfs 192.168.1.10:/shared/data /mnt/nfs
mount -t nfs4 server:/share /mnt  # NFSv4

# /etc/fstab for NFS
# 192.168.1.10:/shared/data  /mnt/nfs  nfs  defaults,_netdev  0  0

# Samba/CIFS (mount Windows shares)
apt install cifs-utils
mount -t cifs //server/share /mnt/windows -o user=alice,password=pass
mount -t cifs //server/share /mnt -o credentials=/etc/smb-credentials
# credentials file: username=alice\npassword=secret\ndomain=WORKGROUP

# inotifywait — filesystem event monitoring
apt install inotify-tools
inotifywait -m /watched/dir    # monitor directory for changes
inotifywait -m -r /watched/dir -e create,modify,delete  # specific events
inotifywait -m /watched/dir | while read dir event file; do
    echo "$file was $event in $dir"
done
```

---

### Loop Devices and Disk Images

```bash
# losetup — work with loop devices (disk images as block devices)
dd if=/dev/zero of=disk.img bs=1M count=100   # create 100MB image file
losetup /dev/loop0 disk.img                   # attach image to loop device
mkfs.ext4 /dev/loop0                          # create filesystem on it
mount /dev/loop0 /mnt/image                   # mount it
losetup -d /dev/loop0                         # detach when done

losetup -a                                    # show all loop devices
losetup -f                                    # show next free loop device
losetup --find --show disk.img                # attach and show device name

# fallocate — fast file creation (allocates disk space without writing)
fallocate -l 1G test.img       # create 1GB file instantly (much faster than dd)
truncate -s 1G test.img        # create sparse 1GB file (takes almost no space)
truncate -s +500M existing.img # extend file by 500MB
```

---

## 12. Package Management

### APT (Debian/Ubuntu)

```bash
# Update package lists and upgrade
apt update                          # fetch latest package lists
apt upgrade                         # upgrade installed packages
apt full-upgrade                    # upgrade + remove obsolete packages (like dist-upgrade)
apt dist-upgrade                    # same as full-upgrade

# Install and remove
apt install nginx                   # install package
apt install -y nginx                # non-interactive (auto yes)
apt install nginx=1.24.0-1          # install specific version
apt remove nginx                    # remove package (keep config files)
apt purge nginx                     # remove package AND config files
apt autoremove                      # remove orphaned dependencies

# Search and info
apt search keyword                  # search package names and descriptions
apt show nginx                      # detailed package info
apt list --installed                # list installed packages
apt list --upgradable               # list packages with upgrades available

# apt-cache (lower level)
apt-cache search nginx
apt-cache show nginx
apt-cache policy nginx              # show available versions and priorities
apt-cache depends nginx             # show dependencies

# /etc/apt/sources.list — package repositories
# deb http://archive.ubuntu.com/ubuntu jammy main restricted universe
# deb http://security.ubuntu.com/ubuntu jammy-security main restricted

# Add PPA (Ubuntu)
add-apt-repository ppa:nginx/stable
apt update && apt install nginx

# dpkg — low-level package tool
dpkg -i package.deb                 # install .deb file
dpkg -r package                     # remove package (keep config)
dpkg -P package                     # purge package and config
dpkg -l                             # list all installed packages
dpkg -l | grep nginx                # search installed packages
dpkg -L nginx                       # list files installed by package
dpkg -S /usr/bin/nginx              # which package owns this file
dpkg --contents package.deb         # list contents of .deb without installing
dpkg --get-selections               # list all packages with their status

# Package pinning — control which version gets installed
# /etc/apt/preferences.d/nginx-pin
# Package: nginx
# Pin: version 1.24.*
# Pin-Priority: 1001
```

---

### DNF/YUM (RHEL/Fedora/CentOS)

```bash
# DNF is the modern replacement for YUM (same syntax mostly)
dnf update                          # update all packages
dnf install nginx                   # install package
dnf install -y nginx                # non-interactive
dnf remove nginx                    # remove package
dnf autoremove                      # remove orphans
dnf search nginx                    # search packages
dnf info nginx                      # package info
dnf list installed                  # list installed
dnf list available | grep nginx     # search available
dnf provides /usr/bin/nginx         # which package provides a file
dnf history                         # transaction history
dnf history undo 5                  # undo transaction number 5
dnf group install "Development Tools"  # install package group
dnf repolist                        # list enabled repositories
dnf repolist all                    # all repos including disabled
dnf config-manager --add-repo URL   # add repository

# /etc/yum.repos.d/*.repo
# [nginx-stable]
# name=nginx stable repo
# baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
# gpgcheck=1
# enabled=1

# RPM — low-level package tool
rpm -i package.rpm                  # install RPM
rpm -U package.rpm                  # upgrade (install if not present)
rpm -e package                      # erase (uninstall)
rpm -q nginx                        # is nginx installed?
rpm -qa                             # list ALL installed packages
rpm -ql nginx                       # list files in package
rpm -qf /usr/sbin/nginx             # which package owns this file
rpm -qi nginx                       # package info
rpm -qd nginx                       # list documentation files
rpm --verify nginx                  # verify package integrity
```

---

### Pacman (Arch Linux)

```bash
pacman -Sy                          # sync package databases
pacman -Syu                         # sync + upgrade all packages (full update)
pacman -S nginx                     # install package
pacman -R nginx                     # remove package
pacman -Rs nginx                    # remove with unused dependencies
pacman -Rns nginx                   # remove + config files + deps
pacman -Ss keyword                  # search available packages
pacman -Qs keyword                  # search installed packages
pacman -Si nginx                    # info on available package
pacman -Qi nginx                    # info on installed package
pacman -Ql nginx                    # list files in installed package
pacman -Qo /usr/bin/nginx           # which package owns file
pacman -U package.pkg.tar.zst       # install from local file
pacman -Sc                          # clear package cache
pacman -Qdt                         # list orphaned packages
pacman -Rns $(pacman -Qdtq)        # remove all orphans

# AUR (Arch User Repository) — community packages
# Using yay (AUR helper):
yay -S package-from-aur            # install from AUR
yay -Syu                           # update including AUR packages

# makepkg — build package from PKGBUILD
git clone https://aur.archlinux.org/package.git
cd package/
makepkg -si                        # build and install
```

---

### Snap, Flatpak, AppImage

```bash
# Snap — canonical's universal package format
snap install vlc                   # install snap
snap install vlc --classic         # install with classic confinement (less restricted)
snap remove vlc                    # remove
snap list                          # list installed snaps
snap refresh                       # update all snaps
snap refresh vlc                   # update specific snap
snap find keyword                  # search snap store
snap info vlc                      # snap details

# Flatpak — distribution-agnostic packages
flatpak install flathub org.videolan.VLC  # install from Flathub
flatpak run org.videolan.VLC              # run flatpak app
flatpak uninstall org.videolan.VLC        # remove
flatpak list                              # list installed
flatpak update                            # update all
flatpak remotes                           # list remote repositories
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

# AppImage — portable, no installation needed
chmod +x application.AppImage     # make executable
./application.AppImage             # run directly (self-contained)

# Compiling from source — traditional approach
wget https://example.com/source-1.0.tar.gz
tar -xzf source-1.0.tar.gz
cd source-1.0/
./configure --prefix=/usr/local    # configure (check dependencies)
make -j$(nproc)                    # compile using all CPU cores
sudo make install                  # install to prefix

# checkinstall — create package from compiled source
sudo checkinstall                  # creates .deb/.rpm and installs it (instead of make install)
```

## 13. Services & systemd

### systemd Architecture

```
systemd (PID 1)
├── Units (configuration files describing system resources)
│   ├── .service    — daemons and processes
│   ├── .timer      — scheduled jobs (cron replacement)
│   ├── .socket     — socket-activated services
│   ├── .mount      — filesystem mounts
│   ├── .target     — groups of units (like runlevels)
│   ├── .path       — path-based activation
│   └── .slice      — cgroup resource management
├── Dependency resolver (handles Before=, After=, Requires=, Wants=)
├── Journal (journald — centralized logging)
└── cgroups integration (resource control per service)

Unit file locations (searched in order):
  /etc/systemd/system/       ← admin-created (highest priority)
  /run/systemd/system/       ← runtime generated
  /lib/systemd/system/       ← package-installed (don't edit here)
```

---

### systemctl — Complete Reference

```bash
# Service control
systemctl start nginx           # start service
systemctl stop nginx            # stop service
systemctl restart nginx         # stop + start
systemctl reload nginx          # reload config without restart (if supported)
systemctl enable nginx          # auto-start on boot (creates symlink)
systemctl disable nginx         # don't auto-start on boot
systemctl mask nginx            # completely disable (prevent start by anything)
systemctl unmask nginx          # remove mask

# Status and inspection
systemctl status nginx          # detailed status: state, logs, PID, memory
systemctl is-active nginx       # returns "active" or "inactive" (exit code 0/1)
systemctl is-enabled nginx      # returns "enabled", "disabled", "masked"
systemctl is-failed nginx       # returns "failed" or other

# Listing units
systemctl list-units                       # all active units
systemctl list-units --type=service        # only services
systemctl list-units --state=failed        # only failed services
systemctl list-unit-files                  # all installed unit files
systemctl list-unit-files --type=service   # only services

# After modifying unit files
systemctl daemon-reload         # reload systemd config (ALWAYS run after editing unit files)

# Targets (like runlevels)
systemctl get-default           # current default target
systemctl set-default multi-user.target   # set default
systemctl isolate rescue.target           # switch to rescue mode
systemctl reboot                # reboot system
systemctl poweroff              # power off
systemctl suspend               # suspend

# Dependencies
systemctl list-dependencies nginx         # show dependency tree
systemctl list-dependencies --reverse nginx  # show what depends on nginx
```

---

### Writing a Service Unit File

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application Server
Documentation=https://example.com/docs
After=network-online.target postgresql.service   # start after these
Wants=network-online.target                       # soft dependency
Requires=postgresql.service                       # hard dependency

[Service]
Type=simple                       # simple: ExecStart is the main process
                                  # forking: forks to background, writes PID file
                                  # oneshot: runs once and exits
                                  # notify: like simple but sends ready notification
                                  # dbus: ready when name appears on D-Bus

User=appuser                      # run as this user
Group=appgroup
WorkingDirectory=/opt/myapp       # working directory

ExecStart=/opt/myapp/bin/server --config /etc/myapp/config.yml
ExecStop=/opt/myapp/bin/server --stop
ExecReload=/bin/kill -HUP $MAINPID  # signal to reload config

# Environment
Environment="PORT=8080" "ENV=production"
EnvironmentFile=/etc/myapp/env    # load environment from file (one KEY=value per line)

# Restart policy
Restart=always                    # always restart if it exits
Restart=on-failure                # restart only on non-zero exit
RestartSec=5s                     # wait 5s between restarts

# Resource limits
LimitNOFILE=65535                 # max open files
LimitNPROC=4096                   # max processes
MemoryLimit=512M                  # memory limit (cgroup)
CPUQuota=50%                      # max 50% of one CPU

# Security hardening
NoNewPrivileges=yes               # can't gain privileges via setuid/etc
ProtectSystem=strict              # /usr, /boot, /etc are read-only
ProtectHome=yes                   # /home, /root are inaccessible
PrivateTmp=yes                    # private /tmp
ReadWritePaths=/var/lib/myapp     # explicitly allow write here
CapabilityBoundingSet=CAP_NET_BIND_SERVICE  # only allow this capability

[Install]
WantedBy=multi-user.target        # enable under multi-user (non-graphical server) target
```

```bash
# Deploy new service
cp myapp.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now myapp      # enable AND start immediately
systemctl status myapp
```

---

### systemd Timers (Cron Replacement)

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily backup timer

[Timer]
OnCalendar=daily                  # every day at midnight
OnCalendar=*-*-* 02:00:00         # every day at 2 AM
OnCalendar=Mon-Fri 09:00          # weekdays at 9 AM
OnCalendar=*:0/15                 # every 15 minutes
OnBootSec=5min                    # 5 minutes after boot
OnUnitActiveSec=1h                # 1 hour after service last ran
Persistent=true                   # run missed jobs after downtime
RandomizedDelaySec=5min           # add random delay (spread load)

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/backup.service (the actual job)
[Unit]
Description=Daily backup
[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
User=backup
```

```bash
systemctl enable --now backup.timer   # enable and start timer
systemctl list-timers                 # show all timers and next run time
systemctl list-timers --all           # include inactive timers
```

---

### cron and crontab

```bash
crontab -e                  # edit YOUR crontab
crontab -l                  # list your crontab
crontab -r                  # REMOVE your entire crontab ⚠️
crontab -u alice -e         # edit alice's crontab (as root)
crontab -u alice -l

# Crontab syntax:
# ┌─── minute (0-59)
# │ ┌─── hour (0-23)
# │ │ ┌─── day of month (1-31)
# │ │ │ ┌─── month (1-12)
# │ │ │ │ ┌─── day of week (0-7, 0 and 7 = Sunday)
# │ │ │ │ │
# * * * * *  command

# Examples:
# 0 * * * *     every hour at minute 0
# */15 * * * *  every 15 minutes
# 0 2 * * *     daily at 2 AM
# 0 2 * * 0     weekly on Sunday at 2 AM
# 0 2 1 * *     monthly on 1st at 2 AM
# 0 2 * * 1-5   weekdays at 2 AM
# @reboot        once at startup
# @daily         once a day (= 0 0 * * *)
# @weekly        once a week
# @monthly       once a month

# System cron directories (run as root)
ls /etc/cron.d/          # arbitrary cron files with username field
ls /etc/cron.daily/      # scripts run daily
ls /etc/cron.hourly/     # scripts run hourly
ls /etc/cron.weekly/     # scripts run weekly

# at — one-time scheduled job
at 14:30                 # run once at 2:30 PM today (interactive input)
at now + 1 hour          # run 1 hour from now
echo "command" | at midnight
atq                      # list pending at jobs
atrm 3                   # remove job number 3
```

---

### systemd-analyze and System Config Commands

```bash
# Boot time analysis
systemd-analyze                         # total boot time
systemd-analyze blame                   # time each service took to start
systemd-analyze critical-chain          # longest dependency chain
systemd-analyze plot > boot.svg         # generate SVG graph of boot

# System configuration
hostnamectl                            # show hostname, OS, kernel
hostnamectl set-hostname newname       # change hostname

timedatectl                            # show time, timezone, NTP status
timedatectl set-timezone Europe/London  # change timezone
timedatectl set-ntp true               # enable NTP synchronisation
timedatectl list-timezones | grep Europe

localectl                              # show locale and keyboard settings
localectl set-locale LANG=en_US.UTF-8  # change locale
localectl list-locales

loginctl list-sessions                 # active login sessions
loginctl list-users                    # logged-in users
loginctl show-session 1               # session details
loginctl terminate-session 1          # kill a session
```

---

## 14. SSH & Remote Access

### SSH Protocol Overview

```
SSH Connection Phases:
1. TCP connection to port 22
2. Protocol version negotiation
3. Algorithm negotiation (key exchange, cipher, MAC, compression)
4. Key exchange (Diffie-Hellman or ECDH) → shared secret
5. Server authentication (host key sent, client verifies against known_hosts)
6. User authentication (password OR public key)
7. Encrypted session begins

Key types (prefer ed25519 for new keys):
  ed25519    — modern, compact, fast, most secure
  ecdsa-p256 — good, hardware token compatible
  rsa-4096   — legacy systems compatibility (use 4096 not 2048)
  dsa        — deprecated, avoid
```

---

### ssh — Connecting and Port Forwarding

```bash
# Basic connection
ssh user@hostname               # connect
ssh -p 2222 user@hostname       # custom port
ssh hostname                    # use ~/.ssh/config for username
ssh -i ~/.ssh/mykey user@host   # specific private key

# Port forwarding
ssh -L 8080:localhost:80 user@server   # LOCAL: forward localhost:8080 → server:80
                                        # Access http://localhost:8080 → hits server's port 80
ssh -L 5432:db.internal:5432 user@jump # LOCAL: access internal DB through jump host
ssh -R 2222:localhost:22 user@server   # REMOTE: server's port 2222 → YOUR port 22
ssh -D 1080 user@server                # DYNAMIC: SOCKS5 proxy on localhost:1080

# Run without interactive session
ssh -N user@server             # -N: don't execute remote command (just forward)
ssh -f user@server -L ... -N   # -f: go to background before exec

# Jump host (bastion)
ssh -J jumpuser@jumphost finaluser@finalhost   # connect through jump host
ssh -J jump1,jump2 user@final                  # chain multiple jumps

# Execute remote command
ssh user@host "ls /var/log"            # run command, get output locally
ssh user@host "sudo systemctl restart nginx"

# Copy files
scp file.txt user@host:/tmp/           # copy file to remote
scp user@host:/tmp/file.txt .          # copy file from remote
scp -r directory/ user@host:/tmp/     # copy directory recursively
scp -P 2222 file.txt user@host:/tmp/   # custom port

# rsync over SSH
rsync -avz -e "ssh -p 2222" src/ user@host:/dest/
```

---

### SSH Key Management

```bash
# Generate SSH key pair
ssh-keygen -t ed25519 -C "alice@laptop"         # recommended key type
ssh-keygen -t rsa -b 4096 -C "alice@laptop"     # RSA for legacy compat
ssh-keygen -t ed25519 -f ~/.ssh/work_key        # custom filename
# Generates: ~/.ssh/work_key (private) and ~/.ssh/work_key.pub (public)

# Copy public key to server
ssh-copy-id user@hostname                        # adds key to ~/.ssh/authorized_keys
ssh-copy-id -i ~/.ssh/work_key.pub user@host    # specific key
# Manual alternative:
cat ~/.ssh/id_ed25519.pub | ssh user@host "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# ssh-agent — keychain, avoids entering passphrase repeatedly
eval "$(ssh-agent -s)"         # start agent
ssh-add ~/.ssh/id_ed25519      # add key (prompts for passphrase once)
ssh-add -l                     # list keys in agent
ssh-add -D                     # remove all keys from agent

# Known hosts management
cat ~/.ssh/known_hosts          # stored server host keys
ssh-keyscan hostname >> ~/.ssh/known_hosts  # add host key without connecting
ssh-keygen -R hostname          # remove host from known_hosts (when host key changes)
```

---

### ~/.ssh/config

```ini
# ~/.ssh/config — per-host SSH settings

# Default settings for all hosts
Host *
    ServerAliveInterval 60       # send keepalive every 60s
    ServerAliveCountMax 3        # disconnect after 3 missed keepalives
    AddKeysToAgent yes           # automatically add keys to ssh-agent
    IdentityFile ~/.ssh/id_ed25519

# Development server
Host dev
    HostName 192.168.1.100       # actual hostname/IP
    User alice                   # username
    Port 2222                    # non-standard port
    IdentityFile ~/.ssh/dev_key  # specific key for this host

# Production with jump host
Host prod
    HostName 10.0.0.50           # private IP (not directly reachable)
    User deploy
    ProxyJump jumpuser@bastion.example.com  # go through bastion first

# Multiple jump hops
Host deephost
    HostName 10.10.0.100
    ProxyJump alice@jump1.example.com,bob@jump2.internal

# Connection multiplexing (reuse existing connection for speed)
Host *.example.com
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h:%p
    ControlPersist 600           # keep connection open 10 min after last session

# Local port forwarding preset
Host db-tunnel
    HostName 192.168.1.10
    User alice
    LocalForward 5432 db.internal:5432
    LocalForward 6379 redis.internal:6379
```

```bash
# Usage with config
ssh dev                    # connects to 192.168.1.100:2222 as alice
ssh prod                   # goes through bastion automatically
ssh db-tunnel -N &         # set up port forwards in background
```

---

### sshd_config Hardening

```ini
# /etc/ssh/sshd_config — server configuration
# Always test changes before disconnecting: sshd -t (syntax check)

Port 2222                          # non-standard port (reduces noise)
AddressFamily inet                 # IPv4 only (or inet6, or any)

# Authentication
PermitRootLogin no                 # NEVER allow root login
PasswordAuthentication no          # require key-based auth only
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
MaxAuthTries 3                     # lock after 3 failed attempts
LoginGraceTime 20                  # 20s to complete auth

# Allow only specific users
AllowUsers alice bob deploy        # whitelist specific users
AllowGroups sshusers               # or whitelist a group

# Session
ClientAliveInterval 300            # send keepalive every 5 min
ClientAliveCountMax 2              # disconnect after 2 missed keepalives
MaxSessions 5                      # max concurrent sessions per connection

# Security
X11Forwarding no                   # no X11 forwarding
AllowAgentForwarding no            # don't allow agent forwarding (unless needed)
PrintMotd no                       # suppress MOTD (use pam_motd instead)
Banner /etc/ssh/banner             # show legal warning before login
```

```bash
sshd -t                    # test config syntax before restarting
systemctl reload sshd      # apply changes (or sshd -HUP)
```

---

### tmux — Terminal Multiplexer

```bash
# Sessions
tmux                            # start new session
tmux new -s mysession           # start named session
tmux ls                         # list sessions
tmux attach                     # attach to last session
tmux attach -t mysession        # attach to named session
tmux kill-session -t mysession  # kill session

# Key bindings (default prefix: Ctrl+b)
# Sessions
Ctrl+b d          # detach from session
Ctrl+b s          # list and switch sessions
Ctrl+b $          # rename current session
Ctrl+b (          # switch to previous session
Ctrl+b )          # switch to next session

# Windows (tabs)
Ctrl+b c          # create new window
Ctrl+b ,          # rename window
Ctrl+b n          # next window
Ctrl+b p          # previous window
Ctrl+b w          # list all windows
Ctrl+b &          # close current window (confirm)
Ctrl+b 0-9        # switch to window number

# Panes (splits)
Ctrl+b %          # split vertically (left/right)
Ctrl+b "          # split horizontally (top/bottom)
Ctrl+b arrow      # move between panes
Ctrl+b z          # zoom pane to full screen (toggle)
Ctrl+b x          # close current pane (confirm)
Ctrl+b {          # swap pane with previous
Ctrl+b q          # show pane numbers
Ctrl+b Space      # cycle through pane layouts

# Copy mode (scroll, search, copy)
Ctrl+b [          # enter copy mode (use arrows to scroll, q to exit)
Ctrl+b ]          # paste

# Scripting
tmux new-session -d -s work     # create session detached
tmux send-keys -t work "vim" Enter  # send keys to session
tmux split-window -h -t work    # split in existing session
```

```ini
# ~/.tmux.conf — customisation
set -g default-terminal "screen-256color"  # enable colours
set -g history-limit 50000                 # scrollback buffer
set -g mouse on                            # enable mouse
set -g base-index 1                        # windows start at 1
setw -g pane-base-index 1

# Better prefix: Ctrl+a (like screen)
unbind C-b
set -g prefix C-a
bind C-a send-prefix

# Easy pane splitting
bind | split-window -h
bind - split-window -v

# Reload config
bind r source-file ~/.tmux.conf \; display "Config reloaded"
```

---

## 15. Security & Hardening

### fail2ban

```bash
# fail2ban — ban IPs after repeated failed auth attempts
apt install fail2ban
systemctl enable --now fail2ban

# /etc/fail2ban/jail.local — override defaults
# [DEFAULT]
# bantime  = 1h
# findtime = 10m
# maxretry = 5
# banaction = iptables-multiport
#
# [sshd]
# enabled = true
# port    = ssh,2222    ← include custom port!
# logpath = /var/log/auth.log
# maxretry = 3
#
# [nginx-http-auth]
# enabled = true
# logpath = /var/log/nginx/error.log

fail2ban-client status              # show all jails
fail2ban-client status sshd         # status of ssh jail: banned IPs
fail2ban-client unban 1.2.3.4       # manually unban IP
fail2ban-client set sshd banip 1.2.3.4  # manually ban IP

systemctl reload fail2ban           # reload after config change
```

---

### auditd — Linux Audit Framework

```bash
# auditd records security-relevant events
apt install auditd audispd-plugins
systemctl enable --now auditd

# auditctl — manage audit rules
auditctl -l                         # list current rules
auditctl -w /etc/passwd -p rwa -k passwd-changes  # watch file (read/write/attr)
auditctl -w /etc/sudoers -p rwa -k sudoers-changes
auditctl -a always,exit -F arch=b64 -S execve -F uid=0 -k root-commands  # all root commands
auditctl -D                         # delete all rules

# Persistent rules: /etc/audit/rules.d/audit.rules
# -w /etc/passwd -p rwa -k passwd-changes
# -w /etc/shadow -p rwa -k shadow-changes

# ausearch — search audit log
ausearch -k passwd-changes          # search by key
ausearch -ua 1000                   # search by UID
ausearch -ts today                  # today's events
ausearch -ts recent -k root-commands  # recent events

# aureport — summary reports
aureport --summary                  # overall audit summary
aureport -au                        # authentication report
aureport -x                         # executable report (what was run)
aureport -f                         # file access report
```

---

### SELinux

```bash
# SELinux — Mandatory Access Control (RHEL/Fedora/CentOS)
# Every process and file has a context (label)
# Policy defines what contexts can interact

# Modes
getenforce                      # show mode: Enforcing, Permissive, or Disabled
setenforce 0                    # set Permissive (temporarily — logs but doesn't block)
setenforce 1                    # set Enforcing (temporarily — blocks)

# Permanent mode: edit /etc/selinux/config
# SELINUX=enforcing

# View contexts
ls -Z /etc/passwd               # file context
ps axZ | grep nginx             # process context
id -Z                           # current user context

# Context format: user:role:type:level
# system_u:object_r:passwd_file_t:s0

# Manage contexts
restorecon -Rv /var/www/html/   # restore default contexts recursively
chcon -t httpd_sys_content_t /custom/path/  # change context temporarily
semanage fcontext -a -t httpd_sys_content_t "/custom/path(/.*)?"  # permanent
semanage fcontext -l | grep httpd   # list file context definitions

# Boolean settings
getsebool -a                    # list all booleans
getsebool httpd_can_network_connect  # check specific boolean
setsebool -P httpd_can_network_connect on  # enable (-P = permanent)

# Troubleshooting
audit2why < /var/log/audit/audit.log  # explain denials
audit2allow -a                    # generate allow rules from denials
ausearch -m AVC -ts recent        # recent SELinux denials
```

---

### AppArmor

```bash
# AppArmor — MAC system (Ubuntu/Debian default)
aa-status                       # show loaded profiles and enforcement status
aa-enforce /etc/apparmor.d/usr.sbin.nginx  # set profile to enforce mode
aa-complain /etc/apparmor.d/usr.sbin.nginx  # set to complain (log only) mode
aa-disable /etc/apparmor.d/usr.sbin.nginx   # disable profile

# Load/reload profiles
apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx  # reload specific profile
service apparmor reload          # reload all profiles
systemctl reload apparmor

# Generate profile from logs
aa-logprof                       # interactive: suggest profile rules from recent denials
aa-genprof /path/to/program      # generate new profile interactively

# Profiles location
ls /etc/apparmor.d/              # installed profiles
```

---

### openssl CLI

```bash
# Generate private key
openssl genrsa -out private.key 4096          # RSA 4096-bit key
openssl genrsa -aes256 -out private.key 4096  # encrypted with passphrase

# Generate CSR (Certificate Signing Request)
openssl req -new -key private.key -out cert.csr \
    -subj "/C=US/ST=CA/L=SF/O=MyOrg/CN=example.com"

# Generate self-signed certificate
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem \
    -days 365 -nodes -subj "/CN=localhost"

# Inspect certificates
openssl x509 -in cert.pem -text -noout          # view certificate details
openssl x509 -in cert.pem -noout -dates          # expiry dates only
openssl x509 -in cert.pem -noout -subject -issuer

# Test SSL connection
openssl s_client -connect example.com:443        # view TLS handshake + cert
openssl s_client -connect example.com:443 -showcerts  # show full chain
openssl s_client -starttls smtp -connect mail.example.com:587

# Verify certificate
openssl verify -CAfile ca.pem cert.pem

# Encrypt/decrypt files
openssl enc -aes-256-cbc -salt -pbkdf2 -in file.txt -out file.enc   # encrypt
openssl enc -d -aes-256-cbc -pbkdf2 -in file.enc -out file.txt      # decrypt

# Hash files
openssl dgst -sha256 file.txt     # SHA-256 hash (like sha256sum)
openssl dgst -md5 file.txt        # MD5 hash
```

---

### GPG

```bash
# Generate key pair
gpg --full-generate-key         # interactive key generation
gpg --gen-key                   # simplified

# Key management
gpg --list-keys                 # list public keys
gpg --list-secret-keys          # list private keys
gpg --fingerprint alice@example.com  # show key fingerprint

# Import/export
gpg --import alice-public.asc   # import someone's public key
gpg --export -a alice@example.com > alice-public.asc  # export public key
gpg --export-secret-keys -a alice@example.com > alice-private.asc  # backup private

# Encrypt and decrypt
gpg -e -r alice@example.com file.txt         # encrypt for alice (creates file.txt.gpg)
gpg -d file.txt.gpg > file.txt              # decrypt

# Sign and verify
gpg --sign file.txt             # sign (creates file.txt.gpg)
gpg --clearsign file.txt        # clearsign (human-readable)
gpg --detach-sign file.txt      # separate signature file (file.txt.sig)
gpg --verify file.txt.sig file.txt  # verify detached signature

# Symmetric encryption (password, no keys)
gpg --symmetric file.txt        # encrypt with passphrase
```

---

### nmap — Network Scanner

```bash
# Basic scans
nmap 192.168.1.1                 # scan single host (default: 1000 common TCP ports)
nmap 192.168.1.0/24              # scan entire subnet
nmap -iL hosts.txt               # scan hosts from file

# Scan types
nmap -sS target              # SYN scan (stealthy, default for root)
nmap -sT target              # TCP connect scan (no root needed)
nmap -sU target              # UDP scan (slow but important)
nmap -sn 192.168.1.0/24      # ping scan only (host discovery, no port scan)

# Port specification
nmap -p 80 target            # scan specific port
nmap -p 80,443,22 target     # scan specific ports
nmap -p 1-1000 target        # port range
nmap -p- target              # all 65535 ports (slow)

# Service and OS detection
nmap -sV target              # detect service versions
nmap -O target               # OS detection (root required)
nmap -A target               # aggressive: OS + version + scripts + traceroute

# Output formats
nmap -oN output.txt target   # normal text output
nmap -oX output.xml target   # XML output
nmap -oG output.gnmap target # greppable output
nmap -oA output target       # all three formats

# Scripting engine
nmap --script=vuln target    # run vulnerability scripts
nmap --script=http-headers target  # specific script
nmap --script-updatedb       # update script database
```

---

### Common CVE Patterns

```bash
# SUID abuse — programs running as root can be exploited
find / -perm /4000 -type f 2>/dev/null    # find all SUID files
# Audit: compare against expected list
# Check GTFOBins (gtfobins.github.io) for known SUID exploits

# PATH hijacking
echo $PATH                          # check path components
which python3                       # where does python resolve to?
# If attacker controls earlier directory in PATH, they can substitute commands

# /tmp race conditions
# Programs that create predictable files in /tmp can be symlink-attacked
# Fix: use mktemp to create unpredictable temp files
tmpfile=$(mktemp /tmp/myapp.XXXXXX)  # secure temp file creation

# Wildcard injection in cron
# If cron runs: tar czf /backup.tgz /some/dir/*
# An attacker creates: --checkpoint-action=exec=sh privesc.sh in that dir!
# Fix: use absolute paths and avoid wildcards in cron
```

---

## 16. Containers & Virtualisation

### Linux Container Primitives

```
Containers are built from Linux kernel features:

Namespaces — isolation (what a process can SEE):
  pid      — process IDs (container has own PID 1)
  net      — network interfaces, routes, firewall rules
  mnt      — filesystem mount points
  uts      — hostname and domain name
  ipc      — inter-process communication
  user     — user and group IDs (rootless containers)
  cgroup   — cgroup root (container sees its own as /)

cgroups — resource limits (what a process can USE):
  cpu      — CPU time allocation
  memory   — memory limits + OOM control
  blkio    — disk I/O throttling
  devices  — device access control

Union filesystems — layered images (how containers store data):
  overlay2 — Docker default: layers stacked, writes to top layer
  images are read-only layers; running container adds writable layer

cgroups v1 vs v2:
  v1 — separate hierarchy per controller, complex
  v2 — unified hierarchy, simpler, default on modern systems
```

---

### Docker CLI

```bash
# Images
docker pull nginx                   # download image from Docker Hub
docker images                       # list local images
docker inspect nginx                # detailed image info (JSON)
docker rmi nginx                    # remove image
docker image prune                  # remove dangling images
docker image prune -a               # remove ALL unused images

# Containers
docker run nginx                    # run container (foreground)
docker run -d nginx                 # run detached (background)
docker run -d -p 8080:80 nginx      # map host:8080 → container:80
docker run -d -v /host/path:/container/path nginx  # bind mount volume
docker run -d --name myapp nginx    # give container a name
docker run -it ubuntu bash          # interactive with TTY
docker run --rm ubuntu echo hello   # auto-remove when exits
docker run -e KEY=value nginx       # set environment variable
docker run --memory 512m nginx      # memory limit
docker run --cpus 0.5 nginx         # CPU limit (0.5 = half a core)

# Container management
docker ps                           # running containers
docker ps -a                        # all containers (including stopped)
docker stop myapp                   # graceful stop (SIGTERM → SIGKILL)
docker kill myapp                   # immediate kill
docker start myapp                  # start stopped container
docker restart myapp
docker rm myapp                     # remove stopped container
docker rm -f myapp                  # force remove running container

# Interact with running container
docker exec -it myapp bash          # open bash in running container
docker exec myapp cat /etc/nginx/nginx.conf  # run one command
docker logs myapp                   # show container stdout/stderr logs
docker logs -f myapp                # follow logs live
docker logs --tail 50 myapp        # last 50 lines

# Build images
docker build -t myapp:v1 .          # build from Dockerfile in current dir
docker build -t myapp:v1 -f path/Dockerfile .  # specific Dockerfile
docker push myapp:v1                # push to registry

# Volumes
docker volume create mydata         # create named volume
docker volume ls                    # list volumes
docker volume inspect mydata        # volume details
docker volume rm mydata             # remove volume
docker run -v mydata:/data nginx    # use named volume

# Networks
docker network ls                   # list networks
docker network create mynet         # create network
docker run --network mynet nginx    # connect to network
docker network inspect mynet        # network details

# Compose
docker compose up -d                # start all services from docker-compose.yml
docker compose down                 # stop and remove containers
docker compose down -v              # also remove volumes
docker compose logs -f              # follow all service logs
docker compose ps                   # show service status
docker compose exec web bash        # exec into service

# Cleanup
docker system prune                 # remove stopped containers + dangling images
docker system prune -a --volumes    # remove everything unused ⚠️
docker stats                        # live resource usage per container
```

---

### Dockerfile Best Practices

```dockerfile
# Use specific version tags — never 'latest' in production
FROM node:20.10-alpine3.19

# Set working directory
WORKDIR /app

# Copy package files first (better layer caching)
# If only app code changes, dependencies layer is reused
COPY package*.json ./
RUN npm ci --only=production   # install deps (cached unless package.json changes)

# Then copy application code
COPY . .

# Use non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Expose port (documentation — doesn't actually publish)
EXPOSE 3000

# Use HEALTHCHECK
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -q --spider http://localhost:3000/health || exit 1

# Use exec form for CMD (not shell form — handles signals correctly)
CMD ["node", "server.js"]          # exec form: signals go directly to node
# CMD node server.js               # shell form: signals go to sh, not node ⚠️
```

---

### podman — Rootless Containers

```bash
# podman is largely a drop-in replacement for docker
# Key difference: rootless by default (runs without root privileges)

podman run -d -p 8080:80 nginx      # same syntax as docker
podman ps                           # list running containers
podman build -t myapp .             # build image
podman exec -it container bash      # exec into container
podman generate systemd --new container  # generate systemd unit for container

# No daemon — each container is a direct child of user's shell
# Docker: client → dockerd (root daemon) → container
# Podman: client → container (directly, no daemon)

alias docker=podman                 # use as docker drop-in
```

---

### KVM/QEMU Virtualisation

```bash
# Check if CPU supports virtualisation
grep -c vmx /proc/cpuinfo      # Intel VT-x
grep -c svm /proc/cpuinfo      # AMD-V
# Result > 0 = supported

# Install KVM stack
apt install qemu-kvm libvirt-daemon-system virtinst virt-manager

# Add user to libvirt and kvm groups
usermod -aG libvirt,kvm $USER

# virsh — manage VMs from command line
virsh list                          # running VMs
virsh list --all                    # all VMs (including stopped)
virsh start myvm                    # start VM
virsh shutdown myvm                 # graceful shutdown
virsh destroy myvm                  # force stop
virsh reboot myvm
virsh suspend myvm                  # pause (save state in RAM)
virsh resume myvm
virsh snapshot-create-as myvm snap1 # create snapshot
virsh snapshot-list myvm            # list snapshots
virsh snapshot-revert myvm snap1    # revert to snapshot
virsh console myvm                  # connect to VM console (Ctrl+] to exit)
virsh dominfo myvm                  # VM info (RAM, CPUs, etc.)
virsh edit myvm                     # edit VM XML config

# virt-install — create new VM
virt-install \
  --name ubuntu-server \
  --ram 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/ubuntu.qcow2,size=20 \
  --cdrom /path/to/ubuntu.iso \
  --network bridge=virbr0 \
  --os-variant ubuntu22.04

# qemu-img — disk image management
qemu-img create -f qcow2 disk.qcow2 20G   # create 20GB qcow2 image
qemu-img info disk.qcow2                   # image info
qemu-img resize disk.qcow2 +10G            # resize image
qemu-img convert -O qcow2 disk.img disk.qcow2  # convert formats
qemu-img snapshot -c snap1 disk.qcow2     # create snapshot
qemu-img snapshot -l disk.qcow2           # list snapshots
```

---

## 17. Kernel & Boot

### Kernel Modules

```bash
# List and inspect modules
lsmod                          # list all loaded modules
lsmod | grep usb               # find USB-related modules
modinfo ext4                   # detailed info about a module
modinfo -F description ext4    # only the description

# Load and unload modules
modprobe ext4                  # load module (and dependencies)
modprobe -r ext4               # remove module (and unneeded dependencies)
insmod /path/to/module.ko      # load module from file (no dep resolution)
rmmod ext4                     # remove module (fails if in use)

# Persistent module loading
echo "br_netfilter" >> /etc/modules-load.d/kubernetes.conf  # load on boot

# Module parameters
modprobe usbcore autosuspend=0  # pass parameter at load time
# Permanent: /etc/modprobe.d/options.conf
# options usbcore autosuspend=0

# Blacklist a module (prevent loading)
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf
update-initramfs -u            # rebuild initramfs

# DKMS — Dynamic Kernel Module Support
# Automatically recompiles modules when kernel updates
dkms status                    # list DKMS-managed modules
dkms autoinstall               # rebuild for current kernel
```

---

### GRUB2

```bash
# GRUB config files
/boot/grub/grub.cfg            # the actual config (DO NOT edit directly)
/etc/default/grub              # variables that control grub config
/etc/grub.d/                   # scripts that generate grub.cfg

# Rebuild grub config
update-grub                    # Debian/Ubuntu
grub2-mkconfig -o /boot/grub2/grub.cfg  # RHEL/Fedora

# /etc/default/grub key settings
# GRUB_DEFAULT=0               # which menu entry to boot (0-indexed, or "saved")
# GRUB_TIMEOUT=5               # seconds to show menu before auto-booting
# GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"  # kernel params for normal boot
# GRUB_CMDLINE_LINUX=""        # kernel params for ALL boots (including recovery)

# Important kernel boot parameters
# quiet          suppress most boot messages
# ro             mount root read-only initially (normal)
# init=/bin/bash override init (single user, no password!) ⚠️
# rd.break       break before pivot root (allows emergency shell with initramfs)
# nomodeset      disable GPU mode setting (fix black screen)
# iommu=off      disable IOMMU (fix some virtualisation issues)
# mitigations=off disable CPU vulnerability mitigations (max perf, less secure)
# systemd.unit=rescue.target   boot to rescue target

# Boot to recovery/rescue:
# In GRUB menu: press 'e' to edit, add 'rd.break' to linux line, Ctrl+x to boot

# After rd.break (to reset root password):
mount -o remount,rw /sysroot    # remount root as writable
chroot /sysroot                  # enter the real root
passwd root                      # set new password
touch /.autorelabel              # tell SELinux to relabel (RHEL)
exit; reboot
```

---

### initramfs

```bash
# initramfs = Initial RAM Filesystem
# Loaded by kernel before real root FS
# Contains: minimal root, drivers for storage, decrypt LUKS, assemble RAID/LVM

# Rebuild initramfs
update-initramfs -u                    # update current kernel (Debian/Ubuntu)
update-initramfs -u -k all             # update for all installed kernels
update-initramfs -c -k 6.1.0-17       # create for specific kernel

dracut -f                              # rebuild initramfs (RHEL/Fedora)
dracut -f /boot/initramfs-$(uname -r).img $(uname -r)  # explicit naming

mkinitcpio -P                          # rebuild all presets (Arch Linux)

# Inspect initramfs contents
lsinitramfs /boot/initrd.img-$(uname -r) | head -50  # Debian/Ubuntu
lsinitrd                               # Fedora/RHEL
```

---

### sysctl Tuning

```bash
# Common performance and security tunables
# Apply permanently in /etc/sysctl.d/99-custom.conf

# Networking — common web server / container host settings
net.ipv4.ip_forward = 1                    # enable IP routing (for containers/VMs)
net.ipv4.tcp_syncookies = 1               # SYN flood protection
net.core.somaxconn = 65535                 # max socket backlog
net.ipv4.tcp_max_syn_backlog = 65535
net.core.netdev_max_backlog = 65535
net.ipv4.tcp_tw_reuse = 1                 # reuse TIME_WAIT sockets
net.ipv4.ip_local_port_range = 1024 65535 # more ephemeral ports
net.ipv4.tcp_fin_timeout = 15             # faster close of connections

# Memory
vm.swappiness = 10                         # prefer RAM over swap (0=never swap, 100=aggressive)
vm.dirty_ratio = 20                        # % RAM for dirty pages before writeback
vm.overcommit_memory = 1                   # allow memory overcommit (needed for some apps)

# File descriptors
fs.file-max = 2097152                      # system-wide max open files
fs.inotify.max_user_watches = 524288       # for VS Code / file watchers

# Kernel security
kernel.dmesg_restrict = 1                  # only root can read dmesg
kernel.randomize_va_space = 2             # full ASLR (default)
```

---

### OOM Killer

```bash
# OOM Killer: when memory is exhausted, kernel kills processes to free it
# Process with highest oom_score is killed first

cat /proc/$(pidof chrome)/oom_score        # current OOM score (higher = more likely killed)
cat /proc/$(pidof nginx)/oom_score_adj     # adjustment: -1000 to +1000

# Protect a critical process
echo -1000 > /proc/$(pidof sshd)/oom_score_adj  # protect sshd from OOM kill
echo 0 > /proc/$(pidof myapp)/oom_score_adj     # reset to default

# Permanent protection (systemd):
# OOMScoreAdjust=-1000  ← in [Service] section of unit file

# Tune OOM behaviour
sysctl vm.panic_on_oom=0        # don't panic (default)
sysctl vm.panic_on_oom=1        # panic and reboot on OOM

# Monitor OOM events
dmesg | grep -i "oom\|killed"   # find OOM kill events
journalctl -k | grep -i oom     # OOM events in journal
```

---

## 18. Quick Reference Tables

### Essential Commands by Category

| Category | Command | Purpose |
|----------|---------|---------|
| **Navigation** | `cd`, `pwd`, `ls -la`, `tree` | Navigate filesystem |
| **Files** | `cp -a`, `mv`, `rm -i`, `mkdir -p` | File operations |
| **Search** | `find`, `grep -r`, `locate` | Find files and content |
| **Text** | `cat`, `head`, `tail -f`, `less` | View files |
| **Process** | `ps aux`, `top`, `kill`, `pgrep` | Process management |
| **Network** | `ss -tulpn`, `ip addr`, `curl -v` | Network status |
| **Disk** | `df -h`, `du -sh`, `lsblk` | Disk usage |
| **System** | `uname -a`, `uptime`, `free -h` | System info |
| **Services** | `systemctl status`, `journalctl -f` | Service management |
| **Users** | `id`, `who`, `last`, `sudo -l` | User info |
| **Packages** | `apt install`, `dpkg -l`, `which` | Package management |
| **Permissions** | `chmod 755`, `chown`, `stat` | File permissions |
| **Archive** | `tar -czf`, `tar -xzf`, `unzip` | Archives |
| **Monitor** | `vmstat 1`, `iostat -x`, `sar` | Performance |
| **SSH** | `ssh`, `scp`, `ssh-keygen` | Remote access |
| **Security** | `nmap`, `openssl`, `gpg` | Security tools |

---

### Signal Names and Numbers

| Number | Name | Default Action | Description |
|--------|------|----------------|-------------|
| 1 | SIGHUP | Terminate | Hangup — reload config for daemons |
| 2 | SIGINT | Terminate | Interrupt (Ctrl+C) |
| 3 | SIGQUIT | Core dump | Quit (Ctrl+\) |
| 9 | SIGKILL | Terminate | Force kill — cannot be caught |
| 10 | SIGUSR1 | Terminate | User-defined signal 1 |
| 11 | SIGSEGV | Core dump | Segmentation fault |
| 12 | SIGUSR2 | Terminate | User-defined signal 2 |
| 15 | SIGTERM | Terminate | Graceful terminate (default) |
| 17 | SIGCHLD | Ignore | Child stopped or terminated |
| 18 | SIGCONT | Continue | Resume stopped process |
| 19 | SIGSTOP | Stop | Pause — cannot be caught |
| 20 | SIGTSTP | Stop | Stop from terminal (Ctrl+Z) |

---

### Common Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General error |
| 2 | Misuse of shell built-ins |
| 126 | Command found but not executable (permission denied) |
| 127 | Command not found |
| 128 | Invalid exit argument |
| 128+n | Fatal signal n (e.g., 137 = killed by SIGKILL=9, 130 = Ctrl+C/SIGINT=2) |
| 255 | Exit status out of range |

---

### File Permission Octal Reference

| Octal | Binary | Symbolic | Meaning |
|-------|--------|----------|---------|
| 7 | 111 | rwx | Read + Write + Execute |
| 6 | 110 | rw- | Read + Write |
| 5 | 101 | r-x | Read + Execute |
| 4 | 100 | r-- | Read only |
| 3 | 011 | -wx | Write + Execute |
| 2 | 010 | -w- | Write only |
| 1 | 001 | --x | Execute only |
| 0 | 000 | --- | No permissions |

| Common Mode | Symbolic | Typical use |
|-------------|----------|-------------|
| 755 | rwxr-xr-x | Directories, executables |
| 644 | rw-r--r-- | Regular files |
| 600 | rw------- | SSH private keys, secrets |
| 700 | rwx------ | Private directories |
| 664 | rw-rw-r-- | Shared group files |
| 777 | rwxrwxrwx | ⚠️ Avoid — world-writable |
| 4755 | rwsr-xr-x | SUID binary |
| 2755 | rwxr-sr-x | SGID directory |
| 1777 | rwxrwxrwt | Sticky (e.g., /tmp) |

---

### find Expression Reference

| Expression | Example | Matches |
|------------|---------|---------|
| `-name` | `-name "*.log"` | Filename wildcard (case-sensitive) |
| `-iname` | `-iname "*.LOG"` | Case-insensitive name |
| `-type f` | `-type f` | Regular files |
| `-type d` | `-type d` | Directories |
| `-type l` | `-type l` | Symbolic links |
| `-size +100M` | `-size +100M` | Larger than 100MB |
| `-size -1k` | `-size -1k` | Smaller than 1KB |
| `-mtime -7` | `-mtime -7` | Modified in last 7 days |
| `-mtime +30` | `-mtime +30` | Modified more than 30 days ago |
| `-newer file` | `-newer /etc/passwd` | Newer than reference file |
| `-perm 755` | `-perm 755` | Exact permissions |
| `-perm /4000` | `-perm /4000` | Any SUID bit set |
| `-user alice` | `-user alice` | Owned by user |
| `-group dev` | `-group dev` | Owned by group |
| `-empty` | `-empty` | Empty files/dirs |
| `-maxdepth N` | `-maxdepth 2` | Max directory depth |
| `-exec` | `-exec rm {} \;` | Execute command per match |
| `-exec + ` | `-exec gzip {} +` | Execute command, batch args |
| `-delete` | `-delete` | Delete matched files |
| `-print0` | `-print0 \| xargs -0` | Null-separated output |

---

### Regex Special Characters

| Character | grep/sed | awk | Meaning |
|-----------|---------|-----|---------|
| `.` | `.` | `.` | Any single character |
| `*` | `*` | `*` | Zero or more of preceding |
| `+` | `\+` or `-E +` | `+` | One or more of preceding |
| `?` | `\?` or `-E ?` | `?` | Zero or one of preceding |
| `^` | `^` | `^` | Start of line |
| `$` | `$` | `$` | End of line |
| `[abc]` | `[abc]` | `[abc]` | Any character in set |
| `[^abc]` | `[^abc]` | `[^abc]` | Any character NOT in set |
| `\b` | `\b` | none | Word boundary |
| `(a\|b)` | `\(a\|b\)` or `-E` | `(a\|b)` | Alternation (OR) |
| `{n,m}` | `\{n,m\}` or `-E` | `{n,m}` | Between n and m repeats |
| `\d` | `-P \d` | none | Digit (PCRE only for grep) |
| `\w` | `-P \w` | `\w` | Word character |
| `\s` | `-P \s` | none | Whitespace |

---

### Useful /proc and /sys Paths

| Path | Contents |
|------|---------|
| `/proc/cpuinfo` | CPU model, features, count |
| `/proc/meminfo` | RAM stats: MemFree, MemAvailable, Cached |
| `/proc/loadavg` | 1/5/15 min load averages |
| `/proc/uptime` | Seconds since boot |
| `/proc/version` | Kernel version string |
| `/proc/mounts` | Currently mounted filesystems |
| `/proc/filesystems` | Supported FS types |
| `/proc/net/tcp` | TCP socket table (hex) |
| `/proc/net/dev` | Network interface stats |
| `/proc/sys/` | All sysctl parameters |
| `/proc/$PID/cmdline` | Process command line |
| `/proc/$PID/status` | Process state, memory, signals |
| `/proc/$PID/fd/` | Open file descriptors |
| `/proc/$PID/maps` | Virtual memory map |
| `/proc/$PID/environ` | Environment variables |
| `/proc/$PID/io` | I/O bytes read/written |
| `/proc/$PID/oom_score` | OOM kill score |
| `/sys/class/net/` | Network interface parameters |
| `/sys/block/` | Block device parameters |
| `/sys/class/hwmon/` | Hardware sensors |
| `/sys/kernel/debug/` | Kernel debugging info |

---

### Common sysctl Tuning Parameters

| Parameter | Value | Effect |
|-----------|-------|--------|
| `vm.swappiness` | `10` | Prefer RAM, rarely swap |
| `vm.dirty_ratio` | `20` | % RAM for dirty pages |
| `vm.overcommit_memory` | `1` | Allow memory overcommit |
| `net.ipv4.ip_forward` | `1` | Enable IP routing |
| `net.ipv4.tcp_syncookies` | `1` | SYN flood protection |
| `net.core.somaxconn` | `65535` | Larger socket backlog |
| `net.ipv4.tcp_fin_timeout` | `15` | Faster TCP close |
| `net.ipv4.ip_local_port_range` | `1024 65535` | More ephemeral ports |
| `fs.file-max` | `2097152` | System open file limit |
| `fs.inotify.max_user_watches` | `524288` | For file watchers |
| `kernel.pid_max` | `4194304` | Max PID value |
| `kernel.dmesg_restrict` | `1` | Only root reads dmesg |
| `net.ipv4.tcp_tw_reuse` | `1` | Reuse TIME_WAIT sockets |

---

### Package Manager Command Equivalents

| Action | apt (Debian/Ubuntu) | dnf (RHEL/Fedora) | pacman (Arch) |
|--------|--------------------|--------------------|----------------|
| Update DB | `apt update` | `dnf check-update` | `pacman -Sy` |
| Upgrade all | `apt upgrade` | `dnf upgrade` | `pacman -Syu` |
| Install | `apt install pkg` | `dnf install pkg` | `pacman -S pkg` |
| Remove | `apt remove pkg` | `dnf remove pkg` | `pacman -R pkg` |
| Purge | `apt purge pkg` | `dnf remove pkg` | `pacman -Rns pkg` |
| Autoremove | `apt autoremove` | `dnf autoremove` | `pacman -Rns $(pacman -Qdtq)` |
| Search | `apt search key` | `dnf search key` | `pacman -Ss key` |
| Show info | `apt show pkg` | `dnf info pkg` | `pacman -Si pkg` |
| Installed info | `dpkg -l \| grep pkg` | `rpm -q pkg` | `pacman -Qi pkg` |
| List files | `dpkg -L pkg` | `rpm -ql pkg` | `pacman -Ql pkg` |
| Find owner | `dpkg -S /path` | `rpm -qf /path` | `pacman -Qo /path` |
| List installed | `dpkg -l` | `rpm -qa` | `pacman -Qa` |
| Install local | `dpkg -i pkg.deb` | `rpm -i pkg.rpm` | `pacman -U pkg.zst` |
| Repo list | `apt-cache policy` | `dnf repolist` | `pacman -Sl` |
| History | `apt list --upgradable` | `dnf history` | (check pacman.log) |
| Clean cache | `apt clean` | `dnf clean all` | `pacman -Sc` |

---

*Generated with ❤️ — Covers Linux fundamentals through kernel internals for sysadmins, DevOps, and developers.*
*Kernel reference: [kernel.org](https://kernel.org) | Man pages: `man <command>` | TLDRPages: [tldr.sh](https://tldr.sh)*
