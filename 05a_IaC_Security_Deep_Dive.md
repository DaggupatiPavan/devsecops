# 05a. IaC Security Deep Dive: Locking, Checkov & Vault

Infrastructure as Code security is about protecting the "Blueprint" of your infrastructure.

## 1. Terraform State Locking (DynamoDB)
**Problem**: Two developers run `terraform apply` at the same time.
*   Dev A wants to create an EC2.
*   Dev B wants to delete the VPC.
*   **Result**: The state file gets corrupted. The cloud is in an unknown state.

**Solution**: Locking.
When Dev A runs `apply`, Terraform writes a "Lock ID" to a DynamoDB table. If Dev B tries to run, Terraform sees the lock and says "Error: State locked by Dev A".

### Configuration (`backend.tf`)
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"  # <--- THIS IS THE KEY
    encrypt        = true
  }
}
```
*   **Requirement**: You must create the DynamoDB table `terraform-locks` with a Partition Key named `LockID` (String).

---

## 2. Checkov Custom Policies
**Problem**: Checkov checks for generic issues (e.g., "S3 bucket public"). But you have company-specific rules (e.g., "All EC2 instances must have tag `CostCenter`").
**Solution**: Custom Python Policies.

### The Python Script (`custom_check.py`)
```python
from checkov.common.models.enums import CheckResult, CheckCategories
from checkov.terraform.checks.resource.base_resource_check import BaseResourceCheck

class EC2CostCenterTag(BaseResourceCheck):
    def __init__(self):
        name = "Ensure EC2 has CostCenter tag"
        id = "CKV_MY_1"
        supported_resources = ['aws_instance']
        categories = [CheckCategories.GENERAL_SECURITY]
        super().__init__(name=name, id=id, categories=categories, supported_resources=supported_resources)

    def scan_resource_conf(self, conf):
        # conf['tags'] is a list of dictionaries
        if 'tags' in conf and isinstance(conf['tags'], list):
            tags = conf['tags'][0]
            if 'CostCenter' in tags:
                return CheckResult.PASSED
        return CheckResult.FAILED

check = EC2CostCenterTag()
```

### Running It
```bash
checkov -d . --external-checks-dir ./my-checks
```

---

## 3. Ansible Vault
**Problem**: You need to deploy a database password, but you can't put it in `vars.yml` because that goes to Git.
**Solution**: Encrypt the variable.

### Method A: Encrypt the Whole File
```bash
ansible-vault create secrets.yml
# Type password: ****
# (Opens vim. Type your secrets.)
```
To run:
```bash
ansible-playbook site.yml --ask-vault-pass
```

### Method B: Encrypt Single String (Better)
You want `vars.yml` to be readable, except for the password.
```bash
ansible-vault encrypt_string 'SuperSecretPassword' --name 'db_password'
```
**Output**:
```yaml
db_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          343536373839...
```
Copy that block into your `vars.yml`. Ansible will decrypt it on the fly during the playbook run.
