# 04a. App Security Deep Dive: SQLi, SonarQube Gates & ZAP

Here is the deep dive into Application Security.

## 1. SQL Injection (The Fix)
**Problem**: `SELECT * FROM users WHERE name = '` + `userInput` + `'`
If user types `' OR 1=1 --`, the query becomes `SELECT * FROM users WHERE name = '' OR 1=1 --`. This returns ALL users.

### The Wrong Way (Sanitization)
Trying to remove quotes (`replace("'", "")`) is bad. Hackers will find a way around it.

### The Right Way (Parameterized Queries)
Treat input as *data*, not *code*.

**Java (JDBC)**:
```java
// BAD
String query = "SELECT * FROM users WHERE name = '" + userName + "'";

// GOOD
String query = "SELECT * FROM users WHERE name = ?";
PreparedStatement pstmt = connection.prepareStatement(query);
pstmt.setString(1, userName); // The DB driver handles the escaping safely.
ResultSet results = pstmt.executeQuery();
```

**Python (Psycopg2)**:
```python
# BAD
cursor.execute("SELECT * FROM users WHERE name = '%s'" % user_name)

# GOOD
cursor.execute("SELECT * FROM users WHERE name = %s", (user_name,))
```

---

## 2. SonarQube Quality Gates
**Problem**: Developers ignore warnings.
**Solution**: Break the build if the Quality Gate fails.

### Configuring the Gate
1.  **Login** to SonarQube as Admin.
2.  **Quality Gates** -> **Create**.
3.  **Add Conditions**:
    *   **Security Rating**: is worse than **A**. (Must be A).
    *   **Vulnerabilities**: is greater than **0**. (Zero tolerance).
    *   **Duplicated Lines**: is greater than **3%**.
    *   **Coverage**: is less than **80%**.
4.  **Enforce**: Set this as the "Default" gate or assign it to your project.

### Jenkins Pipeline Integration
```groovy
stage('SonarQube') {
    steps {
        withSonarQubeEnv('SonarServer') {
            sh 'mvn sonar:sonar'
        }
    }
}
stage('Quality Gate') {
    steps {
        timeout(time: 1, unit: 'HOURS') {
            // This pauses the pipeline until SonarQube sends the result back
            waitForQualityGate abortPipeline: true
        }
    }
}
```

---

## 3. OWASP ZAP Automation
**Problem**: ZAP GUI is for humans. Pipelines need CLI.
**Solution**: `zap-baseline.py` (Dockerized ZAP).

### The Command
```bash
docker run -t owasp/zap2docker-stable zap-baseline.py \
    -t https://www.example.com \
    -g gen.conf \
    -r report.html
```
*   `-t`: Target URL.
*   `-g`: Generate a config file (to ignore specific rules later).
*   `-r`: Output report.

### Handling Failures
By default, ZAP fails on *everything*. You need to tune it.
1.  Run it once to generate `gen.conf`.
2.  Edit `gen.conf` to change "FAIL" to "WARN" or "IGNORE" for false positives.
    ```text
    # Rule ID   Action  Description
    10021       WARN    X-Content-Type-Options Header Missing
    ```
3.  Run it again with the config.
    ```bash
    docker run -v $(pwd):/zap/wrk/:rw -t owasp/zap2docker-stable zap-baseline.py \
        -t https://www.example.com \
        -c gen.conf
    ```
