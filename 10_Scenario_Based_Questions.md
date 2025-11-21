# 10. DevSecOps Scenario-Based Interview Questions (The Ultimate List)

This document contains **140 Real-World Scenarios** (20 per topic) to test your DevSecOps depth.

---

## 1. Linux Security Scenarios (20 Questions)

1.  **The "Unkillable" Process**: Process respawns after `kill -9`. **Fix**: Check parent (`ps -o ppid=`), cron jobs, or systemd services.
2.  **The "Permission Denied" Root**: Root cannot edit file. **Fix**: Check immutable bit (`lsattr`). Remove with `chattr -i`.
3.  **The "Invisible" File**: Disk full, but `du` shows space. **Fix**: Open files deleted (`lsof | grep deleted`). Restart process.
4.  **SSH "Too Many Failures"**: Locked out before password. **Fix**: Too many keys in agent. Use `IdentitiesOnly=yes`.
5.  **The "Suicide" Script**: `/bin/chmod` deleted. **Fix**: Use dynamic linker `/lib64/ld-linux... /bin/cp` or Perl/Python.
6.  **Cron Job Fails**: Script runs manually but fails in Cron. **Fix**: Cron has minimal ENV. Define full paths (`/usr/bin/java`) and variables.
7.  **SSH Key Permissions**: "WARNING: UNPROTECTED PRIVATE KEY FILE". **Fix**: `chmod 600 my-key.pem`.
8.  **Sudo Timeout**: You want to run sudo without password for 1 command only. **Fix**: `sudo -S` or configure `sudoers` with `NOPASSWD` (Risky).
9.  **Chroot Breakout**: How to escape a chroot? **Fix**: If root inside chroot, can `mknod` raw disk device or use `chroot` inside `chroot`.
10. **Tcpdump Traffic**: Capture only HTTP traffic to IP X. **Fix**: `tcpdump -i eth0 host 1.2.3.4 and port 80 -w capture.pcap`.
11. **Netstat vs SS**: `netstat` is slow. **Fix**: Use `ss -tuln` (Socket Statistics) which reads from kernel directly.
12. **Port Conflict**: Port 8080 in use. **Fix**: `lsof -i :8080` or `ss -lptn 'sport = :8080'` to find PID.
13. **Grep Secrets**: Find "password" in all files recursively. **Fix**: `grep -r "password" /path`. Better: `grep -rE "password|secret"`.
14. **Find SetUID**: Find all files that run as root when executed. **Fix**: `find / -perm -4000 -user root`.
15. **Umask Risk**: New files are created `777`. **Fix**: Check `umask`. Default should be `022` (755 folders, 644 files).
16. **Lock User**: Stop user 'bob' from logging in without deleting. **Fix**: `passwd -l bob` or `usermod -L bob`.
17. **Check Reboots**: Who rebooted the server? **Fix**: `last -x | grep shutdown` or check `/var/log/wtmp`.
18. **History Timestamp**: `history` shows commands but not *when*. **Fix**: Set `HISTTIMEFORMAT="%F %T "`.
19. **Proc Filesystem**: What is `/proc`? **Answer**: Virtual filesystem. `/proc/cpuinfo` (CPU), `/proc/meminfo` (RAM).
20. **Kernel Modules**: Check if a rootkit module is loaded. **Fix**: `lsmod`. Remove with `rmmod` (if not hidden).
21. **The Invisible Firewall**: UFW/Firewalld off, but port blocked. **Answer**: Check raw `iptables -L`. UFW is just a frontend; rules can exist without it. Also check Cloud Security Groups.

---

## 2. Git & Version Control Scenarios (20 Questions)

