# GitHub Repository – Push Instructions

## Prerequisites

- Git installed: `git --version`
- GitHub account with access to your existing repository
- Repository URL ready (e.g., `https://github.com/your-username/your-repo.git`)

---

## One-Time Setup (first push)

### 1. Initialize Git in the project folder
```bash
cd "Mist WLAN Best Practices"
git init
git remote add origin https://github.com/your-username/your-repo.git
```

### 2. Create a `.gitignore` to protect sensitive files
```bash
cat > .gitignore << 'EOF'
# Exclude log files (may contain org data)
logs/

# Exclude generated Excel reports
*.xlsx

# Exclude Python cache
__pycache__/
*.pyc
*.pyo

# Exclude environment files
.env
*.env
secrets.txt

# Exclude IDE files
.vscode/
.idea/
EOF
```

### 3. Stage and commit all project files
```bash
git add mist_wlan_best_practices.py
git add requirements.txt
git add README.md
git add INSTRUCTIONS.md
git add SCALABILITY.md
git add GITHUB_PUSH.md
git add .gitignore

git commit -m "Initial commit: Mist WLAN Best Practices automation script v1.0"
```

### 4. Push to GitHub
```bash
git branch -M main
git push -u origin main
```

---

## Subsequent Updates

After making changes to the script:
```bash
git add mist_wlan_best_practices.py
git commit -m "Description of what changed"
git push
```

---

## Recommended Repository Structure

```
your-repo/
├── mist_wlan_best_practices.py   ← Main script
├── requirements.txt              ← Python dependencies
├── README.md                     ← Full documentation
├── INSTRUCTIONS.md               ← Run guide
├── SCALABILITY.md                ← Load analysis
├── GITHUB_PUSH.md                ← This file
└── .gitignore                    ← Excludes logs, tokens, cache
```

---

## Security Reminders

**Never commit:**
- API tokens (read-only or read-write)
- Org IDs if the repository is public
- Log files (they may contain site names, client data)
- `.env` files with credentials

**If you accidentally commit a token:**
1. Immediately revoke it in the Mist portal (Organization → Settings → API Token)
2. Remove it from git history: `git filter-branch` or use `git-filter-repo`
3. Force-push the cleaned history: `git push --force`

---

## Setting Up GitHub Actions (optional CI)

To automatically lint the script on every push, create `.github/workflows/lint.yml`:

```yaml
name: Lint Python

on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - run: pip install flake8
      - run: flake8 mist_wlan_best_practices.py --max-line-length=120
```

---

## Tagging Releases

When creating a new version:
```bash
git tag -a v1.0.0 -m "Initial release"
git push origin v1.0.0
```
