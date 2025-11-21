# 01. Linux Security: The Hardening Handbook

Linux is the operating system of the internet. 90% of the cloud runs on Linux. If you cannot secure Linux, you cannot secure the cloud.

## 1. The Philosophy of Hardening
**Hardening** is the process of reducing the "Attack Surface."
*   **Attack Surface**: The sum of all points where an unauthorized user can try to enter data to or extract data from an environment.
*   **Goal**: Remove everything that is not absolutely necessary. "If it's not installed, it can't be hacked."

---

## 2. User & Access Management (The Front Door)

### The Root Account
*   **Rule #1**: NEVER log in as root.
*   **Rule #2**: Disable the root account password.
    ```bash
    passwd -l root
    ```

### Sudo Configuration (`/etc/sudoers`)
*   **Visudo**: Always use `visudo` to edit this file. It checks for syntax errors so you don't lock yourself out.
*   **Least Privilege Example**:
    *   *Scenario*: A junior dev needs to restart Nginx, but nothing else.
    *   *Config*:
        ```text
        # User  Host=(RunAs)  Command
        junior_dev ALL=(root) /bin/systemctl restart nginx
        ```
    *   *Testing*: If `junior_dev` tries `sudo cat /etc/shadow`, it gets "Sorry, user is not allowed to execute...".

### Password Policies (`/etc/login.defs` & `libpam-pwquality`)
Enforce strong passwords.
*   **File**: `/etc/security/pwquality.conf`
    ```conf
    minlen = 14
    dcredit = -1  # At least 1 digit
    ucredit = -1  # At least 1 uppercase
    ocredit = -1  # At least 1 special char
    retry = 3
    ```
*   **Expiration**: Force password rotation (Controversial, but often required by compliance).
    ```bash
    chage -M 90 username # Password expires in 90 days
    ```

---

## 3. SSH Hardening (The Most Critical Config)
SSH is the most attacked port (22). Bots scan it 24/7.

### The `sshd_config` Deep Dive
File: `/etc/ssh/sshd_config`

| Setting | Value | Reason |
| :--- | :--- | :--- |
| `Port` | `2222` | **Security by Obscurity**. Stops 99% of automated bot scripts. |
| `PermitRootLogin` | `no` | Prevents brute-forcing the 'root' user. |
| `PasswordAuthentication` | `no` | **CRITICAL**. Forces use of SSH Keys. Passwords can be guessed; 4096-bit keys cannot. |
| `PubkeyAuthentication` | `yes` | Enables key-based auth. |
| `PermitEmptyPasswords` | `no` | Obvious. |
| `X11Forwarding` | `no` | Prevents GUI forwarding (reduces attack surface). |
| `MaxAuthTries` | `3` | Disconnects after 3 failed attempts. |
| `AllowUsers` | `admin deploy` | Whitelist approach. Only these users can connect. |
| `Protocol` | `2` | Protocol 1 is insecure. |

### SSH Key Management
*   **Generate a strong key**:
    ```bash
    ssh-keygen -t ed25519 -C "email@example.com"
    ```
    *   *Note*: `ed25519` is newer, smaller, and faster than `rsa`.
*   **Permissions**:
    *   `~/.ssh` directory: `700` (rwx------)
    *   `~/.ssh/authorized_keys`: `600` (rw-------)
    *   If permissions are wrong, SSH will silently fail (StrictModes).

---

## 4. Network Security (Firewall & Fail2Ban)

### UFW (Uncomplicated Firewall)
A wrapper around `iptables`. Easier to use.
```bash
# 1. Reset to safe defaults
ufw default deny incoming
ufw default allow outgoing

# 2. Allow SSH (on your custom port!)
ufw allow 2222/tcp

# 3. Allow Web Traffic
ufw allow 80/tcp
ufw allow 443/tcp

# 4. Enable
ufw enable
```

### Fail2Ban (The Bouncer)
Scans logs for malicious behavior and bans IPs automatically.

*   **Install**: `apt install fail2ban`
*   **Config**: Create `/etc/fail2ban/jail.local`
    ```ini
    [DEFAULT]
    bantime = 1h
    findtime = 10m
    maxretry = 5

    [sshd]
    enabled = true
    port = 2222
    logpath = /var/log/auth.log
    backend = systemd
    ```
*   **How it works**: If an IP fails SSH login 5 times in 10 minutes, `iptables` drops their packets for 1 hour.

---

## 5. File System Security

### Permissions 101
*   **Read (4)**, **Write (2)**, **Execute (1)**.
*   **777**: `rwxrwxrwx` (Everyone can do everything). **NEVER DO THIS**.
*   **SUID (Set User ID)**:
    *   Files that run as root, even if you are a normal user. (e.g., `passwd` needs to update `/etc/shadow`).
    *   **Danger**: If a SUID binary has a bug, a user can become root.
    *   **Audit**: Find all SUID files:
        ```bash
        find / -perm -4000 -type f 2>/dev/null
        ```

