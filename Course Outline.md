# GitHub Actions: Beginner Course
### Goal: Automate Config File Deployments to Staging & Prod

---

## How This Course Is Structured

You'll build a single project across **4 phases**, each one teaching real concepts while moving you toward your actual work goal. By the end, you'll have a working repo that automatically deploys config files to a staging server on one branch and a production server on another — with human approval protecting prod.

**Each phase builds on the last. Do them in order.**

---

## Prerequisites

Before Phase 1, make sure you have:

- A GitHub account (free)
- Git installed in your WSL2/Fedora environment
- VS Code with the **WSL** extension and the **GitHub Actions** extension (syntax highlighting + validation)
- Run this in your Fedora terminal to confirm Git is ready:
  ```bash
  git --version
  git config --global user.name "Your Name"
  git config --global user.email "you@example.com"
  ```

---

## The Mental Model (Read This First)

GitHub Actions is an automation engine that lives inside your GitHub repo. The core idea:

```
Something happens in your repo  →  GitHub runs a script for you
       (the "trigger")                    (the "workflow")
```

Everything is defined in **YAML files** you commit to `.github/workflows/` in your repo. That's it. No separate tool to install, no dashboard to configure — it's all just files in your repo.

**The vocabulary you need to know:**

| Term | What it is | Real-world analogy |
|---|---|---|
| **Workflow** | A YAML file that defines automation | A recipe |
| **Trigger (`on:`)** | What causes the workflow to run | The smoke detector going off |
| **Job** | A group of steps that run together on one machine | One person's job duties |
| **Step** | A single task inside a job | One line item on a to-do list |
| **Runner** | The temporary VM GitHub spins up to run your job | A temp worker hired for the day |
| **Action** | A reusable, pre-built step (from the GitHub Marketplace) | A power tool you borrow |
| **Secret** | An encrypted variable (passwords, SSH keys) | A locked drawer only the runner can open |
| **Environment** | A named deployment target with optional protection rules | The "staging lane" vs "prod lane" |

---

## Phase 1 — Hello World: Understand the YAML Structure

**Concepts taught:** Workflow files, triggers, jobs, steps, runners, the Actions tab

### Step 1: Create Your Repo

In your Fedora terminal:

```bash
mkdir config-deploy-practice
cd config-deploy-practice
git init
git branch -M main
```

Create a simple README:

```bash
echo "# Config Deploy Practice" > README.md
git add .
git commit -m "initial commit"
```

Go to GitHub.com, create a new **public** repo called `config-deploy-practice`, then push:

```bash
git remote add origin https://github.com/YOUR_USERNAME/config-deploy-practice.git
git push -u origin main
```

### Step 2: Create Your First Workflow

In VS Code (open the WSL folder), create this directory structure:

```bash
mkdir -p .github/workflows
```

Create the file `.github/workflows/hello.yml`:

```yaml
# The name that shows up in the GitHub Actions tab
name: Hello World

# TRIGGER: When does this workflow run?
on:
  push:                    # Run when code is pushed...
    branches:
      - main               # ...but only to the main branch

# JOBS: What do we actually do?
jobs:

  say-hello:               # This is the job ID (you name it)
    runs-on: ubuntu-latest # Use GitHub's free Ubuntu runner

    steps:                 # Steps run top-to-bottom, in order

      - name: Print a greeting
        run: echo "Hello from GitHub Actions!"

      - name: Show the current date and time
        run: date

      - name: Show what directory we're in
        run: pwd

      - name: List files (the runner starts empty!)
        run: ls -la
```

Commit and push:

```bash
git add .github/
git commit -m "add hello world workflow"
git push
```

### Step 3: Watch It Run

1. Go to your repo on GitHub.com
2. Click the **Actions** tab
3. Click your "Hello World" run
4. Click the `say-hello` job
5. Expand each step and read the output

**What to notice:**
- The runner is a fresh, empty Ubuntu VM — it has no idea what your project is yet
- Each `run:` command is just a bash command
- The whole thing took maybe 5-10 seconds

> ✅ **Phase 1 complete.** You now understand the skeleton of every GitHub Actions workflow that will ever exist.

---

## Phase 2 — Branches, Conditions, and Config Files

**Concepts taught:** Branch-based triggers, `if:` conditions, `workflow_dispatch` (manual trigger), working with actual files

