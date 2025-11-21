# 05. IaC Security: Terraform, Ansible & Checkov

Infrastructure as Code (IaC) allows us to provision 100 servers in seconds. It also allows us to create 100 security holes in seconds.

## 1. Terraform Security Best Practices

### A. State Management (The Holy Grail)
The `terraform.tfstate` file is dangerous.
1.  **Remote Backend**: ALWAYS use S3/AzureBlob. Never store state locally or in Git.
2.  **Encryption**:
    ```hcl
    terraform {
      backend "s3" {
        bucket         = "my-tf-state"
        key            = "prod/terraform.tfstate"
        region         = "us-east-1"
        encrypt        = true  # Server-Side Encryption
        dynamodb_table = "tf-locks"
      }
    }
    ```
3.  **Access Control**: The S3 bucket should have a Bucket Policy denying access to everyone except the CI/CD IAM Role.

### B. Secure Coding in HCL
*   **Avoid Hardcoding**: Use `variables.tf` and `terraform.tfvars`.
*   **Use Data Sources**: Fetch the latest AMI ID dynamically, don't hardcode old ones.

#### Example: Secure S3 Bucket
```hcl
resource "aws_s3_bucket" "secure_bucket" {
  bucket = "my-secure-data"
}

# 1. Block Public Access
resource "aws_s3_bucket_public_access_block" "block" {
  bucket = aws_s3_bucket.secure_bucket.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# 2. Enable Encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "enc" {
  bucket = aws_s3_bucket.secure_bucket.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# 3. Enable Versioning (Ransomware protection)
resource "aws_s3_bucket_versioning" "ver" {
  bucket = aws_s3_bucket.secure_bucket.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

---

## 2. Checkov: Policy as Code
We don't want to manually review 1,000 lines of Terraform. We use Checkov.

### Running Checkov
```bash
pip install checkov
checkov -d . --framework terraform
```

### Handling Failures
*   **Fix it**: Change the code.
*   **Skip it**: Sometimes you *need* a public bucket (e.g., for a static website).
    *   Add a comment to suppress the check:
    ```hcl
    # checkov:skip=CKV_AWS_20: "This bucket is for public website hosting"
    resource "aws_s3_bucket" "website" { ... }
    ```

### Custom Policies (Python)
You can write complex logic that YAML can't handle.
*   *Scenario*: "All EC2 instances must use an AMI that is less than 30 days old."

```python
from checkov.common.models.enums import CheckResult, CheckCategories
from checkov.terraform.checks.resource.base_resource_check import BaseResourceCheck

class AMIAgeCheck(BaseResourceCheck):
    def __init__(self):
        name = "Ensure AMI is not older than 30 days"
        id = "CKV_AWS_999"
        categories = [CheckCategories.GENERAL_SECURITY]
        super().__init__(name=name, id=id, categories=categories, supported_resources=["aws_instance"])

    def scan_resource_conf(self, conf):
        # Logic to fetch AMI creation date and compare
        # ...
        return CheckResult.PASSED

scanner = AMIAgeCheck()
```

---

## 3. Ansible Security

### Ansible Vault
Encrypting secrets within the repo.

1.  **Create Encrypted File**:
    ```bash
    ansible-vault create secrets.yml
    # Prompts for password
    ```
2.  **Use in Playbook**:
    ```yaml
    - hosts: all
      vars_files:
        - secrets.yml
      tasks:
        - name: Create DB User
          mysql_user:
            name: admin
            password: "{{ db_password }}" # From secrets.yml
    ```
3.  **Run**:
    ```bash
    ansible-playbook site.yml --ask-vault-pass
    # OR use a password file for CI/CD
    echo "my_password" > .vault_pass
    ansible-playbook site.yml --vault-password-file .vault_pass
    ```

### Hardening with Ansible
Don't run shell commands manually. Use a hardening role.
*   **DevSec Hardening Framework**: A popular open-source collection of hardening roles.
    ```yaml
    - hosts: servers
      roles:
        - dev-sec.os-hardening
        - dev-sec.ssh-hardening
    ```
    *   This automatically applies the CIS Benchmark settings we discussed in the Linux section.

---

## 4. Immutable Infrastructure & Drift
*   **Mutable (Ansible)**: You SSH into a server and update it.
    *   *Risk*: "Configuration Drift". Over years, the server becomes a "Snowflake" (Unique, fragile).
*   **Immutable (Terraform + Packer)**:
    *   You bake a Golden Image (AMI) with Packer.
    *   You deploy it with Terraform.
    *   If you need to update, you bake a *new* image and replace the server.
    *   *Security*: You know EXACTLY what is running. No lingering malware.

---

## 5. Interview Preparation

### Q1: How do you handle Terraform state locking?
*   **Answer**:
    *   "I use DynamoDB with S3 backend.
    *   When Terraform runs, it writes a Lock ID to DynamoDB.
    *   If another developer tries to run `terraform apply` at the same time, it sees the lock and fails.
    *   This prevents corrupting the state file."

### Q2: What is "Drift Detection"?
*   **Answer**:
    *   "Drift is when the *actual* infrastructure (AWS) differs from the *code* (Terraform).
    *   Example: Someone manually opened Port 22 in the AWS Console.
    *   I detect this by running `terraform plan` periodically (e.g., nightly) in Jenkins. If it shows changes, it means there is drift."

### Q3: Checkov vs. TFSec?
*   **Answer**:
    *   "Both are great.
    *   **Checkov** is broader (covers K8s, CloudFormation, ARM templates) and supports Python policies.
    *   **TFSec** is faster and specific to Terraform.
    *   I prefer Checkov for its graph-based scanning (it understands relationships between resources)."

---

## 6. Hands-On Lab
1.  **Terraform**: Write a `main.tf` that creates an S3 bucket.
2.  **Checkov**: Run `checkov -f main.tf`. It should scream about encryption and public access.
3.  **Fix**: Add the security resources. Run Checkov again until it passes.
4.  **Ansible Vault**: Create an encrypted file, view it (it's garbage text), edit it (it opens in vim), and run a playbook using it.
