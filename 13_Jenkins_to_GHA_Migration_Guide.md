# 13. Jenkins to GitHub Actions Migration Guide (Senior/Principal Level)

**Target Audience**: Senior/Principal DevOps Engineers (8+ Years Exp).
**Goal**: To demonstrate strategic leadership, architectural depth, and operational excellence during a migration.

---

## 1. The Paradigm Shift (Executive Summary)

Migrating from Jenkins to GitHub Actions (GHA) is a shift from **CapEx to OpEx** and from **Centralized to Decentralized**.

| Feature | Jenkins (Legacy) | GitHub Actions (Modern) | Strategic Implication |
| :--- | :--- | :--- | :--- |
| **Model** | **Server-based**. Pet servers, maintenance heavy. | **Serverless / Runner-based**. On-demand, ephemeral. | Shift from "Maintaining CI Servers" to "Optimizing Developer Experience". |
| **Cost** | **Fixed (CapEx)**. You pay for EC2s even if idle. | **Usage-based (OpEx)**. Pay per minute. | **Cost Optimization** becomes a daily engineering concern. |
| **Governance** | **Centralized**. Admins control plugins. | **Decentralized**. Developers can use any Action. | **Supply Chain Security** policies are critical to prevent "Shadow IT". |
| **Scale** | **Vertical**. Add more RAM to Master. | **Horizontal**. Add more Runner Groups / ARC. | Infinite scale, but requires careful quota management. |

---

## 2. Concept Mapping (The Rosetta Stone)

| Jenkins Concept | GitHub Actions Equivalent | Senior Engineer Notes |
| :--- | :--- | :--- |
| `Shared Libraries` | **Reusable Workflows** | Move away from complex Groovy logic. Use Reusable Workflows for standard pipelines and Composite Actions for steps. |
| `Credentials Plugin` | **OIDC (OpenID Connect)** | Stop rotating long-lived AWS keys. Use OIDC for short-lived, keyless authentication. |
| `Agent Nodes` | **ARC (Actions Runner Controller)** | Run self-hosted runners on K8s. Auto-scale based on queue depth (webhook-driven). |
| `Folder Authorization` | **GitHub Teams / CODEOWNERS** | Use CODEOWNERS to enforce who can change the pipeline (`.github/workflows`). |

---

## 3. Migration Strategy: The "Strangler Fig" Pattern

**The Trap**: Trying to "lift and shift" 500 jobs in one month.
**The Fix**: A phased, risk-managed approach.

### Phase 1: Audit & Governance (The "Stop the Bleeding" Phase)
1.  **Inventory**: Use the `jenkins-migration-tool` to audit plugins and job types.
2.  **Policy Definition**: Before opening the gates, define **Organization Policies**:
    *   *Allowed Actions*: Only allow Actions from `actions/*` and `verified-partners`.
    *   *Reusable Workflows*: Create a "Golden Path" repo (e.g., `my-org/pipeline-templates`).

### Phase 2: The Parallel Run (Risk Mitigation)
1.  Pick a **Pilot Team** (high velocity, low risk).
2.  Implement **Dual-Running**:
    *   Trigger GHA on `push`.
    *   Keep Jenkins running.
    *   **Compare Artifacts**: Ensure SHA checksums of built binaries match.
3.  **Cost Baseline**: Measure the cost per build to project future spend.

### Phase 3: The Cutover & Optimization
1.  Disable Jenkins jobs.
2.  **Right-Size Runners**: Don't use a 64-core runner for a linter.
3.  **Implement Caching**: Aggressively use `actions/cache` to reduce billable minutes.

---

## 4. Technical Deep Dives (The "8+ Years" Topics)

### A. Enterprise Governance & Supply Chain Security
*   **Problem**: In Jenkins, you installed a plugin once. In GHA, every `uses: author/action@v1` is a potential supply chain attack.
*   **Solution**:
    1.  **Pin by SHA**: `uses: actions/checkout@a5ac7...` (Immutable).
    2.  **Internal Forking**: Fork popular actions to your internal org and vet them.
    3.  **OpenSSF Scorecard**: automated checks for action security.

### B. Cost Management (FinOps)
*   **Jenkins**: "Free" (hidden cost of maintenance).
*   **GHA**: $0.008/min (Ubuntu).
*   **Optimization Strategies**:
    *   **Fail Fast**: Use `fail-fast: true` in matrix builds.
    *   **Timeouts**: Set `timeout-minutes: 10` on every job. Default is 6 hours ($$$).
    *   **Self-Hosted for Heavy Loads**: Use spot instances in your own AWS account for heavy Java builds to save 70%.

### C. Self-Hosted Runners at Scale (ARC)
*   **Architecture**: Install **Actions Runner Controller (ARC)** on EKS/AKS.
*   **Mode**: Use **Runner Sets** (Ephemeral).
    *   Pod starts -> Job runs -> Pod dies.
    *   Ensures a clean environment every time (like Jenkins Docker agents).
*   **Security**: **NEVER** put self-hosted runners on public repos (Fork Bomb risk).

---

## 5. Advanced Migration Scenarios

### Scenario A: The "Groovy Trap" (Complex Logic)
*   **Challenge**: You have a 2000-line `Jenkinsfile` with complex conditional logic, loops, and API calls.
*   **Solution**: **Do NOT try to write this in YAML.** YAML is not a programming language.
    *   **Refactor to Python/Go**: Move the logic into a script (`scripts/deploy.py`).
    *   **Call it from GHA**:
        ```yaml
        - run: python scripts/deploy.py
          env:
            TOKEN: ${{ secrets.TOKEN }}
        ```
    *   *Benefit*: You can unit test the Python script locally. You can't unit test YAML.

