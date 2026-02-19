Excellent progress, Pavan 👏
By finishing Weeks 1 & 2, you now cover **container + Kubernetes security** — already ahead of most DevOps engineers.

Now Week 3 focuses on:

✅ **Shift-Left Security**
✅ securing CI/CD pipelines
✅ preventing secrets leaks
✅ application vulnerability scanning
✅ software supply chain protection

This is **highly valued in DevSecOps interviews**.

---

# 🥉 WEEK 3: CI/CD Security & Shift-Left DevSecOps

## 🎯 Week Goal

By the end of this week you will:

✅ detect secrets before code is pushed
✅ scan code for vulnerabilities (SAST)
✅ scan applications for runtime vulnerabilities (DAST)
✅ secure dependencies & software supply chain
✅ enforce security gates in pipelines

---

# 🔹 Day 1 — Shift-Left Security Basics

## ❓ What is Shift-Left Security?

👉 Move security **earlier** in development lifecycle.

### Traditional (bad):

Code → Build → Deploy → Security check ❌

### DevSecOps (good):

Code → Security scan → Build → Security scan → Deploy ✔

---

## 🔐 Why this matters?

Fixing vulnerabilities:

* During coding → ₹500 cost
* After production → ₹50,000+ cost

👉 Companies want early detection.

---

# 🔹 Day 2 — Detect Secrets in Code Repositories

## Install Gitleaks

Detects:
✔ AWS keys
✔ tokens
✔ passwords
✔ private keys

### Install:

```bash
brew install gitleaks   # mac
```

or

```bash
sudo apt install gitleaks
```

---

### Scan repo:

```bash
gitleaks detect
```

---

## Example Secret Leak

```python
AWS_SECRET_ACCESS_KEY=abc123secret
```

👉 Tool flags it immediately.

---

## Integrate in CI/CD

```yaml
- name: Scan secrets
  run: gitleaks detect
```

---

## 🎯 Interview Tip:

Hardcoded secrets are one of the **most common security breaches**.

---

# 🔹 Day 3 — SAST (Static Application Security Testing)

## Use SonarQube

SAST scans source code for:
✔ vulnerabilities
✔ insecure coding patterns
✔ injection risks
✔ code quality issues

You already used SonarQube — now highlight **security rules**.

---

### Example vulnerabilities detected:

❌ SQL Injection
❌ Hardcoded passwords
❌ unsafe deserialization
❌ insecure API usage

---

### Enable security scanning:

In SonarQube → enable security profiles & rules.

---

## Pipeline Integration

```groovy
stage('Code Security Scan') {
  steps {
    sh 'mvn sonar:sonar'
  }
}
```

---

# 🔹 Day 4 — DAST (Dynamic Application Security Testing)

## Install OWASP ZAP

DAST scans running applications.

### Detects:

✔ XSS
✔ SQL injection
✔ broken authentication
✔ insecure headers

---

## Run basic scan (Docker)

```bash
docker run -t owasp/zap2docker-stable zap-baseline.py -t http://your-app-url
```

---

## Sample Scan Output

![Image](https://devopedia.org/images/article/72/2450.1523863706.jpg)

![Image](https://developer.harness.io/assets/images/security-tests-tab-b44d37913e046a02ca990114bf9c6c58.png)

---

# 🔹 Day 5 — Dependency & Supply Chain Security

Modern attacks target dependencies.

Example:
👉 malicious npm package
👉 vulnerable log4j library

---

## Use Trivy (filesystem scan)

```bash
trivy fs .
```

Scans:
✔ dependencies
✔ configuration files
✔ IaC issues

---

## Alternative tools (optional)

* Snyk
* Dependabot
* OWASP Dependency Check

---

# 🔹 Day 6 — Secure CI/CD Pipeline Design

## Add security stages:

✔ secret scanning
✔ SAST
✔ dependency scanning
✔ container scanning
✔ policy enforcement

---

## Secure Pipeline Flow

Code Commit
↓
Secrets Scan
↓
SAST Scan
↓
Build
↓
Dependency Scan
↓
Container Scan
↓
Deploy

👉 Deployment blocked if HIGH/CRITICAL vulnerabilities found.

---

# 🔹 Day 7 — Pipeline Security Best Practices

## ✅ Protect CI/CD systems

✔ restrict pipeline permissions
✔ store secrets securely (Vault later)
✔ enable audit logs
✔ use signed container images
✔ enforce branch protection
✔ implement approval gates

---

# 🎯 Interview Questions You Can Now Answer

### ❓ What is SAST vs DAST?

**SAST**

* scans source code
* early detection

**DAST**

* scans running app
* real attack simulation

---

### ❓ How do you prevent secrets leakage?

✔ Gitleaks scanning
✔ Vault secrets management
✔ avoid hardcoding

---

### ❓ What is software supply chain security?

Protect dependencies & packages from vulnerabilities and tampering.

---

### ❓ How do you secure CI/CD pipelines?

✔ security scans
✔ access control
✔ secrets management
✔ signed artifacts

---

# 🧠 Real DevSecOps Pipeline

Developer pushes code →
Gitleaks scans →
SonarQube SAST →
dependency scan →
build image →
Trivy scan →
OWASP ZAP test →
deploy only if secure

---

# ✅ End of Week 3 Skills

✔ secrets detection automation
✔ SAST & secure coding checks
✔ DAST vulnerability scanning
✔ dependency & supply chain security
✔ secure pipeline architecture

---

## ⭐ Resume Impact After Week 3

You can add:

✔ Implemented shift-left security using SAST, DAST & secrets scanning
✔ Integrated Gitleaks & SonarQube security rules in CI/CD pipelines
✔ Performed dynamic application security testing using OWASP ZAP
✔ Implemented supply chain security scanning for dependencies

🔥 This signals **advanced DevSecOps maturity**.

---

## Next Step Options

Ready for final step:

🏅 Week 4 → Secrets Management with Vault (enterprise-level skill)
🎯 End-to-end DevSecOps project (resume ready)
🎤 DevSecOps interview Q&A
🧠 Zero Trust & advanced security concepts
💰 How to present DevSecOps experience to get higher salary

Just tell me 👍
