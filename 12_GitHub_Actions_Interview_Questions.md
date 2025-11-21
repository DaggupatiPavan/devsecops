# 12. GitHub Actions Interview Questions (60 Questions)

This document contains **60 Interview Questions** specifically for GitHub Actions, ranging from Beginner to Expert.

---

## Part 1: Basics & Architecture (1-10)

1.  **What is a Workflow?**
    *   A configurable automated process made up of one or more jobs. Defined in a YAML file in `.github/workflows`.
2.  **What is the difference between a Job and a Step?**
    *   **Job**: A set of steps that execute on the same runner. Jobs run in parallel by default.
    *   **Step**: An individual task (command or action) inside a job. Steps run sequentially.
3.  **Where must workflow files be located?**
    *   In the `.github/workflows/` directory of the repository.
4.  **What is a Runner?**
    *   A server that runs your workflow jobs. It can be GitHub-hosted (Ubuntu, Windows, Mac) or Self-hosted.
5.  **How do you trigger a workflow manually?**
    *   Use the `workflow_dispatch` event in the `on:` section.
6.  **How do you make Job B wait for Job A?**
    *   Use `needs: [JobA]` in the definition of Job B.
7.  **What is an "Action"?**
    *   A custom application for the GitHub Actions platform that performs a complex but frequently repeated task.
8.  **How do you reference an action in a step?**
    *   Using the `uses` keyword (e.g., `uses: actions/checkout@v4`).
9.  **Can you run a shell command in a step?**
    *   Yes, using the `run` keyword (e.g., `run: echo "Hello"`).
10. **What is the default shell on Windows runners?**
    *   PowerShell (`pwsh`). On Linux/Mac, it is `bash`.

---

## Part 2: Triggers & Contexts (11-20)

11. **How do you trigger a workflow only when a specific file changes?**
    *   Use `paths:` or `paths-ignore:` under the `push` or `pull_request` event.
12. **How do you trigger a workflow on a schedule?**
    *   Use `on: schedule` with a cron syntax (e.g., `- cron: '0 0 * * *'`).
13. **What is the `github` context?**
    *   It contains information about the workflow run and the event that triggered it (e.g., `github.sha`, `github.ref`, `github.actor`).
14. **How do you access the branch name in a workflow?**
    *   `${{ github.ref_name }}` (for simple name) or `${{ github.ref }}` (full ref like `refs/heads/main`).
15. **How do you skip a workflow run for a specific commit?**
    *   Add `[skip ci]` or `[ci skip]` to the commit message.
16. **What is `concurrency` used for?**
    *   To ensure only a single job or workflow using the same concurrency group runs at a time. Often used to cancel outdated builds on PRs.
17. **How do you pass data between Steps in the same Job?**
    *   By writing to the `$GITHUB_OUTPUT` file (e.g., `echo "my_val=123" >> $GITHUB_OUTPUT`) and reading it via `${{ steps.id.outputs.my_val }}`.
18. **How do you pass data between Jobs?**
    *   Using `outputs` at the job level. Job A defines outputs, Job B uses `needs: JobA` and accesses `${{ needs.JobA.outputs.val }}`.
19. **What is the `env` context?**
    *   It contains environment variables set in the workflow, job, or step.
20. **Can you use `if` conditionals at the Job level?**
    *   Yes, to prevent a job from running (e.g., `if: github.ref == 'refs/heads/main'`).

---

## Part 3: Secrets & Variables (21-30)

21. **Where do you store sensitive data like API keys?**
    *   In GitHub Secrets (`Settings -> Secrets and variables -> Actions`).
22. **How do you access a secret in a workflow?**
    *   `${{ secrets.MY_SECRET_NAME }}`.
23. **What happens if you try to print a secret to the logs?**
    *   GitHub Actions automatically masks it with `***`.
24. **What is the difference between Repository Secrets and Environment Secrets?**
    *   Repo Secrets are global. Environment Secrets only apply when the job references that specific `environment`.
25. **What is `GITHUB_TOKEN`?**
    *   An automatically generated secret used to authenticate in a workflow run. It has permissions to the repo.
26. **How do you change the permissions of `GITHUB_TOKEN`?**
    *   Use the `permissions:` key at the workflow or job level.
27. **Can `GITHUB_TOKEN` trigger new workflows?**
    *   No, by default it cannot trigger new recursive workflow runs to prevent infinite loops. You need a PAT (Personal Access Token) for that.