1.  **The "Dangling" Commit**: Lost commit after reset. **Fix**: `git reflog` -> `git reset --hard <SHA>`.
2.  **The "Secret" in History**: Password committed then deleted. **Fix**: `git-filter-repo` or BFG. Force push.
3.  **Merge Conflict JSON**: Conflict in `package-lock.json`. **Fix**: Accept ours/theirs, then `npm install` to regenerate.
4.  **Detached HEAD**: Commits lost after checkout. **Fix**: `git reflog` to find SHA, then branch off it.
5.  **Unverified GPG**: GitHub says Unverified. **Fix**: Email in GPG key must match `git config user.email`.
6.  **Git Bisect**: Find which commit introduced a bug. **Fix**: `git bisect start` -> `bad` -> `good`. Git binary searches.
7.  **Stash Pop Conflict**: Conflict when popping. **Fix**: Resolve conflict manually, then `git stash drop`.
8.  **Cherry Pick**: Apply specific commit to another branch. **Fix**: `git cherry-pick <SHA>`.
9.  **Revert vs Reset**: Public branch mistake. **Fix**: Use `git revert` (adds new commit). Never `reset` public history.
10. **Submodule Update**: Submodule folder is empty. **Fix**: `git submodule update --init --recursive`.
11. **Git LFS Lock**: Binary file locked. **Fix**: `git lfs lock <file>` / `unlock`.
12. **Blame Whitespace**: `git blame` shows you because you formatted code. **Fix**: `git blame -w`.
13. **Local vs Global**: Different email for work repo. **Fix**: `git config --local user.email "work@corp.com"`.
14. **Remote Prune**: Too many deleted branches in list. **Fix**: `git remote prune origin`.
15. **Tag Signing**: Sign a release tag. **Fix**: `git tag -s v1.0 -m "Release"`.
16. **Interactive Rebase**: Squash 5 commits into 1. **Fix**: `git rebase -i HEAD~5`. Change `pick` to `squash`.
17. **Clean Untracked**: Remove all new files. **Fix**: `git clean -fd` (Force Directory).
18. **Diff Staged**: View changes ready to commit. **Fix**: `git diff --staged`.
19. **Log Graph**: Visualize history. **Fix**: `git log --graph --oneline --all`.
20. **Bundle Transfer**: Git without network. **Fix**: `git bundle create repo.bundle --all`. Transfer file. `git clone repo.bundle`.

---

## 3. Container Security Scenarios (20 Questions)

1.  **Root Breakout**: Container modified host files. **Fix**: Block `hostPath` mounts, enforce `runAsNonRoot`.
2.  **CrashLoopBackOff**: Pod dies instantly. **Fix**: `kubectl logs -p`. Change cmd to `sleep 3600` to debug.
3.  **ImagePullBackOff**: Cannot pull image. **Fix**: Check `imagePullSecret` and Network/DNS.
4.  **OOMKilled**: Java app dies. **Fix**: Set JVM Heap (`-Xmx`) lower than Container Memory Limit.
5.  **Zombie Pods**: Stuck Terminating. **Fix**: `kubectl delete pod --force --grace-period=0`. Check Finalizers.
6.  **Docker Socket**: Container has `/var/run/docker.sock`. **Risk**: Full root access to host. **Fix**: Remove it.
7.  **Docker History**: Secrets in `docker history`. **Fix**: Use Multi-stage builds. Secrets don't persist in final image.
8.  **Multi-Stage**: Why use it? **Answer**: Build in `golang:alpine` (large), copy binary to `scratch` (tiny/secure).
9.  **USER Instruction**: Dockerfile missing `USER`. **Fix**: Add `USER 1000` before `CMD`.
10. **ReadOnly Root**: Malware tries to download file. **Fix**: `readOnlyRootFilesystem: true`.
11. **Cap Drop**: Reduce attack surface. **Fix**: `securityContext: capabilities: drop: ["ALL"]`.
12. **Seccomp**: Block syscalls (like `chmod`). **Fix**: Apply Seccomp profile in SecurityContext.
13. **AppArmor**: Restrict file access. **Fix**: Annotation `container.apparmor.security.beta.kubernetes.io/pod: localhost/profile`.
14. **K8s Secrets**: Are they encrypted? **Answer**: Base64 encoded by default (NOT encrypted). Enable Encryption-at-Rest (KMS).
15. **Image Pull Secrets**: Private registry fails. **Fix**: Create secret `kubectl create secret docker-registry`.
16. **Liveness vs Readiness**: App restarts loop. **Fix**: Liveness kills it. Readiness stops traffic. Check probe logic.
17. **CPU Throttling**: App slow but not high CPU. **Fix**: CPU Limits cause throttling. Monitor `container_cpu_cfs_throttled_seconds`.
18. **Node Affinity**: Run secure pods on secure nodes. **Fix**: `nodeSelector` or `affinity`.
19. **Taints**: Prevent regular pods on Admin nodes. **Fix**: `kubectl taint nodes admin-node key=value:NoSchedule`.
20. **PSA**: Pod Security Admission. **Fix**: Label namespace `pod-security.kubernetes.io/enforce: restricted`.
21. **The Healthy Crash**: App logs fine, but restarts every 30s. **Answer**: **Liveness Probe** failure. The app is alive, but the health endpoint is failing or timing out.