This phase simulates the staging vs. prod split you want at work.

### Step 1: Create a Config File Structure

This mirrors how your Perceptive Content environment might be structured:

```bash
mkdir -p config/staging config/prod
```

Create `config/staging/app.conf`:

```ini
[database]
host = staging-db.internal
port = 5432
name = perceptive_staging

[app]
log_level = DEBUG
max_connections = 10
environment = staging
```

Create `config/prod/app.conf`:

```ini
[database]
host = prod-db.internal
port = 5432
name = perceptive_prod

[app]
log_level = WARN
max_connections = 100
environment = production
```

Commit these:

```bash
git add config/
git commit -m "add staging and prod config files"
git push
```

### Step 2: Create a Staging Branch

```bash
git checkout -b staging
git push -u origin staging
```

### Step 3: Create a Smarter Workflow

Create `.github/workflows/deploy-configs.yml`:

```yaml
name: Deploy Config Files

# TRIGGER: Run on pushes to main or staging branches
# Also allow manual runs from the GitHub UI
on:
  push:
    branches:
      - main      # Push to main = deploy to prod
      - staging   # Push to staging = deploy to staging
    paths:
      - 'config/**'              # trigger on config changes
      - '.github/workflows/**'   # also trigger when the workflow file itself changes
  workflow_dispatch:  # This adds a "Run workflow" button in the UI
                      # ⚠️ Note: the button only appears once this workflow exists
                      # on the repo's DEFAULT branch (usually `main`). If you've
                      # only pushed to `staging` so far, you won't see the button
                      # until after you merge to `main` (Test 3 below).
    inputs:
      target:
        description: 'Which environment to deploy?'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - prod

jobs:

  determine-environment:
    runs-on: ubuntu-latest
    # This job figures out WHERE we're deploying
    # and passes that info to the next job

    outputs:
      environment: ${{ steps.set-env.outputs.environment }}

    steps:
      - name: Figure out target environment
        id: set-env
        run: |
          # If manually triggered, use the input value
          if [ "${{ github.event_name }}" == "workflow_dispatch" ]; then
            echo "environment=${{ inputs.target }}" >> $GITHUB_OUTPUT

          # If pushed to main branch, deploy to prod
          elif [ "${{ github.ref_name }}" == "main" ]; then
            echo "environment=prod" >> $GITHUB_OUTPUT

          # If pushed to staging branch, deploy to staging
          elif [ "${{ github.ref_name }}" == "staging" ]; then
            echo "environment=staging" >> $GITHUB_OUTPUT

          else
            echo "environment=unknown" >> $GITHUB_OUTPUT
          fi

      - name: Print what we decided
        run: echo "Target environment is ${{ steps.set-env.outputs.environment }}"


  deploy:
    runs-on: ubuntu-latest
    needs: determine-environment  # Wait for the job above to finish

    steps:

      # This built-in action checks out YOUR repo onto the runner
      # Without this step, the runner has no idea your files exist
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Show the config file we would deploy
        run: |
          TARGET="${{ needs.determine-environment.outputs.environment }}"
          echo "=== Deploying configs for: $TARGET ==="
          echo ""
          echo "--- Contents of config/$TARGET/app.conf ---"
          cat config/$TARGET/app.conf

      # This is where real SSH deployment would go (Phase 4)
      # For now, we just simulate it
      - name: Simulate deployment
        run: |
          TARGET="${{ needs.determine-environment.outputs.environment }}"
          echo "✅ Would now SCP config/$TARGET/app.conf to $TARGET server"
          echo "✅ Would restart application service on $TARGET"
          echo "✅ Deployment to $TARGET complete (simulated)"
```

