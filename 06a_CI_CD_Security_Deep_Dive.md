# 06a. CI/CD Security Deep Dive: Shared Libs, Air-Gap & SLSA

Advanced CI/CD security is about scale and resilience.

## 1. Jenkins Shared Libraries (Scale Security)
**Problem**: You have 50 microservices. Copy-pasting the `Jenkinsfile` 50 times is a nightmare. If you need to add a new security scan, you have to edit 50 files.
**Solution**: Shared Libraries. Define the logic ONCE, use it everywhere.

### The Library Structure (Git Repo)
Create a repo `jenkins-shared-lib`.
```text
vars/
  securePipeline.groovy
```

### The Code (`securePipeline.groovy`)
```groovy
def call(Map config) {
    pipeline {
        agent any
        stages {
            stage('Checkout') { steps { checkout scm } }
            
            stage('Security Scan') {
                steps {
                    // This runs for EVERYONE. They cannot skip it.
                    sh 'trivy fs .'
                    sh 'mvn sonar:sonar'
                }
            }
            
            stage('Build') {
                steps {
                    // Run the team's specific build command
                    sh config.buildCommand
                }
            }
        }
    }
}
```

### The Usage (In the Microservice Repo)
The developer's `Jenkinsfile` becomes tiny:
```groovy
@Library('jenkins-shared-lib') _

securePipeline(buildCommand: 'mvn clean package')
```
**Result**: You control the pipeline. You can inject new security checks instantly for all 50 teams.

---

## 2. Trivy in Air-Gapped Environments
**Problem**: Your bank's server has NO internet. `trivy image python:3.9` fails because it cannot download the vulnerability DB.
**Solution**: Download the DB offline and transfer it.

### Step 1: Download DB (On Internet Machine)
```bash
# Download the DB files
oras pull ghcr.io/aquasecurity/trivy-db:2
oras pull ghcr.io/aquasecurity/trivy-java-db:1
```

### Step 2: Transfer
Copy the files to the air-gapped server (USB stick, Secure Copy, etc.).

### Step 3: Run Trivy (Offline)
```bash
trivy image \
  --offline-scan \
  --skip-db-update \
  --cache-dir /path/to/local/cache \
  python:3.9
```

---

## 3. SLSA (Supply-chain Levels for Software Artifacts)
**Concept**: How "safe" is your build process?

### Level 1: Scripted Build
*   You are not building manually on a laptop.
*   You have a `Jenkinsfile` or `Makefile`.

### Level 2: Provenance (Signed Metadata)
*   The build server says: "I built `app.jar` (Hash: abc123...) from Git Commit `f4a32...` at 10:00 AM."
*   This metadata is **signed**.

### Level 3: Ephemeral Environment
*   The build runs in a fresh container.
*   It does NOT run on a shared server where a previous build could have left a virus in `/tmp`.

### Level 4: Hermetic (The Holy Grail)
*   **No Internet**: The build container has `network: none`.
*   **Explicit Deps**: ALL dependencies (Maven jars, NPM packages) must be pre-fetched or available in a local mirror.
*   **Why?**: If the build has no internet, it cannot download a malicious script from `evil.com` during the build.