---

## 4. App Security (OWASP) Scenarios (20 Questions)

1.  **Reflected XSS**: Alert box pops up. **Fix**: Context-aware encoding (HTML entities).
2.  **Blind SQLi**: App doesn't crash, but logic leaks. **Fix**: Parameterized Queries (Prepared Statements).
3.  **Missing HSTS**: Downgrade attack possible. **Fix**: Add header `Strict-Transport-Security: max-age=31536000`.
4.  **CSRF**: Attacker forces user to click button. **Fix**: Anti-CSRF Tokens (Synchronizer Token Pattern).
5.  **SSRF**: Webhook calls internal metadata IP. **Fix**: Allowlist URLs. Block internal IP ranges (169.254...).
6.  **IDOR**: User A accesses `/users/123` (User B). **Fix**: Check ownership on server side (`if user.id == resource.owner_id`).
7.  **JWT None Alg**: Token signature ignored. **Fix**: Enforce algorithm verification (HS256/RS256). Reject `none`.
8.  **XXE**: XML parser reads `/etc/passwd`. **Fix**: Disable DTDs (External Entities) in XML parser config.
9.  **Deserialization**: Python Pickle/Java Object run code. **Fix**: Don't deserialize untrusted data. Use JSON instead.
10. **Clickjacking**: Site loaded in iframe. **Fix**: `X-Frame-Options: DENY` or CSP `frame-ancestors`.
11. **CORS**: `Access-Control-Allow-Origin: *`. **Fix**: Specify exact domains. Don't use wildcard with credentials.
12. **CSP**: Script injection. **Fix**: `Content-Security-Policy: default-src 'self'`.
13. **HttpOnly Cookie**: XSS steals cookie. **Fix**: Set `HttpOnly` flag (JS cannot read it).
14. **Secure Cookie**: Cookie sent over HTTP. **Fix**: Set `Secure` flag (HTTPS only).
15. **Password Hashing**: DB leak reveals passwords. **Fix**: Use Argon2 or bcrypt. Never MD5/SHA1.
16. **Rate Limiting**: Brute force API. **Fix**: Implement Token Bucket or Leaky Bucket limiting (e.g., Nginx limit_req).
17. **Input Validation**: Malicious input. **Fix**: Allowlist (Accept known good) > Blocklist (Reject known bad).
18. **Error Handling**: Stack trace reveals DB version. **Fix**: Generic error pages. Log details internally.
19. **Logging Secrets**: Password in logs. **Fix**: Mask/Redact sensitive fields in logger config.
20. **Directory Traversal**: `../../etc/passwd`. **Fix**: Validate filenames. Don't allow path separators.

---

## 5. IaC Security (Terraform) Scenarios (20 Questions)