Commit and push (you're on the `staging` branch right now):

```bash
git add .github/ config/
git commit -m "add deploy workflow with branch logic"
git push
```

### Step 4: Test All Three Triggers

**Important ordering note:** the manual "Run workflow" button (`workflow_dispatch`)
only appears in the GitHub UI once the workflow file exists on the repo's
**default branch** (`main`). Because we've only committed it on `staging` so far,
we'll do the manual-trigger test LAST — after merging to `main` makes the
button available.

**Test 1 — Staging auto-deploy:**
You just pushed to `staging`, so a workflow should already be running. Check the
Actions tab.

> Gotcha: the `paths:` filter means the workflow only fires if files matching
> those globs changed in this push. If you ever push a commit that *only* edits
> something outside those paths (e.g. just the README), GitHub will skip the
> run — that's not a bug, that's the filter working.

**Test 2 — Prod deploy on merge:**
```bash
git checkout main
git merge staging
git push
```
Watch the workflow trigger on `main` and deploy to prod. As a side effect, the
workflow file now lives on `main`, which unlocks the manual-dispatch button for
the next test.

**Test 3 — Manual trigger:**
1. Go to Actions tab → "Deploy Config Files" workflow
2. Click **Run workflow** (this button only appears now that the workflow is on `main`)
3. Choose `prod` from the dropdown
4. Watch it run and notice it says "prod" in the output

> ✅ **Phase 2 complete.** You understand branch triggers, conditions, job outputs, `uses:` (pre-built actions), and manual dispatch.

---

## Phase 3 — Secrets and Environments (The Safety Net)

**Concepts taught:** GitHub Secrets, Environments, required reviewers (approval gates), environment-scoped secrets

This is the phase that makes it safe to automate prod.

### Step 1: Create GitHub Environments

In your repo on GitHub.com:

1. Go to **Settings → Environments**
2. Click **New environment** → name it `staging` → click **Configure environment**
   - Leave everything default for now. Click **Save protection rules**.
3. Click **New environment** → name it `production` → click **Configure environment**
   - Check **Required reviewers**
   - Add yourself as a required reviewer
   - Click **Save protection rules**

This means: any workflow that targets the `production` environment will **pause and wait for your approval** before running.

### Step 2: Add Secrets

Still in **Settings → Environments**, click on `staging`:
- Click **Add secret**
- Name: `SERVER_HOST` | Value: `192.168.1.100` (fake staging IP for now)
- Add another: `SERVER_USER` | Value: `deploy`

Click on `production`:
- Add `SERVER_HOST` | Value: `192.168.1.200` (fake prod IP)
- Add `SERVER_USER` | Value: `deploy`

> **Why environment-scoped secrets?** A secret on the `production` environment can ONLY be accessed by a job that declares `environment: production`. Even if someone tricks the staging workflow into running, it can't reach the prod secret.

### Step 3: Update the Workflow to Use Environments

Update `.github/workflows/deploy-configs.yml` — replace the `deploy` job with this:

```yaml
  deploy:
    runs-on: ubuntu-latest
    needs: determine-environment

    # THIS LINE is what activates environment protection and secrets
    environment: ${{ needs.determine-environment.outputs.environment }}

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Show deployment target
        run: |
          echo "Deploying to: ${{ needs.determine-environment.outputs.environment }}"
          echo "Server host: ${{ secrets.SERVER_HOST }}"
          echo "Server user: ${{ secrets.SERVER_USER }}"
          # Note: actual secret VALUES are masked in logs (shown as ***)

      - name: Simulate SSH deployment
        run: |
          TARGET="${{ needs.determine-environment.outputs.environment }}"
          HOST="${{ secrets.SERVER_HOST }}"
          USER="${{ secrets.SERVER_USER }}"
          echo "✅ Would run: scp config/$TARGET/app.conf $USER@$HOST:/etc/perceptive/"
          echo "✅ Would run: ssh $USER@$HOST 'systemctl restart perceptive-content'"
```

Commit and push to `staging`, then merge to `main`:

```bash
git add .github/
git commit -m "add environments and secrets to workflow"
git push origin staging
git checkout main
git merge staging
git push origin main
```

**Watch the magic happen:**
- The `staging` deploy runs immediately ✅
- The `production` deploy **stops and shows a yellow "Waiting" badge** 🟡
- You'll get a GitHub notification to approve it
- Go to the Actions tab, click the waiting run, click **Review deployments**, approve it
- Watch prod deploy only after your approval ✅

> ✅ **Phase 3 complete.** You now have the safety infrastructure that makes automated prod deployments trustworthy.

---

## Phase 4 — The Real Thing: Actual SSH Deployment

**Concepts taught:** SSH key secrets, `appleboy/ssh-action`, real file transfer with SCP

This phase connects to real servers. Use your actual staging server first.

### Step 1: Generate a Dedicated Deploy SSH Key

On your Fedora/WSL machine (do NOT use your personal SSH key):

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_actions_deploy
# Press Enter twice for no passphrase (required for automation)
```

This creates two files:
- `~/.ssh/github_actions_deploy` — the **private key** (goes into GitHub Secrets)
- `~/.ssh/github_actions_deploy.pub` — the **public key** (goes on your servers)

### Step 2: Install the Public Key on Your Servers

On each server (staging first), run:

```bash
ssh your-admin-user@YOUR_STAGING_SERVER
mkdir -p ~/.ssh
echo "PASTE_PUBLIC_KEY_CONTENTS_HERE" >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Get the public key contents with:
```bash
cat ~/.ssh/github_actions_deploy.pub
```

### Step 3: Add the Private Key as a GitHub Secret

1. Get the private key: `cat ~/.ssh/github_actions_deploy`
2. Copy the entire output (including the `-----BEGIN...` and `-----END...` lines)
3. In GitHub → **Settings → Environments → staging**
4. Add secret: `SSH_PRIVATE_KEY` → paste the private key

Repeat for the `production` environment.

Also update `SERVER_HOST` and `SERVER_USER` with your real server values.

### Step 4: Update the Workflow for Real Deployment

Replace the `deploy` job's steps with:

```yaml
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Deploy config file via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            # Create the config directory if it doesn't exist
            mkdir -p /etc/perceptive/

      - name: Copy config file to server
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "config/${{ needs.determine-environment.outputs.environment }}/app.conf"
          target: "/etc/perceptive/"
          strip_components: 2   # removes the config/staging/ prefix

      - name: Restart application service
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            echo "Config deployed successfully"
            # Uncomment when ready to actually restart:
            # systemctl restart perceptive-content
            # systemctl status perceptive-content
```

> **Note on `appleboy` actions:** These are the most widely-used SSH/SCP actions on the GitHub Marketplace. The `uses:` syntax pulls them automatically — no installation needed.

Commit, push to staging, test, then merge to main and approve the prod deployment.

> ✅ **Phase 4 complete.** You now have a real, working, safe automated deployment pipeline.

---

## Your Final Workflow Reference

Here's how your complete flow works when done:

```
Edit config/staging/app.conf  →  push to staging branch
        ↓
GitHub Actions triggers automatically
        ↓
Checks out your repo on a fresh Ubuntu VM
        ↓
Detects branch = staging → target = staging
        ↓
Reads staging environment secrets (SERVER_HOST, SSH key)
        ↓
SCPs the file to your staging server
        ↓
Done ✅  (no approval needed for staging)

─────────────────────────────────────────────────

git checkout main && git merge staging && git push
        ↓
GitHub Actions triggers automatically
        ↓
Detects branch = main → target = production
        ↓
Workflow PAUSES — sends you an approval notification 🟡
        ↓
You review the changes and click Approve
        ↓
Reads production environment secrets
        ↓
SCPs the file to your prod server
        ↓
Done ✅
```

---

## Key Concepts Cheat Sheet

```yaml
# Trigger on push to specific branches, only when certain files change
on:
  push:
    branches: [main, staging]
    paths: ['config/**']

# Add a manual "Run workflow" button with inputs
on:
  workflow_dispatch:
    inputs:
      target:
        type: choice
        options: [staging, prod]

# Check out your repo onto the runner (almost always needed)
- uses: actions/checkout@v4

# Run a bash command
- name: My step
  run: echo "hello"

# Multi-line bash
- name: Multi-line
  run: |
    echo "line 1"
    echo "line 2"

# Use a secret (from repo or environment settings)
run: ssh ${{ secrets.SERVER_USER }}@${{ secrets.SERVER_HOST }}

# Reference another job's output
needs: some-other-job
run: echo ${{ needs.some-other-job.outputs.my_output }}

# Only run this step if a condition is true
- name: Only on main
  if: github.ref_name == 'main'
  run: echo "this is main"

# Attach this job to an environment (enables secrets + approval gates)
jobs:
  deploy:
    environment: production
```

---

## What to Explore Next

Once you're comfortable with this project:

- **Notifications** — Add a step that posts to a Teams/Slack channel on success or failure
- **Diff comments** — Have the workflow post a summary of what config lines changed as a PR comment
- **Matrix jobs** — Deploy to multiple servers in parallel
- **Reusable workflows** — Extract the deploy logic into a shared workflow file so you can reuse it across repos
- **GitHub Actions cache** — Speed up workflows that install dependencies