### Disk Encryption (LUKS)
*   **Linux Unified Key Setup**. Encrypts the entire partition.
*   Essential for laptops or physical servers. If someone steals the hard drive, data is garbage without the passphrase.

---

## 6. Auditing & Logging (The Black Box)

### Auditd (The Flight Recorder)
Logs every system call. Required for PCI-DSS/HIPAA.

*   **Install**: `apt install auditd`
*   **Rules File**: `/etc/audit/audit.rules`
    ```bash
    # Delete all existing rules
    -D

    # Buffer Size
    -b 8192

    # Watch critical files
    -w /etc/passwd -p wa -k identity
    -w /etc/shadow -p wa -k identity
    -w /etc/group -p wa -k identity
    -w /etc/sudoers -p wa -k sudo_actions

    # Watch for file deletion (by users with ID >= 1000)
    -a always,exit -F arch=b64 -S unlink -S unlinkat -S rename -S renameat -F auid>=1000 -F auid!=4294967295 -k delete

    # Lock the configuration (Must be the last rule)
    -e 2
    ```
*   **Search Logs**:
    ```bash
    ausearch -k identity  # Show me who touched /etc/passwd
    ```

---

## 7. Automated Hardening Script
In DevSecOps, we don't do this manually. We use a script (or Ansible).

```bash
#!/bin/bash
# hardening.sh - Basic Server Hardening

echo "Starting Hardening..."

# 1. Update System
apt-get update && apt-get upgrade -y

# 2. Secure Shared Memory (Prevents some exploits)
echo "tmpfs /run/shm tmpfs defaults,noexec,nosuid 0 0" >> /etc/fstab

# 3. Network Hardening (Sysctl)
cat <<EOF >> /etc/sysctl.conf
# Disable IP Spoofing
net.ipv4.conf.all.rp_filter = 1
# Disable ICMP Redirects (Man in the Middle)
net.ipv4.conf.all.accept_redirects = 0
# Disable IP Forwarding (Don't act as a router)
net.ipv4.ip_forward = 0
EOF
sysctl -p

# 4. Set Permissions on Bootloader
chown root:root /boot/grub/grub.cfg
chmod 600 /boot/grub/grub.cfg

# 5. Install Security Tools
apt-get install -y fail2ban auditd ufw

echo "Hardening Complete. Please configure SSH and UFW manually."
```

---

## 8. Incident Response: "I've been hacked!"
What do you do if you see a strange process running as root?

1.  **Don't Panic**.
2.  **Isolate**: Disconnect the network cable (or block via Security Group). Do NOT turn it off (you lose RAM evidence).
3.  **Identification**:
    *   `top` / `htop`: Check high CPU usage.
    *   `netstat -tulnp`: Check for strange open ports (e.g., connection to a Russian IP).
    *   `lsof -p <PID>`: See what files the process has open.
    *   `ls -la /proc/<PID>/exe`: See where the executable really is.
4.  **Containment**:
    *   Pause the instance (VM).
    *   Snapshot the disk for forensics.
5.  **Eradication**:
    *   **Nuke it from orbit**. In DevSecOps, we rarely "clean" a server. We terminate the instance and spin up a fresh, patched one from Terraform.

---

## 9. Interview Preparation

### Q1: How do you secure a Linux server exposed to the internet?
*   **Answer**:
    1.  **Update**: Patch all packages.
    2.  **User**: Create a non-root user, disable root login.
    3.  **SSH**: Key-based auth only, change port, fail2ban.
    4.  **Firewall**: UFW deny all incoming except 80/443/SSH.
    5.  **SELinux**: Ensure it is Enforcing.
    6.  **Logging**: Enable auditd and ship logs to ELK/Splunk.

### Q2: What is a "Fork Bomb" and how do you stop it?
*   **Answer**:
    *   A script that creates copies of itself recursively until the system freezes. `:(){ :|:& };:`
    *   **Prevention**: `ulimit`. Limit the number of processes a user can spawn. Edit `/etc/security/limits.conf`.

### Q3: What is the difference between `chmod` and `chown`?
*   **Answer**:
    *   `chmod` changes **permissions** (Who can do what? Read/Write/Execute).
    *   `chown` changes **ownership** (Who owns the file?).

### Q4: Explain "Sticky Bit".
*   **Answer**:
    *   A permission bit set on a directory (usually `/tmp`).
    *   It ensures that **only the file owner** can delete their own files, even if the directory is writable by everyone.
    *   `chmod +t /tmp`

---

## 10. Hands-On Lab
1.  Spin up an Ubuntu VM (VirtualBox or AWS Free Tier).
2.  Create a new user `admin_user`.
3.  Setup SSH Keys and disable password auth.
4.  Install `fail2ban` and try to brute-force yourself (fail 5 times). Watch the ban happen (`iptables -L`).
5.  Write a script to check if `root` is locked.