1.  **State File Leak**: `terraform.tfstate` in Git. **Fix**: Rotate secrets. Delete from history. Use Remote Backend.
2.  **Drift Detection**: Manual changes made. **Fix**: `terraform plan --refresh-only` to see drift.
3.  **Sensitive Output**: Password shown in console. **Fix**: `sensitive = true` in output definition.
4.  **Import Resource**: Existing S3 bucket needs management. **Fix**: `terraform import aws_s3_bucket.b bucket-name`.
5.  **Taint Resource**: Force recreate EC2. **Fix**: `terraform taint aws_instance.web` (or `-replace` in v1.0+).
6.  **Workspace Isolation**: Dev/Prod in same state? **Fix**: Use `terraform workspace` or separate folders/backends.
7.  **Provider Pinning**: Update broke build. **Fix**: Pin version `version = "~> 4.0"` in `required_providers`.
8.  **Ansible No Log**: Password printed in Ansible log. **Fix**: `no_log: true` on the task.
9.  **Ansible Become**: Run task as root. **Fix**: `become: yes`.
10. **Ansible Handlers**: Restart Nginx only if config changed. **Fix**: `notify: restart nginx` in task.
11. **Inventory Separation**: Dev accidentally deployed to Prod. **Fix**: Separate `dev.ini` and `prod.ini`.
12. **CloudFormation Drift**: Stack out of sync. **Fix**: "Detect Drift" in AWS Console.
13. **StackSets**: Deploy to 50 accounts. **Fix**: Use CloudFormation StackSets.
14. **Pulumi Secrets**: Secrets in code. **Fix**: `pulumi config set --secret dbPassword`.
15. **Hardcoded IPs**: IP changed, broke TF. **Fix**: Use Data Sources (`data "aws_instance" ...`) to fetch dynamic IPs.
16. **Security Group Loop**: 50 rules needed. **Fix**: Use `for_each` or `dynamic` block in Terraform.
17. **Wildcard Policy**: `Action: *`. **Fix**: Checkov/tfsec to block wildcards. Least Privilege.
18. **Module Versioning**: Module changed upstream. **Fix**: Source module from Git tag `ref=v1.0`.
19. **State Locking**: Two devs apply same time. **Fix**: DynamoDB table for locking.
20. **Encrypted State**: S3 bucket unencrypted. **Fix**: Enable SSE-S3 or KMS on the backend bucket.

---

## 6. CI/CD Security Scenarios (20 Questions)

1.  **Poisoned Pipeline**: PR steals secrets. **Fix**: Restrict PRs from forks. "Safe to Test" label.
2.  **Dependency Confusion**: Public pkg installed instead of private. **Fix**: Configure Nexus/Artifactory priority. Scoped packages.
3.  **Jenkins Plugin**: Vulnerable plugin. **Fix**: Update plugin. Use "Plugin Manager" to audit.
4.  **Agent Security**: Master node compromised. **Fix**: Don't build on Master. Use ephemeral Docker agents.
5.  **GitHub Secrets**: Fork can access secrets? **Answer**: No, forks generally don't have access to secrets unless passed explicitly.
6.  **Pull Request Target**: `pull_request_target` runs in context of base. **Risk**: Malicious code can steal secrets.
7.  **Shell vs Docker Runner**: Shell runner leaves junk. **Fix**: Use Docker runner for clean environment.
8.  **Artifact Signing**: Binary modified after build. **Fix**: Sign with Cosign/Sigstore. Verify before deploy.
9.  **Dependency Pinning**: `npm install` pulls new bad version. **Fix**: Use `package-lock.json` (`npm ci`).
10. **Pipeline Timeout**: Miner runs for 6 hours. **Fix**: Set `timeout(time: 30, unit: 'MINUTES')`.
11. **Approval Gates**: Prod deploy happens automatically. **Fix**: Manual approval step (`input` in Jenkins).
12. **Env Segregation**: Dev creds used in Prod. **Fix**: Separate credentials in CI (e.g., `PROD_AWS_KEY`).
13. **Secret Rotation**: Key leaked. **Fix**: Rotate in AWS, update in Jenkins Credentials.
14. **Build Logs**: `echo $PASSWORD`. **Fix**: Use `withCredentials` (Jenkins masks it). Grep logs for patterns.
15. **Cache Poisoning**: Shared cache infected. **Fix**: Clear cache regularly. Hash keys based on lockfiles.
16. **Typosquatting**: `react` vs `rreact`. **Fix**: SCA tools (Snyk/Trivy) warn about unknown/low-rep packages.
17. **Code Coverage**: Tests skipped. **Fix**: Enforce coverage threshold (e.g., 80%) to pass build.
18. **Static Analysis**: Dev ignores linting. **Fix**: Fail build on lint errors.
19. **Golden Image**: Patching takes too long. **Fix**: Build new AMI/Image with patches baked in (Packer).
20. **Pipeline As Code**: GUI config lost. **Fix**: Store ALL pipeline config in Git (`Jenkinsfile`, `.gitlab-ci.yml`).
21. **Trojan NPM Package**: Public package has higher version. **Answer**: Configure Nexus/Artifactory to **block** public lookup for internal package names. Use Scoped Packages (`@corp/lib`).