28. **What are Configuration Variables?**
    *   Non-sensitive configuration data stored in `vars` context (`${{ vars.MY_CONFIG }}`).
29. **How do you pass a secret to a generic action?**
    *   Usually via `with:` or `env:` depending on the action's documentation.
30. **Can forks access secrets?**
    *   No, by default workflows triggered by pull requests from forks do not have access to secrets.

---

## Part 4: Runners & Environments (31-40)

31. **What is a Self-Hosted Runner?**
    *   A machine you manage and maintain that runs the GitHub Actions runner application.
32. **Why would you use a Self-Hosted Runner?**
    *   For custom hardware (GPU), access to private internal networks, or cost savings on massive usage.
33. **What is the security risk of Self-Hosted Runners?**
    *   If used on public repos, forks can run malicious code on your physical server.
34. **What is an "Environment" in GitHub Actions?**
    *   A logical target for deployment (e.g., "Production", "Staging") that can have protection rules.
35. **How do you require manual approval for a deployment?**
    *   Configure "Required Reviewers" in the Environment settings and reference `environment: name` in the job.
36. **What is a Matrix Strategy?**
    *   A way to run a job multiple times with different variable combinations (e.g., Node 14, 16, 18).
37. **How do you stop the entire matrix if one job fails?**
    *   By default it stops. Set `fail-fast: false` to keep others running.
38. **What is a Container Job?**
    *   A job where steps run inside a specified Docker container instead of directly on the runner machine.
39. **How do you use a specific Docker image for a job?**
    *   `container: image: node:14`.
40. **What is a Service Container?**
    *   Additional containers (like Redis or Postgres) spun up alongside the job for testing.

---

## Part 5: Artifacts & Caching (41-50)

41. **What is an Artifact?**
    *   Files generated by a job (e.g., binaries, test reports) that you want to persist after the job ends.
42. **How do you upload an artifact?**
    *   Use `actions/upload-artifact`.
43. **How do you download an artifact in a later job?**
    *   Use `actions/download-artifact`.
44. **What is Caching?**
    *   Storing dependencies (like `node_modules`) to speed up future runs.
45. **How is Caching different from Artifacts?**
    *   Cache is for speed (restoring deps). Artifacts are for data preservation (build outputs). Cache can be evicted; Artifacts expire.
46. **How do you implement caching?**
    *   Use `actions/cache` with a unique `key` (often based on a hash of lockfiles).
47. **What happens if a cache key misses?**
    *   The step runs (downloads deps), and at the end of the job, the new cache is saved.
48. **Can you share artifacts between workflow runs?**
    *   No, artifacts are scoped to the specific run. You would need external storage (S3) or a download script.
49. **What is the retention period for artifacts?**
    *   Default is 90 days (configurable).
50. **Does `setup-python` or `setup-node` support caching?**
    *   Yes, modern versions have built-in caching (e.g., `cache: 'pip'`).

---

## Part 6: Advanced & Custom Actions (51-60)

51. **What are Reusable Workflows?**
    *   Workflows that can be called by other workflows. Defined with `on: workflow_call`.
52. **What are the 3 types of Custom Actions?**
    *   Composite, JavaScript, and Docker.
53. **What is a Composite Action?**
    *   An action written in YAML that combines multiple run steps. Easiest to create.
54. **Why use a Docker Action?**
    *   To ensure a specific environment/language (like Python or Ruby) is available regardless of the runner.
55. **Why use a JavaScript Action?**
    *   It runs directly on the runner (fast) and has access to the `@actions/toolkit` for complex logic.
56. **How do you debug a failing action?**
    *   Enable `ACTIONS_RUNNER_DEBUG` secret to true to see verbose logs.
57. **What is OIDC in the context of AWS?**
    *   OpenID Connect. Allows GitHub to authenticate with AWS using a temporary token instead of long-lived Access Keys.
58. **How do you test actions locally?**
    *   Using a tool like `nektos/act`.
59. **What is `pull_request_target`?**
    *   A trigger that runs in the context of the *base* repository (not the merge commit). It has access to secrets but is dangerous if not handled carefully.
60. **How do you pin an action version securely?**
    *   Use the SHA hash instead of the tag (e.g., `uses: actions/checkout@a1b2c3d4...`).
