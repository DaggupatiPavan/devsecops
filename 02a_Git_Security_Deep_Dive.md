# 02a. Git Security Deep Dive: GPG, Pre-commit & History Rewrite

Since you requested "All", here is the deep dive into the advanced Git security topics.

## 1. GPG Signing (The "Verified" Badge)
**Problem**: Anyone can set `git config user.name "Elon Musk"` and commit code.
**Solution**: GPG (GNU Privacy Guard) cryptographically signs your commits.

### Step-by-Step Setup (Windows)
1.  **Install GPG**: Download "Gpg4win" and install it.
2.  **Generate Key**:
    ```powershell
    gpg --full-generate-key
    # Choose (1) RSA and RSA
    # Key size: 4096
    # Expiration: 0 (No expiration)
    # Enter your Name and Email (Must match GitHub email!)
    ```
3.  **Get Key ID**:
    ```powershell
    gpg --list-secret-keys --keyid-format LONG
    # Look for 'sec   rsa4096/3AA5C34371567BD2'
    # Your ID is 3AA5C34371567BD2
    ```
4.  **Export Public Key**:
    ```powershell
    gpg --armor --export 3AA5C34371567BD2
    # Copy the output (BEGIN PGP PUBLIC KEY BLOCK...)
    ```
5.  **Add to GitHub**: Go to Settings -> SSH and GPG Keys -> New GPG Key -> Paste it.
6.  **Configure Git**:
    ```powershell
    git config --global user.signingkey 3AA5C34371567BD2
    git config --global commit.gpgsign true
    ```

---

## 2. Pre-commit Hooks (Blocking Secrets)
**Problem**: You accidentally type `AWS_KEY=AKIA...` and hit Enter. It's too late.
**Solution**: `pre-commit` runs checks *before* the commit is created.

### Setup
1.  **Install**: `pip install pre-commit`
2.  **Create Config**: Make a file named `.pre-commit-config.yaml` in your repo.
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
3.  **Install Hook**: `pre-commit install`
4.  **Test It**: Try to commit a file with `password = "12345"`.
    *   *Result*: `Detect Secrets.........Failed`. The commit is blocked.

---

## 3. Rewriting History (git-filter-repo)
**Problem**: You ignored the advice above and committed a password 3 months ago.
**Solution**: `git-filter-repo` (The modern BFG).

### The Procedure
**WARNING**: This is destructive. Backup your repo first.

1.  **Install**: `pip install git-filter-repo`
2.  **Find the file**: Let's say you committed `config/database.yml` with a password.
3.  **Nuke it**:
    ```powershell
    # This removes 'config/database.yml' from EVERY commit since the beginning of time.
    git filter-repo --path config/database.yml --invert-paths --force
    ```
4.  **Verify**: Check `git log --stat` to ensure the file is gone.
5.  **Push**:
    ```powershell
    # You must force push because history has changed.
    git push origin --force --all
    ```
6.  **Tell Team**: Everyone else must delete their local copy and re-clone. If they pull, they will merge the "bad" history back in!
