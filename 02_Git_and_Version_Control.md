# 02. Git & Version Control Security: Protecting the Source

Code is the blueprint of your organization. If an attacker can modify your code, they own your product.

## 1. Git Internals: Understanding the Object Model
To secure Git, you must understand how it works. Git is a content-addressable filesystem.

*   **Blob**: The file content (source code).
*   **Tree**: The directory structure (maps filenames to blobs).
*   **Commit**: A snapshot of the tree, with author, message, and parent commit hash.
*   **SHA-1 Hash**: Every object is identified by a hash. If you change one bit of a file, the Commit Hash changes. This provides **Integrity**.

### The "Detached HEAD" State
*   Often seen in CI/CD pipelines.
*   Means you are checked out to a specific Commit Hash, not a Branch.
*   **Security Implication**: CI pipelines should build from specific, verified commits, not just "whatever is on main."

---

## 2. Identity & Non-Repudiation (GPG Signing)
How do we know `UserA` actually wrote the code? Anyone can configure `git config user.name "Linus Torvalds"`.

### GPG (GNU Privacy Guard)
GPG allows you to digitally sign your commits. GitHub displays a "Verified" badge.

### Step-by-Step Setup
1.  **Install GPG**:
    *   Windows: `gpg4win`
    *   Linux: `apt install gnupg`
    *   Mac: `brew install gnupg`

2.  **Generate Key**:
    ```bash
    gpg --full-generate-key
    # Select: RSA and RSA
    # Key size: 4096
    # Expiration: 1y
    ```

3.  **Get Key ID**:
    ```bash
    gpg --list-secret-keys --keyid-format LONG
    # Copy the ID after 'sec', e.g., 3AA5C34371567BD2
    ```

4.  **Export Public Key**:
    ```bash
    gpg --armor --export 3AA5C34371567BD2
    # Paste this block into GitHub -> Settings -> SSH and GPG Keys
    ```

5.  **Configure Git**:
    ```bash
    git config --global user.signingkey 3AA5C34371567BD2
    git config --global commit.gpgsign true
    ```

6.  **Test**:
    ```bash
    git commit -m "Signed commit"
    # It will ask for your passphrase.
    ```

---

## 3. Pre-Commit Hooks (The First Line of Defense)
Prevent bad code from ever entering the repo.

### The `pre-commit` Framework
A Python tool to manage hooks.

1.  **Install**: `pip install pre-commit`
2.  **Configure**: Create `.pre-commit-config.yaml` in your repo root.

    ```yaml
    repos:
    -   repo: https://github.com/pre-commit/pre-commit-hooks
        rev: v4.4.0
        hooks:
        -   id: trailing-whitespace
        -   id: end-of-file-fixer
        -   id: check-yaml
        -   id: check-added-large-files

    -   repo: https://github.com/detect-secrets/detect-secrets
        rev: v1.4.0
        hooks:
        -   id: detect-secrets
            args: ['--baseline', '.secrets.baseline']
    ```

3.  **Install Hooks**: `pre-commit install`
4.  **Run**: Now, every time you run `git commit`, these checks run. If `detect-secrets` finds an AWS key, the commit is **blocked**.

---

## 4. Secret Scanning & Remediation

### The Problem
You accidentally committed `AWS_ACCESS_KEY_ID=AKIA...`.
*   **Mistake**: You delete the file and commit again.
*   **Reality**: The key is still in the git history. Hackers clone the repo and run `git log -p` to find it.

### The Solution: `git-filter-repo`
The modern tool to rewrite history (replaces BFG Repo-Cleaner).

1.  **Install**: `pip install git-filter-repo`
2.  **Analyze**: Find the secrets.
    ```bash
    git filter-repo --analyze
    # Check .git/filter-repo/analysis
    ```
3.  **Nuke the file**:
    ```bash
    git filter-repo --path-glob '*.pem' --invert-paths --force
    ```
    *   This removes `*.pem` from *every commit in history*.
4.  **Force Push**: `git push origin --force --all`
    *   *Warning*: This breaks history for everyone else. Coordinate with the team!

### Automated Scanning Tools
*   **TruffleHog**: Scans history for high entropy strings.
    ```bash
    docker run -it -v "$PWD:/pwd" trufflesecurity/trufflehog:latest git file:///pwd
    ```
*   **GitLeaks**: Fast, rule-based scanner.
    ```bash
    gitleaks detect --source . -v
    ```

---

## 5. Branch Protection Rules (GitHub/GitLab)
Enforce security at the server level.

### Recommended Settings for `main` / `master`:
1.  **Require Pull Request Reviews**:
    *   Minimum 1 reviewer (2 is better).
    *   *Code Owners*: Require review from the Security Team if `security/` folder is touched.
2.  **Require Status Checks to Pass**:
    *   Must pass CI Build.
    *   Must pass SonarQube Quality Gate.
    *   Must pass Trivy Image Scan.
3.  **Require Signed Commits**:
    *   Block any commit without a GPG signature.
4.  **Include Administrators**:
    *   Even the CTO cannot push directly to main.

---

## 6. Secure .gitignore
Don't rely on developers to remember. Provide a robust `.gitignore`.

```gitignore
# Secrets
.env
*.pem
*.key
secrets.json
creds.xml

# IDE files
.vscode/
.idea/

# Build artifacts
dist/
build/
*.jar
*.o
```

---

## 7. Interview Preparation

### Q1: I accidentally pushed a password to a public GitHub repo. What do I do?
*   **Answer**:
    1.  **Rotate the Secret**: Immediately revoke the password/key and generate a new one. This is the most important step. The old key is compromised the moment it hits GitHub.
    2.  **Rewrite History**: Use `git-filter-repo` or BFG to remove the file from history.
    3.  **Contact Support**: Ask GitHub Support to remove cached views/forks if necessary.
    4.  **Audit**: Check logs to see if the key was used maliciously in the window it was exposed.

### Q2: What is the difference between `git merge` and `git rebase`? Why does it matter for security?
*   **Answer**:
    *   `merge` preserves history and creates a merge commit.
    *   `rebase` rewrites history to make it linear.
    *   **Security**: Rebasing public branches is dangerous because it changes commit hashes, breaking verification. However, a clean linear history makes auditing changes easier.

### Q3: How do you prevent developers from pushing secrets?
*   **Answer**: Defense in Depth.
    1.  **IDE**: SonarLint to warn them while typing.
    2.  **Pre-commit**: `detect-secrets` hook to block the commit.
    3.  **CI/CD**: TruffleHog scan in the pipeline to block the merge.
    4.  **Platform**: GitHub Secret Scanning to alert if it bypasses everything.

---

## 8. Hands-On Lab
1.  Create a dummy repo.
2.  Create a file `creds.txt` with "password=12345".
3.  Commit it.
4.  Delete the file and commit again.
5.  Use `git log -p` to find the password.
6.  Use `git-filter-repo` to completely remove it from history.
7.  Verify `git log -p` no longer shows it.
