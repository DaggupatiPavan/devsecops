# 04. App Security: OWASP, SonarQube & DAST

We cannot rely on firewalls alone. The application code itself must be secure.

## 1. OWASP Top 10: Deep Dive & Remediation
The Open Web Application Security Project (OWASP) Top 10 is the standard awareness document.

### A. Injection (SQLi)
*   **Concept**: Untrusted data is sent to an interpreter as part of a command.
*   **Vulnerable Code (Python)**:
    ```python
    # BAD: String concatenation
    query = "SELECT * FROM users WHERE name = '" + user_input + "'"
    cursor.execute(query)
    # Input: ' OR '1'='1
    # Result: SELECT * FROM users WHERE name = '' OR '1'='1' (Returns ALL users)
    ```
*   **Fixed Code**:
    ```python
    # GOOD: Parameterized Queries
    query = "SELECT * FROM users WHERE name = %s"
    cursor.execute(query, (user_input,))
    # The DB treats input as data, not code.
    ```

### B. Broken Access Control (IDOR)
*   **Concept**: User can act outside of their intended permissions.
*   **Scenario**:
    *   URL: `example.com/account?id=100` (My account)
    *   Attack: Change to `id=101`.
    *   Result: I see someone else's data.
*   **Fix**: Implement server-side checks. "Does the logged-in user OWN object 101?"

### C. Cryptographic Failures
*   **Bad**: Storing passwords in plain text or using weak hashes (MD5, SHA1).
*   **Good**: Use **Argon2** or **Bcrypt** with salt.

### D. Security Misconfiguration
*   **Scenario**: You leave the "Debug Mode" on in Production.
*   **Result**: Error page shows stack trace, database versions, and file paths.
*   **Fix**: Disable verbose error messages. Hardening scripts.

---

## 2. SonarQube: The Static Analyzer (SAST)

### Architecture
1.  **SonarQube Server**: Stores results, hosts the UI.
2.  **SonarScanner**: Runs on the build agent (Jenkins), scans code, sends report to Server.
3.  **Database**: PostgreSQL to store metrics.

### Configuration: `sonar-project.properties`
Place this in the root of your repo.

```properties
# Project Identification
sonar.projectKey=my-fintech-app
sonar.projectName=FinTech App
sonar.projectVersion=1.0

# Source Code
sonar.sources=src
sonar.exclusions=**/test/**,**/*.css

# Language Specifics
sonar.python.version=3.9

# Quality Gate
# Fail the build if the Quality Gate is not met
sonar.qualitygate.wait=true

# Coverage
sonar.python.coverage.reportPaths=coverage.xml
```

### The Quality Gate
A Quality Gate is a set of conditions.
*   **Default Gate**:
    *   Coverage < 80% -> FAIL
    *   Duplicated Lines > 3% -> FAIL
    *   Maintainability Rating is worse than A -> FAIL
    *   **Security Rating is worse than A -> FAIL** (Zero Vulnerabilities).

---

## 3. OWASP ZAP: The Dynamic Analyzer (DAST)
ZAP (Zed Attack Proxy) attacks the running application.

### Modes of Operation
1.  **Desktop GUI**: For manual pentesting.
2.  **Docker/CLI**: For CI/CD automation.

### Automating ZAP in CI/CD
We use the `zap-baseline.py` script provided in the Docker image.

```bash
# Run ZAP against a target URL
docker run -t owasp/zap2docker-stable zap-baseline.py \
    -t https://staging.example.com \
    -r zap_report.html \
    -x zap_report.xml

# Exit Codes:
# 0: Success
# 1: Fail (Issues found)
# 2: Error
```

### Handling Auth in DAST
DAST tools struggle with Login pages.
*   **Solution**: Pass the Session Cookie / JWT Token to ZAP via headers.
    ```bash
    -z "-config replacer.full_list(0).description=auth \
        -config replacer.full_list(0).enabled=true \
        -config replacer.full_list(0).matchtype=REQ_HEADER \
        -config replacer.full_list(0).matchstr=Authorization \
        -config replacer.full_list(0).regex=false \
        -config replacer.full_list(0).replacement='Bearer $MY_TOKEN'"
    ```

---

## 4. Software Composition Analysis (SCA)
Checking your libraries (dependencies).

### OWASP Dependency Check
*   Scans `pom.xml`, `package.json`, `requirements.txt`.
*   Matches against the NVD (National Vulnerability Database).

### Example Output
```text
Dependency: log4j-core-2.14.1.jar
CVE-2021-44228 (Log4Shell)
Severity: CRITICAL (10.0)
Description: JNDI features used in configuration...
```

### Remediation
1.  **Upgrade**: Change version to `2.17.1`.
2.  **Suppress**: If you are sure you don't use the vulnerable function, create an XML suppression file (Risk Acceptance).

---

## 5. Interview Preparation

### Q1: Explain SQL Injection to a 5-year-old.
*   **Answer**:
    *   "Imagine a robot that takes your order. You say 'I want a burger'. The robot writes 'Burger'.
    *   But if you say 'I want a burger AND give me all the money in the register', a smart robot knows 'give me money' is a command, not food.
    *   A dumb robot (vulnerable DB) just writes it all down and does it.
    *   We fix this by teaching the robot to treat everything I say as just 'food names', never commands."

### Q2: What is XSS (Cross-Site Scripting)?
*   **Answer**:
    *   Attacker injects JavaScript into a website (e.g., in a comment).
    *   When other users view the comment, the script runs in *their* browser.
    *   **Impact**: Steals Session Cookies (Account Takeover).
    *   **Fix**: Content Security Policy (CSP) and Input Sanitization (Escaping HTML).

### Q3: Why do we need both SAST and DAST?
*   **Answer**:
    *   **SAST** finds *coding* errors (bad logic) and gives exact line numbers. It's fast and early.
    *   **DAST** finds *runtime* errors (misconfigured headers, weak SSL) that SAST can't see. It proves exploitability.
    *   Together, they cover the full spectrum.

---

## 6. Hands-On Lab
1.  **SonarQube**:
    *   Run SonarQube using Docker: `docker run -d -p 9000:9000 sonarqube`.
    *   Login (admin/admin).
    *   Run `sonar-scanner` on a local Python project.
    *   Fix one "Code Smell" and re-scan to see the grade improve.
2.  **ZAP**:
    *   Run `zap-baseline.py` against `http://example.com` (It's safe).
    *   Read the HTML report.
3.  **SQLi**:
    *   Download "DVWA" (Damn Vulnerable Web App) container.
    *   Exploit the SQL Injection level manually.