### Scenario B: Monorepo Handling
*   **Challenge**: 50 microservices in one repo. A change to `service-A` shouldn't trigger builds for `service-B`.
*   **Solution**:
    ```yaml
    on:
      push:
        paths:
          - 'services/service-A/**'
    concurrency:
      group: service-A-${{ github.ref }}
      cancel-in-progress: true
    ```

### Scenario C: Observability Gap
*   **Challenge**: Jenkins had a dashboard. GHA is just a list of runs. How do we track "Mean Time to Recovery" (MTTR)?
*   **Solution**:
    *   Use **Webhooks** to stream job data to Datadog/Splunk/Prometheus.
    *   Track: `Queue Time`, `Duration`, `Failure Rate` by Team.

---

## 6. The 600-Line Monster: Decomposing a Monolith

**Scenario**: You have a `Jenkinsfile` that is 600+ lines long. It does everything: builds, tests, security scans, deploys to Dev/Stage/Prod, and sends Slack notifications.
**The Mistake**: Trying to rewrite it line-by-line into one giant `.github/workflows/main.yml`.
**The Solution**: **Decomposition**.

### Step 1: Extract Logic to Scripts (The "Lift & Shift")
*   **Problem**: 200 lines of your Jenkinsfile is just complex Bash/Groovy logic (e.g., "If branch is X and time is Y, loop through these folders...").
*   **Fix**: Move this logic to a Python or Bash script.
    *   *Jenkins*:
        ```groovy
        // 50 lines of Groovy logic to calculate version
        ```
    *   *GHA*:
        ```yaml
        - run: python scripts/calculate_version.py
        ```
    *   **Benefit**: You reduce YAML complexity and make the logic testable locally.

### Step 2: Composite Actions (The "Private Plugins")
*   **Problem**: You repeat the same 20 lines of "Setup AWS + Login to ECR + Pull Base Image" in every job.
*   **Fix**: Create a local Composite Action at `.github/actions/setup-aws/action.yml`.
    ```yaml
    # .github/actions/setup-aws/action.yml
    name: 'Setup AWS'
    runs:
      using: "composite"
      steps:
        - uses: aws-actions/configure-aws-credentials@v4
          with: { role-to-assume: ... }
        - run: aws ecr get-login-password | docker login ...
          shell: bash
    ```
    *   *Usage in Workflow*:
        ```yaml
        - uses: ./.github/actions/setup-aws
        ```

### Step 3: Reusable Workflows (The "Pipeline Split")
*   **Problem**: The file is still too big because it contains Build, Test, and Deploy stages.
*   **Fix**: Split it into 3 files.
    1.  `.github/workflows/build.yml` (Reusable)
    2.  `.github/workflows/deploy.yml` (Reusable)
    3.  `.github/workflows/orchestrator.yml` (The Trigger)

    **The Orchestrator**:
    ```yaml
    name: Master Pipeline
    on: [push]
    jobs:
      call-build:
        uses: ./.github/workflows/build.yml
      call-deploy:
        needs: call-build
        uses: ./.github/workflows/deploy.yml
        with:
          artifact-name: ${{ needs.call-build.outputs.artifact }}
    ```

### Visualizing the Transformation

| Legacy Jenkinsfile (600 lines) | Modern GHA Architecture |
| :--- | :--- |
| **Stage: Build** (150 lines) | `build.yml` (Reusable Workflow) |
| **Stage: Security** (100 lines) | `security.yml` (Reusable Workflow) |
| **Function: DeployToAWS** (50 lines) | `.github/actions/deploy-aws` (Composite Action) |
| **Complex Logic** (200 lines) | `scripts/deploy_logic.py` (Python Script) |
| **Orchestration** (100 lines) | `main.yml` (Calls the above) |

---

## 7. Interview Questions (Principal Engineer Level)

### Q1: "How do you justify the migration cost to the CTO?"
*   **Answer**:
    *   "I focus on **Total Cost of Ownership (TCO)**.
    *   Jenkins looks free, but we spend 20 engineering hours/week patching it, managing plugins, and fixing 'disk full' errors. That's $100k/year in engineering time.
    *   GHA reduces that maintenance to near zero.
    *   Plus, the **Developer Velocity** gain (faster feedback, PR integration) translates to faster time-to-market."

### Q2: "How do you handle secrets management in a multi-cloud environment?"
*   **Answer**:
    *   "I avoid long-lived secrets entirely.
    *   I implement **OIDC** for AWS/Azure/GCP to allow runners to assume roles temporarily.
    *   For other secrets (API keys), I integrate with **HashiCorp Vault** or **AWS Secrets Manager** using a dynamic retrieval step, rather than storing static secrets in GitHub repo settings."

### Q3: "Design a 'Golden Path' for 50 teams."
*   **Answer**:
    *   "I would create a centralized `infrastructure-templates` repository.
    *   It would contain **Reusable Workflows** (`.github/workflows/java-build.yml`, `docker-publish.yml`).
    *   Teams would reference these with `uses: my-org/templates/.github/workflows/java-build.yml@v1`.
    *   This allows me to inject security scanning (Trivy/SonarQube) into *every* pipeline centrally. If I need to update the scanner, I update the template, and all 50 teams get the update immediately."