---

## 7. Cloud Security (AWS) Scenarios (20 Questions)

1.  **Open S3 Bucket**: Public data leak. **Fix**: "Block Public Access" at Account level.
2.  **Compromised EC2**: Mining bitcoin. **Fix**: Isolate SG (Deny All). Snapshot. Forensics.
3.  **IAM Role Assumption**: Who can assume Admin? **Fix**: Check Trust Policy (`Principal`).
4.  **Pre-signed URL**: Temporary access to private object. **Fix**: Generate URL with short expiration (5 mins).
5.  **Server Access Logging**: Who deleted the file? **Fix**: Enable S3 Server Access Logging or CloudTrail Data Events.
6.  **IMDSv2**: SSRF steals role keys. **Fix**: Enforce IMDSv2 (Session Token required). Disable IMDSv1.
7.  **RDS Encryption**: DB unencrypted. **Fix**: Snapshot -> Copy (Encrypt) -> Restore.
8.  **RDS Public**: Database has public IP. **Fix**: `PubliclyAccessible=false`. Put in Private Subnet.
9.  **VPC Peering**: Prod peered with Dev. **Risk**: Lateral movement. **Fix**: Use Transit Gateway with route tables or PrivateLink.
10. **Flow Logs**: Debug blocked traffic. **Fix**: Enable VPC Flow Logs. Check REJECT records.
11. **CloudTrail Integrity**: Hacker deletes logs. **Fix**: Enable Log File Validation. Send to separate Audit account.
12. **KMS Key Deletion**: Accidental delete. **Fix**: Schedule deletion (7-30 days). Cannot delete instantly.
13. **WAF Rules**: Block SQLi. **Fix**: Enable AWS Managed Rules (SQLi, XSS) on WAF.
14. **Shield**: DDoS attack. **Fix**: Shield Standard is on by default. Shield Advanced offers cost protection.
15. **Config Rules**: Compliance check. **Fix**: AWS Config rule "s3-bucket-public-read-prohibited".
16. **SCPs**: Stop regions. **Fix**: Org Service Control Policy (SCP) Deny all regions except `us-east-1`.
17. **Route53 DNSSEC**: DNS spoofing. **Fix**: Enable DNSSEC signing in Route53.
18. **EBS Encryption**: Disk unencrypted. **Fix**: Enable "Default Encryption" for EBS in region.
19. **Lambda Injection**: Code injection in Lambda. **Fix**: Input validation. Least Privilege IAM role.
20. **Secrets Manager**: Hardcoded DB pass. **Fix**: App retrieves pass from Secrets Manager at runtime.
