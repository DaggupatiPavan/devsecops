# 11. GitHub Actions: The Ultimate Masterclass (1000+ Lines)

**Version**: 2.0 (Python Edition)
**Target Audience**: Beginner to Principal Engineer
**Goal**: To be the single source of truth for GitHub Actions.

---

# Table of Contents

1.  [Part 1: The Core Architecture](#part-1-the-core-architecture)
2.  [Part 2: Workflow Syntax Deep Dive](#part-2-workflow-syntax-deep-dive)
3.  [Part 3: Python in GitHub Actions](#part-3-python-in-github-actions)
4.  [Part 4: Reusable Workflows (The Enterprise Pattern)](#part-4-reusable-workflows)
5.  [Part 5: Creating Custom Actions with Python (Docker)](#part-5-creating-custom-actions-with-python)
6.  [Part 6: Advanced Logic & Matrix Strategies](#part-6-advanced-logic--matrix-strategies)
7.  [Part 7: Security, Secrets & OIDC](#part-7-security-secrets--oidc)
8.  [Part 8: Self-Hosted Runners](#part-8-self-hosted-runners)
9.  [Part 9: API, Webhooks & ChatOps](#part-9-api-webhooks--chatops)
10. [Part 10: Debugging & Local Development](#part-10-debugging--local-development)

---

# Part 1: The Core Architecture

GitHub Actions is an event-driven automation platform. It is not just "CI/CD"; it is a generic automation engine.

## 1.1 The Components

### 1. Workflows (`.yml`)
*   **Location**: Must be in `.github/workflows/`.
*   **Limit**: You can have up to 20 workflows running concurrently per repo (Free tier).
*   **Identity**: Defined by `name:`. If missing, the filename is used.

### 2. Events (`on:`)
*   **Push**: `on: push` (Most common).
*   **Pull Request**: `on: pull_request`.
*   **Schedule**: `on: schedule` (Cron syntax).
*   **Manual**: `on: workflow_dispatch`.
*   **External**: `on: repository_dispatch` (API trigger).

### 3. Jobs
*   **Parallelism**: Jobs run in parallel by default.
*   **Dependencies**: Use `needs: [job1]` to make them sequential.
*   **Environment**: Each job runs on a fresh Virtual Machine (Runner).

### 4. Steps
*   **Sequence**: Steps run sequentially inside a job.
*   **Sharing**: Steps share the same filesystem (`workspace`). If Step 1 writes a file, Step 2 can read it.

### 5. Runners
*   **GitHub Hosted**: Ubuntu, Windows, macOS. Managed by GitHub.
*   **Self-Hosted**: Your own EC2/VM. You pay for compute, GitHub manages orchestration.

---

# Part 2: Workflow Syntax Deep Dive

Let's dissect every important keyword.

## 2.1 The `on` Trigger (Advanced)

### Path Filtering
Don't run CI if only `README.md` changed.
```yaml
on:
  push:
    paths-ignore:
      - '**.md'
      - 'docs/**'
    branches:
      - main
      - 'releases/**'
```

### Tag Filtering
Trigger only on version tags.
```yaml
on:
  push:
    tags:
      - 'v*.*.*'  # Matches v1.0.0
```

### Concurrency (Saving Money)
If I push 3 commits in 1 second, cancel the first 2 builds.
```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

## 2.2 Defaults
Set default shell or working directory for ALL steps.
```yaml
defaults:
  run:
    shell: bash
    working-directory: ./backend
```

## 2.3 Permissions (The GITHUB_TOKEN)
**Best Practice**: Start with 0 permissions and add what you need.
```yaml
permissions:
  contents: read       # Clone code
  pull-requests: write # Comment on PR
  id-token: write      # AWS OIDC
```

---

# Part 3: Python in GitHub Actions

Since you asked for Python specifically, let's master it.

## 3.1 The Setup (`actions/setup-python`)
This is the official action. It handles caching automatically now.

```yaml
steps:
  - uses: actions/checkout@v4
  
  - uses: actions/setup-python@v4
    with:
      python-version: '3.11'
      cache: 'pip' # Automatically caches ~/.cache/pip
      
  - run: pip install -r requirements.txt
```

## 3.2 Running Python Scripts
You have two ways.

### Method A: Inline (Good for short scripts)
```yaml
- name: Run Inline Script
  run: |
    import os
    print(f"Hello from {os.environ['GITHUB_REPOSITORY']}")
  shell: python
```

### Method B: File (Good for complex logic)
```yaml
- name: Run Script File
  run: python scripts/deploy.py
  env:
    API_KEY: ${{ secrets.API_KEY }}
```

## 3.3 Testing with Pytest & Coverage
Generate a report and upload it.

```yaml
- name: Run Tests
  run: |
    pip install pytest pytest-cov
    pytest --cov=./app --cov-report=xml

- name: Upload Coverage
  uses: codecov/codecov-action@v3
  with:
    files: ./coverage.xml
```

## 3.4 Linting (Flake8 / Black)
Fail the build if code is ugly.

```yaml
- name: Lint with Flake8
  run: |
    pip install flake8
    # stop the build if there are Python syntax errors or undefined names
    flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
```

---

# Part 4: Reusable Workflows

This is how you scale. Instead of copying YAML, you call it.

## 4.1 The "Called" Workflow (The Template)
File: `.github/workflows/python-build.yml`

```yaml
name: Python Build Template

on:
  workflow_call:
    inputs:
      python-version:
        required: true
        type: string
      upload-artifact:
        required: false
        type: boolean
        default: false
    secrets:
      PYPI_TOKEN:
        required: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-python@v4
        with:
          python-version: ${{ inputs.python-version }}
          
      - run: pip install .
      
      - run: python setup.py sdist bdist_wheel
      
      - name: Publish
        if: ${{ inputs.upload-artifact }}
        env:
          TWINE_USERNAME: __token__
          TWINE_PASSWORD: ${{ secrets.PYPI_TOKEN }}
        run: twine upload dist/*
```

## 4.2 The "Caller" Workflow (The User)
File: `.github/workflows/my-app.yml`

```yaml
name: My App CI

on: [push]

jobs:
  build-and-publish:
    uses: ./.github/workflows/python-build.yml
    with:
      python-version: '3.9'
      upload-artifact: true
    secrets:
      PYPI_TOKEN: ${{ secrets.PYPI_TOKEN }}
```

## 4.3 Nesting
You can nest reusable workflows up to 4 levels deep.
*   `Caller` -> `Template A` -> `Template B`.

---

# Part 5: Creating Custom Actions with Python

This is the "Expert" level. You want to package a Python script as a GitHub Action that anyone can use with `uses: your/action`.

Since GitHub Runners don't guarantee your specific Python dependencies are present, the best way to distribute a Python Action is via **Docker**.

## 5.1 The Folder Structure
```text
my-python-action/
├── action.yml        # The metadata
├── Dockerfile        # The environment
├── entrypoint.py     # The logic
└── requirements.txt  # The deps
```

## 5.2 Step 1: The Logic (`entrypoint.py`)
This script reads inputs from GitHub and sets outputs.

```python
import os
import sys

def main():
    # 1. Read Inputs (Passed as env vars INPUT_NAME)
    # Input 'who-to-greet' becomes 'INPUT_WHO-TO-GREET'
    name = os.environ.get("INPUT_WHO-TO-GREET", "World")
    
    print(f"Hello, {name}!")
    
    # 2. Business Logic
    if name == "Voldemort":
        print("::error::He who must not be named!")
        sys.exit(1)
        
    # 3. Set Output
    # Old way: print(f"::set-output name=time::{time}")
    # New way: Write to GITHUB_OUTPUT file
    with open(os.environ['GITHUB_OUTPUT'], 'a') as fh:
        print(f"greeting=Hello {name}", file=fh)

if __name__ == "__main__":
    main()
```

## 5.3 Step 2: The Environment (`Dockerfile`)
We need a container with Python installed.

```dockerfile
# Use a slim Python image
FROM python:3.9-slim

# Copy requirements and install
COPY requirements.txt /requirements.txt
RUN pip install -r /requirements.txt

# Copy the script
COPY entrypoint.py /entrypoint.py

# Run it
ENTRYPOINT ["python", "/entrypoint.py"]
```

## 5.4 Step 3: The Metadata (`action.yml`)
This tells GitHub how to run it.

```yaml
name: 'Python Greeter'
description: 'Greets a user using Python'
author: 'Your Name'

inputs:
  who-to-greet:
    description: 'Who to greet'
    required: true
    default: 'World'

outputs:
  greeting:
    description: 'The greeting message'

runs:
  using: 'docker'
  image: 'Dockerfile'
```

## 5.5 Step 4: Usage
Push this to a repo (e.g., `my-org/python-greeter`).

```yaml
steps:
  - uses: my-org/python-greeter@v1
    id: greet
    with:
      who-to-greet: 'Alice'
      
  - run: echo "The output was ${{ steps.greet.outputs.greeting }}"
```

---

# Part 6: Advanced Logic & Matrix Strategies

## 6.1 The Matrix (`strategy`)
Run tests across OS and Python versions.

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false # If one fails, let others finish
      matrix:
        os: [ubuntu-latest, windows-latest]
        python: ['3.8', '3.9', '3.10']
        include:
          - os: macos-latest
            python: '3.11' # Add specific combo
        exclude:
          - os: windows-latest
            python: '3.8' # Skip broken combo
```

## 6.2 Conditional Logic (`if`)
Run steps only under specific conditions.

```yaml
- name: Deploy to Prod
  if: github.ref == 'refs/heads/main' && success()
  run: ./deploy.sh
```

### Common Contexts
*   `failure()`: Returns true if previous step failed.
*   `always()`: Runs even if cancelled or failed (Good for cleanup).
*   `cancelled()`: Runs only if cancelled.

## 6.3 Environments (Manual Approval)
Protect production.
1.  Go to Repo Settings -> Environments -> Create "production".
2.  Add "Required Reviewers".

```yaml
jobs:
  deploy:
    environment: production
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```
The workflow will **PAUSE** here until a human approves it in the UI.

---

# Part 7: Security, Secrets & OIDC

## 7.1 Secrets Management
*   **Repo Secrets**: Available to all actions in repo.
*   **Env Secrets**: Available only to jobs using that Environment.
*   **Org Secrets**: Shared across the organization.

**Warning**: `echo ${{ secrets.PASS }}` will print `***`. But `echo ${{ secrets.PASS }} | base64` might leak it.

## 7.2 OIDC (OpenID Connect) - The "Keyless" Way
Stop storing `AWS_ACCESS_KEY_ID` in GitHub Secrets. Keys expire. Keys leak.
Use OIDC to make AWS trust GitHub directly.

### AWS Side (Trust Policy)
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": { "Federated": "arn:aws:iam::123:oidc-provider/token.actions.githubusercontent.com" },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:*"
                }
            }
        }
    ]
}
```

### GitHub Side
```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123:role/GitHubActionRole
      aws-region: us-east-1
```

---

# Part 8: Self-Hosted Runners

Sometimes `ubuntu-latest` isn't enough. You need 64 cores, or GPU, or access to a private VPN.

## 8.1 Setup
1.  Repo Settings -> Actions -> Runners -> New self-hosted runner.
2.  Run the commands on your VM (Download agent, config, run).

## 8.2 Security Risk (CRITICAL)
**NEVER** use self-hosted runners on **Public Repositories**.
*   A hacker can fork your repo.
*   Modify `.github/workflows/hack.yml`.
*   Run `rm -rf /` or `cat /etc/shadow`.
*   Since it runs on **YOUR** server, they own your server.

## 8.3 Scaling
*   **Actions Runner Controller (ARC)**: Runs runners in Kubernetes. Autoscales based on queue depth.

---

# Part 9: API, Webhooks & ChatOps

## 9.1 Repository Dispatch (Trigger via API)
You want to trigger a build from Postman or Slack.

**Workflow**:
```yaml
on:
  repository_dispatch:
    types: [deploy-command]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying version ${{ github.event.client_payload.version }}"
```

**The Curl Command**:
```bash
curl -X POST https://api.github.com/repos/owner/repo/dispatches \
  -H "Authorization: token $GITHUB_TOKEN" \
  -d '{"event_type": "deploy-command", "client_payload": {"version": "1.2.3"}}'
```

## 9.2 Workflow Dispatch (Manual Inputs)
Great for "Rollback" or "Hotfix" buttons.

```yaml
on:
  workflow_dispatch:
    inputs:
      logLevel:
        description: 'Log Level'
        required: true
        default: 'warning'
        type: choice
        options:
        - info
        - warning
        - debug
```

---

# Part 10: Debugging & Local Development

## 10.1 Enable Debug Logs
If a step fails and the logs are vague:
1.  Go to Secrets.
2.  Add `ACTIONS_RUNNER_DEBUG` = `true`.
3.  Add `ACTIONS_STEP_DEBUG` = `true`.
4.  Re-run. You will see verbose logs.

## 10.2 Running Locally (`act`)
Don't push-and-pray. Run actions on your laptop.
Tool: [nektos/act](https://github.com/nektos/act)

```bash
# Run the push event
act push

# Run a specific job
act -j test
```
*Note*: `act` uses Docker to simulate the runner. It's 95% accurate, but not 100%.

---

# Part 11: The "Anti-Patterns" (What NOT to do)

1.  **Hardcoding Secrets**: Never commit `AWS_KEY` to YAML.
2.  **Ignoring Lockfiles**: Always use `npm ci` or `pip install --frozen` to ensure reproducible builds.
3.  **Over-using `latest`**: `uses: actions/checkout@main` is dangerous. If they break main, you break. Use tags (`v4`) or SHAs.
4.  **Giant Workflows**: If your YAML is 500 lines, split it into Reusable Workflows or Composite Actions.
5.  **Chaining `run` commands**:
    *   *Bad*: `run: cd app && npm install && npm test`
    *   *Good*: Use `working-directory` or separate steps. It makes logs readable.

---

# Part 12: Final Project (The "Kitchen Sink" Workflow)

Let's combine EVERYTHING into one massive, production-grade workflow.

```yaml
name: Production Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  id-token: write
  checks: write

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # 1. Linting (Fastest)
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v4
        with: { python-version: '3.11', cache: 'pip' }
      - run: pip install flake8
      - run: flake8 .

  # 2. Testing (Matrix)
  test:
    needs: lint
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python: ['3.10', '3.11']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v4
        with: { python-version: ${{ matrix.python }}, cache: 'pip' }
      - run: pip install -r requirements.txt
      - run: pytest --cov=./app --cov-report=xml
      - uses: codecov/codecov-action@v3

  # 3. Build Docker Image (Only on Main)
  build-image:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Docker Meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: my-org/my-app
          tags: type=sha
          
      - name: Login to DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ vars.DOCKER_USER }}
          password: ${{ secrets.DOCKER_PASS }}
          
      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}

  # 4. Deploy to Staging (Reusable Workflow)
  deploy-staging:
    needs: build-image
    uses: ./.github/workflows/deploy-template.yml
    with:
      environment: staging
      image: ${{ needs.build-image.outputs.image_tag }}
    secrets: inherit

  # 5. Deploy to Prod (Manual Approval)
  deploy-prod:
    needs: deploy-staging
    environment: production
    uses: ./.github/workflows/deploy-template.yml
    with:
      environment: production
      image: ${{ needs.build-image.outputs.image_tag }}
    secrets: inherit
```

**End of Masterclass.**
