# 14. Jenkins to GitHub Actions Migration Guide (Mid-Level / 4 Years Exp)

**Target Audience**: DevOps Engineers (3-5 Years Exp).
**Goal**: To demonstrate you can **execute** the migration, **debug** issues, and **translate** pipelines independently.

---

## 1. The Role Expectation

At 4 years of experience, the interviewer expects you to be a "Doer".
*   **Yes**: "I migrated 20 microservices," "I fixed the Docker caching issue," "I wrote the workflow templates."
*   **No**: "I designed the multi-year budget strategy" (Too senior) or "I just ran the script my boss gave me" (Too junior).

---

## 2. Core Migration Tasks (The "Hands-On" Work)

### A. Freestyle Jobs -> Workflows
*   **Challenge**: Old Jenkins "Freestyle" jobs are just UI checkboxes and shell scripts.
*   **Migration**:
    1.  **Extract Scripts**: Copy the "Execute Shell" block into a file `scripts/build.sh`.
    2.  **Create Workflow**:
        ```yaml
        name: Build
        on: [push]
        jobs:
          build:
            runs-on: ubuntu-latest
            steps:
              - uses: actions/checkout@v4
              - run: chmod +x scripts/build.sh
              - run: ./scripts/build.sh
        ```

### B. Jenkinsfile -> GitHub Actions
Direct translation of common patterns.

| Jenkins (Groovy) | GitHub Actions (YAML) |
| :--- | :--- |
| `stage('Build') { ... }` | `jobs: build: steps: ...` |
| `post { always { ... } }` | `if: always()` |
| `environment { FOO = 'bar' }` | `env: FOO: 'bar'` |
| `input message: 'Deploy?'` | `environment: production` (Manual Approval) |
| `triggers { cron('H 4 * * *') }` | `on: schedule: - cron: '0 4 * * *'` |

### C. Handling Plugins
Jenkins relies on plugins. GHA relies on **Actions**.
*   **Email Extension Plugin** -> `uses: dawidd6/action-send-mail@v3`
*   **Slack Notification Plugin** -> `uses: slackapi/slack-github-action@v1`
*   **Ansible Plugin** -> `run: ansible-playbook ...` (Just install ansible in the runner).

---

## 3. Technical Deep Dive (Implementation Focused)

### A. Debugging (The #1 Skill)
When a pipeline fails, how do you fix it?
1.  **Enable Debug Logs**: Set secret `ACTIONS_RUNNER_DEBUG` to `true`.
2.  **Local Testing**: Don't commit 50 times to fix a typo. Use **`act`**.
    *   `act -j build` runs the job locally in Docker.
3.  **SSH Debugging**: Use `mxschmitt/action-tmate` to SSH into the failing runner.
    ```yaml
    - name: Debug via SSH
      if: failure()
      uses: mxschmitt/action-tmate@v3
    ```

### B. Artifacts (Passing Data)
Jenkins workspaces persist. GHA runners are wiped.
*   **Job 1 (Build)**: Creates `app.jar`.
    ```yaml
    - uses: actions/upload-artifact@v4
      with:
        name: my-app
        path: target/app.jar
    ```
*   **Job 2 (Deploy)**: Needs `app.jar`.
    ```yaml
    - uses: actions/download-artifact@v4
      with:
        name: my-app
    ```

### C. Caching (Speed)
Jenkins caches `~/.m2` or `node_modules` globally. GHA starts fresh.
*   **Fix**: Use the `setup-*` actions built-in caching.
    ```yaml
    - uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm' # Magic! Caches ~/.npm automatically
    ```

### D. Matrix Builds (The "For Loop")
Jenkins used complex Groovy loops to test multiple versions. GHA makes it native.
*   **Scenario**: Test on Node 14, 16, and 18.
    ```yaml
    jobs:
      test:
        runs-on: ubuntu-latest
        strategy:
          matrix:
            node-version: [14, 16, 18]
        steps:
          - uses: actions/setup-node@v4
            with:
              node-version: ${{ matrix.node-version }}
          - run: npm test
    ```
    *   *Result*: 3 parallel jobs run automatically.

### E. Service Containers (Sidecars)
Jenkins required `docker run` or complex plugins to spin up a DB for testing.
*   **Scenario**: Integration test needing Postgres.
    ```yaml
    jobs:
      test:
        runs-on: ubuntu-latest
        services:
          postgres:
            image: postgres:15
            env:
              POSTGRES_PASSWORD: root
            ports:
              - 5432:5432
            options: >-
              --health-cmd pg_isready
              --health-interval 10s
              --health-timeout 5s
              --health-retries 5
        steps:
          - run: npm test # Can connect to localhost:5432
    ```

### F. Permissions (Security)
Jenkins had "Full Admin" or "Read Only". GHA is granular.
*   **Best Practice**: Always define `permissions` at the top.
    ```yaml
    permissions:
      contents: read       # Can clone code
      pull-requests: write # Can comment on PRs
      id-token: write      # Can talk to AWS (OIDC)
    ```
    *   *Tip*: If you get "403 Forbidden" when pushing a tag or release, check this first.

### G. Contexts (Making Decisions)
How do you access metadata?
*   `${{ github.ref }}` -> Branch name (e.g., `refs/heads/main`).
*   `${{ github.sha }}` -> Commit hash.
*   `${{ runner.os }}` -> `Linux`, `Windows`, or `macOS`.
*   `${{ secrets.MY_KEY }}` -> Encrypted secrets.
    ```yaml
    - name: Print Branch
      run: echo "Running on branch ${{ github.ref }}"
    ```

---

## 4. Common Challenges & Fixes

### Challenge 1: "It works on my machine/Jenkins but fails in GHA"
*   **Cause**: Environment Variables. Jenkins loads `/etc/profile`. GHA does not.
*   **Fix**: Explicitly set `PATH` or env vars in `$GITHUB_ENV`.
    ```yaml
    - run: echo "/opt/my-tool/bin" >> $GITHUB_PATH
    ```

### Challenge 2: "Secrets aren't working"
*   **Cause**: You tried to use `echo $SECRET`.
*   **Fix**: You must map secrets to env vars.
    ```yaml
    env:
      API_KEY: ${{ secrets.API_KEY }} # Required!
    run: echo $API_KEY
    ```

### Challenge 3: "The script isn't executable"
*   **Cause**: Git permissions.
*   **Fix**: `run: chmod +x myscript.sh` before running it.

---

## 5. Interview Questions (Mid-Level)

### Q1: "How do you troubleshoot a failing Action that works locally?"
*   **Answer**:
    *   "First, I check the logs. If they are vague, I enable `ACTIONS_RUNNER_DEBUG`.
    *   I verify the environment (Node version, Java version) matches my local.
    *   I check if secrets are correctly mapped.
    *   If stuck, I use `tmate` to SSH into the runner and run commands manually."

### Q2: "Explain the difference between `jobs` and `steps`."
*   **Answer**:
    *   "**Steps** run sequentially in the *same* runner (VM). They share the filesystem.
    *   **Jobs** run in parallel on *different* runners. They do *not* share files (unless you use Artifacts).
    *   I use Steps for a single process (Build) and Jobs for independent tasks (Test on Linux, Test on Windows)."

### Q3: "How do you run a job only on the `main` branch?"
*   **Answer**:
    *   "I use the `if` conditional at the job level:
        `if: github.ref == 'refs/heads/main'`
    *   Or I filter it at the trigger level:
        ```yaml
        on:
          push:
            branches: [main]
        ```
    *   Trigger level is better because it saves runner startup time."
