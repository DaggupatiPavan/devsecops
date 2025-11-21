# 07. AWS Cloud Security: The Shared Responsibility

AWS secures the concrete; you secure the config.

## 1. IAM: The Identity Firewall
Identity is the new perimeter.

### A. JSON Policy Deep Dive
You must be able to read and write JSON policies.

#### The Anatomy of a Policy
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowS3Read",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::my-bucket",
                "arn:aws:s3:::my-bucket/*"
            ],
            "Condition": {
                "IpAddress": {"aws:SourceIp": "1.2.3.4/32"}
            }
        }
    ]
}
```
*   **Effect**: Allow or Deny. (Deny always wins).
*   **Action**: What API call?
*   **Resource**: Which object?
*   **Condition**: When? (e.g., Only from VPN IP).

### B. Roles vs. Users
*   **User**: Long-term credentials (AKIA...). Avoid creating these.
*   **Role**: Temporary credentials (ASIA...). Use these for EC2, Lambda, and Humans (via SSO).

### C. Cross-Account Access
How does Dev account access Prod account?
*   **Trust Policy**: The "Handshake".
    *   Prod Role says: "I trust Account Dev to assume me."
    *   Dev User says: "I have permission to assume Prod Role."

---

## 2. AWS CLI for Security Auditing
The GUI is slow. Pros use the CLI.

### Top Security Commands
1.  **Check Password Policy**:
    ```bash
    aws iam get-account-password-policy
    ```
2.  **Find Keys older than 90 days**:
    ```bash
    aws iam generate-credential-report
    aws iam get-credential-report --output text | base64 -d | cut -d, -f1,4,5,6
    ```
3.  **List Public S3 Buckets**:
    ```bash
    # Requires complex filtering, usually done with tools like Prowler
    ```
4.  **Check Security Groups with Open 0.0.0.0/0**:
    ```bash
    aws ec2 describe-security-groups --filters Name=ip-permission.cidr,Values=0.0.0.0/0
    ```

---

## 3. VPC Security: Network ACLs vs. Security Groups

| Feature | Security Group (SG) | Network ACL (NACL) |
| :--- | :--- | :--- |
| **Level** | Instance (ENI) | Subnet |
| **State** | Stateful (Return traffic allowed auto) | Stateless (Must explicitly allow return) |
| **Rules** | Allow Only (Implicit Deny) | Allow and Deny |
| **Order** | No order (All rules evaluated) | Numbered order (Low to High) |

### Best Practice
*   Use **SGs** for primary defense (Allow port 80 from LB).
*   Use **NACLs** for blocking specific bad IPs (Deny IP 1.2.3.4).

---

## 4. KMS (Key Management Service)
Encryption at rest is a checkbox. KMS manages the keys.

### Customer Managed Keys (CMK) vs. AWS Managed
*   **AWS Managed**: Free, easy, but you don't control the key policy.
*   **CMK**: You control rotation and who can use it. Costs $1/month.

### Key Rotation
*   **Automatic**: AWS rotates the backing key material every year. Old data can still be decrypted.
*   **Manual**: You create a new key and re-encrypt data (Hard).

---

## 5. CloudTrail & GuardDuty
*   **CloudTrail**: The "CCTV" of AWS.
    *   *Tip*: Enable **Log File Validation** so hackers can't delete logs to hide tracks.
    *   *Tip*: Send logs to a separate "Log Archive" account that no one has write access to.
*   **GuardDuty**: The "AI Security Guard".
    *   Detects: Port scanning, Crypto mining, Tor exit nodes.
    *   *Action*: Set up EventBridge to email you when GuardDuty finds High severity issues.

---

## 6. Interview Preparation

### Q1: What is the difference between a Bucket Policy and an IAM Policy?
*   **Answer**:
    *   **IAM Policy**: Attached to a *User/Role*. "What can I do?" (I can read Bucket A).
    *   **Bucket Policy**: Attached to the *Resource (S3)*. "Who can access me?" (User Bob can read me).
    *   **Evaluation**: AWS evaluates BOTH. You need permission in BOTH (or at least one Allow and no Denys, depending on account boundaries).

### Q2: How do you secure an S3 bucket?
*   **Answer**:
    1.  **Block Public Access**: Enable the account-level block.
    2.  **Encryption**: Enable SSE-S3 or SSE-KMS.
    3.  **Versioning**: Enable it to recover from accidental/malicious deletion.
    4.  **Logging**: Enable Server Access Logging.
    5.  **Policy**: Enforce HTTPS-only access via Bucket Policy (`aws:SecureTransport`).

### Q3: You found an EC2 instance mining bitcoin. What do you do?
*   **Answer**:
    1.  **Isolate**: Change Security Group to deny all Outbound traffic.
    2.  **Snapshot**: Take a snapshot for forensics.
    3.  **Memory**: If possible, capture RAM.
    4.  **Tag**: Tag it as "Compromised".
    5.  **Terminate**: Kill it.
    6.  **Root Cause**: Check CloudTrail to see *who* launched it or *how* they got in (SSH keys? RCE vulnerability?).

---

## 7. Hands-On Lab
1.  **IAM**: Create a Policy that allows `s3:ListBucket` but DENIES `s3:GetObject`. Attach to a user and test.
2.  **CLI**: Install AWS CLI. Run `aws sts get-caller-identity`.
3.  **VPC**: Create a VPC with a Public and Private subnet. Launch an EC2 in Private. Try to SSH to it directly (Fail). Launch a Bastion in Public. SSH to Bastion -> SSH to Private (Success).
4.  **KMS**: Create a CMK. Encrypt an S3 bucket with it.
