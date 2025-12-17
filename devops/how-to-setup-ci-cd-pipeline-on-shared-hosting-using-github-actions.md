# CI/CD Is Not Only for the Cloud: Building CI/CD for Shared Hosting with GitHub Actions (Laravel)

## Introduction: CI/CD Beyond Cloud Platforms
Continuous Integration and Continuous Deployment (CI/CD) is often associated with cloud platforms like AWS, GCP, or Azure. However, CI/CD pipelines are **not exclusive to cloud infrastructure**. This guide demonstrates how to successfully implement a **CI/CD pipeline for shared hosting** using **GitHub Actions**, even with the limitations of traditional cPanel-based servers.

This article is based on a **real-world deployment** on Namecheap shared hosting, highlighting challenges, fixes, and best practices.

---

## What We Are Building
- CI/CD pipeline using **GitHub Actions**
- Deployment to **shared hosting (Namecheap)**
- Laravel application
- Secure deployment using **SSH**
- Zero-downtime-friendly rsync deployment

---

## Why CI/CD on Shared Hosting Is Challenging
Shared hosting environments are restrictive by design. Some of the difficulties we encountered include:

### 1. Limited Server Control
- No root access
- No system-wide package installation
- Restricted SSH configuration

### 2. SSH Authentication Issues
- SSH keys generated on server vs local machine
- Incorrect key formats causing `libcrypto` errors
- Permission errors (`Permission denied (publickey)`)

### 3. File Permission & Ownership Problems
- GitHub Actions deploy user vs hosting user
- `.htaccess` not respected due to permission mismatch

### 4. Risk of Accidental File Deletion
- Using `rsync --delete` without exclusions
- Losing `.env` and storage files

---

## Step-by-Step: Setting Up CI/CD for Shared Hosting

### Step 1: Generate SSH Key on the Server
Log into your shared hosting via SSH and run:

```bash
ssh-keygen -t rsa -b 2048 -f ~/.ssh/github_actions_key
```

- Do **not** set a passphrase
- This creates:
  - `github_actions_key` (private key)
  - `github_actions_key.pub` (public key)

---

### Step 2: Add Public Key to Authorized Keys
Append the public key:

```bash
cat ~/.ssh/github_actions_key.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

---

### Step 3: Add Private Key to GitHub Secrets
In your GitHub repository:

**Settings → Secrets and variables → Actions → New Repository Secret**

| Name | Value |
|----|------|
| SSH_PRIVATE_KEY | *(contents of github_actions_key)* |
| SSH_HOST | yourdomain.com |
| SSH_USER | hosting_username |
| SSH_PORT | custom_ssh_port |

✅ Use **Repository Secrets**, not Environment secrets.

---

## GitHub Actions Workflow for Shared Hosting Deployment

### Sample Workflow (`deploy.yml`)

```yaml
name: Deploy to Shared Hosting

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan -p ${{ secrets.SSH_PORT }} ${{ secrets.SSH_HOST }} >> ~/.ssh/known_hosts

      - name: Deploy via rsync
        run: |
          rsync -az             --exclude=".env"             --exclude="storage/**"             --exclude="bootstrap/cache/**"             --exclude=".git"             ./ ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST }}:/home/username/project_path
```

---

## Why rsync Is Ideal for Shared Hosting CI/CD
- Transfers **only changed files**
- Faster deployments
- Lower server load
- Avoids full folder replacement

### Does rsync delete everything?
❌ No — unless you use `--delete`.

If you use:
```bash
rsync -az --delete
```
Files **not present in the repo** will be removed on the server.

✅ Recommended:
- Avoid `--delete` on shared hosting
- Or use exclusions very carefully

---

## Fixing .htaccess Not Working After Deployment

### Common Causes
- Wrong file permissions
- Apache not reading `.htaccess`
- Directory listing enabled due to missing rewrite rules

### Recommended Permissions
```bash
chmod 644 .htaccess
chmod 755 project_folder
```

### Correct Laravel Rewrite Rule
```apache
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteRule ^(.*)$ public/$1 [L]
</IfModule>
```

---

## Best Practices for CI/CD on Shared Hosting

### ✅ Security
- Never commit `.env`
- Use SSH keys, not passwords
- Restrict SSH ports

### ✅ Deployment
- Deploy only changed files
- Backup before major changes
- Exclude runtime directories

### ✅ Stability
- Avoid `composer install` on shared hosting
- Build locally or in CI when possible
- Monitor file permissions after deploy

---

## Key Takeaways
- CI/CD is **not limited to cloud platforms**
- Shared hosting can support CI/CD with proper setup
- GitHub Actions + SSH + rsync is a powerful combo
- Understanding hosting limitations is critical

---

## Conclusion
CI/CD pipelines are about **automation**, not infrastructure prestige. With the right approach, even shared hosting environments can benefit from modern DevOps workflows.

If you're running Laravel on shared hosting, CI/CD is absolutely achievable — and worth it.

---

**Keywords:** CI/CD shared hosting, GitHub Actions shared hosting, Laravel CI/CD, deploy Laravel without cloud, CI/CD Namecheap
