# 01a. Linux Firewalls: Understanding iptables

**iptables** is the legendary firewall of Linux. Even if you use UFW or Firewalld, they are just "wrappers" that talk to iptables (or nftables) underneath. Understanding iptables is a superpower.

## 1. The Mental Model (How it works)
Imagine a building (your server) with a security guard (iptables) at the front door.

### A. The Three Chains (The Queues)
Every network packet enters one of these three queues:

1.  **INPUT**: Traffic coming **TO** your server. (e.g., Someone trying to SSH in).
    *   *Guard says*: "Are you allowed to enter?"
2.  **OUTPUT**: Traffic going **FROM** your server. (e.g., Your server downloading a patch).
    *   *Guard says*: "Are you allowed to leave?"
3.  **FORWARD**: Traffic passing **THROUGH** your server. (e.g., If your server is acting as a router/VPN).
    *   *Guard says*: "Are you allowed to pass to the next building?"

### B. The Rules (The Checklist)
The guard has a checklist. He reads it **Top to Bottom**.
*   **Rule 1**: Allow SSH. (Packet matches? -> **ACCEPT**. Stop reading).
*   **Rule 2**: Allow HTTP. (Packet matches? -> **ACCEPT**. Stop reading).
*   **Rule 3**: Block IP 1.2.3.4. (Packet matches? -> **DROP**. Stop reading).
*   **Default Policy**: If no rule matches by the end? -> **DROP** (usually).

---

## 2. Basic Commands

### List Rules
```bash
sudo iptables -L -n -v
```
*   `-L`: List.
*   `-n`: Numeric (Show IPs like `1.2.3.4` instead of `google.com`. Faster).
*   `-v`: Verbose (Show packet counts).

### The "Default Drop" Strategy (Best Practice)
First, we set the default policy to DROP everything. Then we allow only what we need.
*   **Warning**: Do NOT run this over SSH without allowing SSH first, or you will lock yourself out!

```bash
# 1. Allow Loopback (Localhost talking to itself)
iptables -A INPUT -i lo -j ACCEPT

# 2. Allow Established Connections (Don't kill my current SSH session!)
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# 3. Allow SSH (Port 22)
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# 4. Allow Web (Port 80/443)
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# 5. Set Default Policy to DROP (Block everything else)
iptables -P INPUT DROP
```

### Explaining the Flags
*   `-A INPUT`: **Append** to the bottom of the INPUT chain.
*   `-p tcp`: Protocol is TCP.
*   `--dport 22`: Destination Port is 22.
*   `-j ACCEPT`: **Jump** to ACCEPT target (Let it in).
*   `-j DROP`: Silently kill the packet (Attacker sees "Request Timed Out").
*   `-j REJECT`: Kill packet and send "Connection Refused" back (Attacker knows you are there).

---

## 3. Common Scenarios

### Scenario A: Block a specific Attacker IP
You see `192.168.1.50` spamming your logs.
```bash
# -I INPUT 1: Insert at the TOP (Rule #1) of the list
iptables -I INPUT 1 -s 192.168.1.50 -j DROP
```

### Scenario B: Delete a Rule
You made a mistake.
1.  List with line numbers:
    ```bash
    iptables -L --line-numbers
    ```
2.  Delete rule #3 from INPUT:
    ```bash
    iptables -D INPUT 3
    ```

### Scenario C: Port Forwarding (NAT)
You want traffic hitting Port 80 to go to a Docker container on Port 8080.
*   This uses the **NAT** table (`-t nat`).
```bash
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080
```

---

## 4. Saving Your Rules
**Crucial**: `iptables` rules are stored in RAM. If you reboot, they are **GONE**.

### On Ubuntu/Debian
You need a helper package.
```bash
sudo apt install iptables-persistent
sudo netfilter-persistent save
```
This saves rules to `/etc/iptables/rules.v4`.

### On CentOS/RHEL
```bash
service iptables save
```

---

## 5. iptables vs. UFW vs. Firewalld

| Tool | What is it? | Pros | Cons |
| :--- | :--- | :--- | :--- |
| **iptables** | The core tool. | Maximum control. Universal. | Complex syntax. Hard to manage order. |
| **UFW** | Wrapper for Ubuntu. | Simple (`ufw allow 22`). | Hides complexity (can be bad for debugging). |
| **Firewalld** | Wrapper for CentOS. | Dynamic (Zones). | Overkill for simple servers. |

**DevSecOps Advice**: Learn `iptables` syntax because when UFW breaks, or when you are inside a minimal Docker container, `iptables` is all you have.
