# 06. CI/CD Security: Jenkins, Trivy & Supply Chain

The CI/CD pipeline is the "Factory Floor" of software. If the factory is compromised, every product leaving it is poisoned.

## 1. Jenkins Security: Hardening the Heart

### A. The "Admin" Problem
*   **Default**: Jenkins often installs with "Anyone can do anything".
*   **Fix**: Install **Role-Based Strategy Plugin**.
    *   **Global Roles**:
        *   `admin`: Full control (Only 2-3 people).
        *   `read-only`: Can view builds, cannot run them.
    *   **Project Roles**:
        *   `payment-team`: Can build/cancel jobs matching `payment-service-*`.

### B. Agent Isolation (The Sandbox)
*   **Risk**: If a build runs on the Master node and a malicious script runs `rm -rf /`, it kills Jenkins.
*   **Fix**: **NEVER build on Master**.
    *   Use **Docker Agents**. Every build spins up a fresh container and destroys it afterwards.
    *   *Jenkinsfile*:
        ```groovy
        pipeline {
            agent {
                docker { image 'maven:3.8-alpine' }
            }
            stages { ... }
        }
        ```

### C. Credentials Management
*   **Bad**: `sh 'docker login -u user -p password'` (Visible in process list).
*   **Good**: Use `withCredentials`.
    ```groovy
    withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
        sh 'echo $PASS | docker login -u $USER --password-stdin'
    }
    ```
    *   Jenkins automatically masks `$PASS` in logs as `****`.

---

## 2. Trivy: Advanced Scanning
Trivy is the industry standard because it is fast, stateless, and comprehensive.

### A. Scanning Modes
1.  **Image Mode**: `trivy image python:3.9`
2.  **Filesystem Mode**: `trivy fs .` (Scans `requirements.txt`, `package.json`).
3.  **Repo Mode**: `trivy repo https://github.com/foo/bar`.
4.  **K8s Mode**: `trivy k8s --report summary cluster`.

### B. Advanced Flags
*   `--ignore-unfixed`: Don't fail the build for bugs that have no patch yet. (Reduces noise).
*   `--severity HIGH,CRITICAL`: Ignore Low/Medium.
*   `--exit-code 1`: Return 1 if vulnerabilities are found (breaks Jenkins).
*   `--format json`: Output for parsing.

### C. The .trivyignore file
Sometimes you accept a risk.
*   Create `.trivyignore` in repo root:
    ```text
    CVE-2023-1234  # Accepted risk: We don't use the vulnerable function
    CVE-2023-5678  # False positive
    ```

---

## 3. The Secure Pipeline (Jenkinsfile)
A complete DevSecOps pipeline example.

```groovy
pipeline {
    agent any
    environment {
        SCANNER_HOME = tool 'SonarQubeScanner'
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Secret Scan') {
            steps {
                // Fail if secrets found
                sh 'trufflehog filesystem . --fail'
            }
        }

        stage('Build & Test') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('SAST (SonarQube)') {
            steps {
                withSonarQubeEnv('SonarServer') {
                    sh "${SCANNER_HOME}/bin/sonar-scanner"
                }
                // Wait for Quality Gate
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('SCA (Dependency Check)') {
            steps {
                dependencyCheck additionalArguments: '--format HTML', odzId: 'dep-check'
            }
        }

        stage('Build Image') {
            steps {
                sh 'docker build -t myapp:${BUILD_NUMBER} .'
            }
        }

        stage('Container Scan (Trivy)') {
            steps {
                // Break build on CRITICAL only
                sh 'trivy image --exit-code 1 --severity CRITICAL --ignore-unfixed myapp:${BUILD_NUMBER}'
            }
        }

        stage('Deploy to Dev') {
            steps {
                sh 'kubectl apply -f k8s/deployment.yaml'
            }
        }

        stage('DAST (ZAP)') {
            steps {
                // Run ZAP against the Dev environment
                sh 'docker run -t owasp/zap2docker-stable zap-baseline.py -t http://dev.myapp.internal'
            }
        }
    }
}
```

---

## 4. Supply Chain Security (SLSA)
**SolarWinds** changed everything. Hackers didn't attack the code; they attacked the *build system*.

### SLSA (Supply-chain Levels for Software Artifacts)
*   **Level 1**: Scripted Build. (You have a Jenkinsfile).
*   **Level 2**: Provenance. (The build server signs a metadata file saying "I built this binary from Commit X").
*   **Level 3**: Ephemeral Environment. (Build runs in a fresh container, not a shared server).
*   **Level 4**: Hermetic. (No internet access during build).

### Generating Provenance
Tools like **Cosign** (by Sigstore) can sign images and generate provenance.
```bash
cosign sign --key cosign.key myapp:v1
```

---

## 5. Interview Preparation

### Q1: How do you secure Jenkins?
*   **Answer**:
    1.  **Network**: Put it in a private subnet, behind VPN.
    2.  **Auth**: Integrate with LDAP/AD/SSO. Disable "Sign Up".
    3.  **Authorization**: Use Matrix/Role-Based strategy.
    4.  **Agents**: Never build on Master. Use ephemeral Docker agents.
    5.  **Secrets**: Use Credentials plugin, never env vars in shell history.

### Q2: What is "Golden Pipeline"?
*   **Answer**:
    *   "A standardized, approved pipeline template that all teams must use.
    *   It has security checks (SAST, SCA, DAST) built-in.
    *   Developers just reference it (e.g., `@Library('security-pipeline')`).
    *   This ensures no one bypasses security."

### Q3: How do you handle a False Positive in the pipeline?
*   **Answer**:
    *   "I verify it manually.
    *   If it is indeed false, I add it to the suppression file (e.g., `.trivyignore` or `sonar-project.properties`).
    *   I document *why* it is false.
    *   I never turn off the scanner."

---

## 6. Hands-On Lab
1.  **Jenkins**: Run Jenkins in Docker.
2.  **Pipeline**: Create a "Hello World" pipeline.
3.  **Trivy**: Add a stage to install Trivy and scan an image.
4.  **Fail it**: Scan an old image (e.g., `python:3.4`) and watch the pipeline fail.
5.  **Fix it**: Change to `python:3.9-slim` and watch it pass.
