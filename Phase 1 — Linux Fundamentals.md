# Phase 1 — Linux Fundamentals

**Goal:** Become comfortable working on a Linux server without a GUI.

## 1. Files and Directories

Learn the basic commands for navigating and managing files and directories:

```bash
pwd
ls
cd
mkdir
touch
cp
mv
rm
find
```

### What to understand

* `pwd` — Show the current directory.
* `ls` — List files and directories.
* `cd` — Change directory.
* `mkdir` — Create a directory.
* `touch` — Create an empty file.
* `cp` — Copy files or directories.
* `mv` — Move or rename files.
* `rm` — Remove files or directories.
* `find` — Search for files and directories.

---

## 2. File Contents

Learn how to read, search, and process text files:

```bash
cat
less
head
tail
grep
sort
uniq
wc
```

### What to understand

* `cat` — Display file contents.
* `less` — Read a large file page by page.
* `head` — Show the beginning of a file.
* `tail` — Show the end of a file.
* `grep` — Search for text.
* `sort` — Sort lines.
* `uniq` — Remove/count repeated lines.
* `wc` — Count lines, words, and characters.

Example:

```bash
grep "ERROR" app.log
```

---

## 3. Linux Permissions

Learn:

```bash
chmod
chown
chgrp
```

Understand the three basic permissions:

```text
r = read
w = write
x = execute
```

Permissions are commonly represented using numbers:

```text
755
644
600
```

### Common examples

```bash
chmod 755 script.sh
chmod 644 config.txt
chmod 600 secret.txt
```

You should understand what each permission means for:

```text
Owner
Group
Others
```

---

## 4. Processes

Learn how to see and manage running processes:

```bash
ps
ps aux
top
kill
kill -9
```

### Important concepts

```bash
ps aux
```

Shows running processes.

```bash
top
```

Shows processes and resource usage in real time.

```bash
kill PID
```

Requests a process to stop gracefully.

```bash
kill -9 PID
```

Forces the process to stop.

**Important:** `kill -9` should normally be a last resort because it does not give the application a chance to clean up properly.

---

## 5. Disk Usage

Learn how to investigate disk space:

```bash
df -h
du -sh
ls -lh
```

### What to understand

```bash
df -h
```

Shows how much disk space is available on mounted filesystems.

```bash
du -sh /path
```

Shows how much space a directory is using.

```bash
ls -lh
```

Shows file sizes in a human-readable format.

### Practice scenario

Pretend it is a **3 AM production incident**:

1. The server reports that the disk is almost full.
2. Find which filesystem is full.
3. Find which directory is consuming space.
4. Find the large file.
5. Remove the unnecessary file.
6. Confirm that disk space has been recovered.

---

## 6. Networking

Learn the basic Linux networking commands:

```bash
ip
ss
ping
curl
wget
dig
nslookup
```

### What to understand

* `ip` — View and manage network interfaces and IP addresses.
* `ss` — View network connections and listening ports.
* `ping` — Test basic network connectivity.
* `curl` — Make HTTP/API requests.
* `wget` — Download files.
* `dig` — Query DNS information.
* `nslookup` — Look up DNS information.

Example:

```bash
curl https://example.com
```

Check listening ports:

```bash
ss -tulpn
```

---

## 7. Logs

Learn how to inspect logs:

```bash
tail -f
journalctl
```

### `tail -f`

Useful for watching a log file while an application is running:

```bash
tail -f /var/log/app.log
```

### `journalctl`

Useful for viewing logs managed by `systemd`:

```bash
journalctl
```

For a specific service:

```bash
journalctl -u nginx
```

Follow new logs:

```bash
journalctl -u nginx -f
```

---

## 8. Important Linux Log Directory

Understand:

```text
/var/log
```

This directory commonly contains system and application logs.

Examples may include:

```text
/var/log/syslog
/var/log/auth.log
/var/log/nginx/
```

The exact files depend on the Linux distribution and services installed.

---

