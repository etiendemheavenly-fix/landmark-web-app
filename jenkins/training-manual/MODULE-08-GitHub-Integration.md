# MODULE 8: INTEGRATING GITHUB WITH JENKINS

## Learning Objectives

By the end of this module, students will be able to:
- Configure GitHub authentication (PAT, SSH)
- Set up GitHub Webhooks for automatic builds
- Create Multibranch Pipelines
- Understand branch-based build strategies

---

## 8.1 Authentication Methods

### Method 1: Personal Access Token (PAT) — Recommended

```
┌──────────┐    HTTPS + PAT     ┌──────────┐
│  Jenkins │ ──────────────────▶ │  GitHub  │
│          │                     │          │
└──────────┘                     └──────────┘
```

**Create a PAT:**
1. GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Click **Generate new token (classic)**
3. Scopes: `repo`, `admin:repo_hook`
4. Copy the token

**Add to Jenkins:**
1. **Manage Jenkins → Credentials → System → Global credentials → Add**
2. Kind: **Secret text**
3. Secret: (paste your PAT)
4. ID: `github-token`
5. Description: "GitHub PAT"

### Method 2: SSH Key

**Generate key:**
```bash
ssh-keygen -t ed25519 -C "jenkins@company.com" -f ~/.ssh/jenkins_github
cat ~/.ssh/jenkins_github.pub
```

**Add public key to GitHub:**
- GitHub → Settings → SSH and GPG keys → New SSH key

**Add private key to Jenkins:**
1. **Manage Jenkins → Credentials → Add**
2. Kind: **SSH Username with private key**
3. Username: `git`
4. Private Key: Enter directly (paste content of `jenkins_github`)
5. ID: `github-ssh`

---

## 8.2 GitHub Webhooks

Webhooks tell Jenkins to build immediately when code is pushed — no polling needed.

```
┌──────────┐                        ┌──────────┐
│  GitHub  │─── POST /github-webhook/ ──▶│  Jenkins │
│          │    (on push/PR)         │          │
└──────────┘                        └──────────┘
```

### Setup Steps

**Step 1: Jenkins Configuration**
1. **Manage Jenkins → System → GitHub**
2. Click **Add GitHub Server**
3. Name: `GitHub`
4. API URL: `https://api.github.com`
5. Credentials: Select `github-token`
6. Click **Test connection** — should say "Credentials verified"

**Step 2: Create Webhook on GitHub**
1. Go to your repo → **Settings → Webhooks → Add webhook**
2. Fill in:

| Field | Value |
|-------|-------|
| Payload URL | `http://<jenkins-ip>:8080/github-webhook/` |
| Content type | `application/json` |
| Secret | (leave empty or set a secret) |
| Events | "Just the push event" (or "Send me everything" for PRs) |

3. Click **Add webhook**

**Step 3: Enable in Job**
- In your Pipeline job → Build Triggers → ✅ **GitHub hook trigger for GITScm polling**

> **Note:** Jenkins must be accessible from the internet for GitHub webhooks. For private networks, use ngrok for testing or GitHub Enterprise with VPN.

### Verifying Webhook

On GitHub → Settings → Webhooks → click your webhook:
- **Recent Deliveries** tab shows all webhook attempts
- ✅ Green check = successful delivery
- ❌ Red X = failed (check URL/network)

---

## 8.3 Multibranch Pipeline

A Multibranch Pipeline automatically discovers all branches with a `Jenkinsfile` and creates a job for each.

```
┌─────────────────────────────────────────────┐
│        Multibranch Pipeline Job              │
│        (landmark-web-app)                    │
├─────────────────────────────────────────────┤
│                                             │
│   ┌─────────┐ ┌─────────┐ ┌─────────────┐ │
│   │  main   │ │ develop │ │ release/1.0 │ │
│   │ (build) │ │ (build) │ │   (build)   │ │
│   └─────────┘ └─────────┘ └─────────────┘ │
│   ┌─────────┐ ┌─────────┐                  │
│   │feature/ │ │ hotfix/ │                  │
│   │login    │ │ fix-bug │                  │
│   └─────────┘ └─────────┘                  │
│                                             │
└─────────────────────────────────────────────┘
```

### Creating a Multibranch Pipeline

1. **Jenkins Dashboard → New Item**
2. Name: `landmark-web-app`
3. Select **Multibranch Pipeline** → OK

**Branch Sources:**
- Click **Add source → GitHub**
- Credentials: `github-token`
- Repository HTTPS URL: `https://github.com/CHAFAH/landmark-web-app.git`

**Build Configuration:**
- Mode: by Jenkinsfile
- Script Path: `Jenkinsfile`

**Scan Multibranch Pipeline Triggers:**
- ✅ Periodically if not otherwise run: `1 minute`
- Or rely on webhooks

**Orphaned Item Strategy:**
- Discard old items: ✅
- Days to keep: 7
- Max # to keep: 10

Click **Save**. Jenkins will scan the repo and create a job for each branch that has a `Jenkinsfile`.

### How env.BRANCH_NAME Works

In a Multibranch Pipeline, Jenkins sets `env.BRANCH_NAME` automatically:

```groovy
stage('Deploy') {
    when { branch 'main' }   // Only runs when building the 'main' branch
    steps {
        echo "Deploying from branch: ${env.BRANCH_NAME}"
    }
}
```

This is how our Landmark project's Jenkinsfile deploys to different environments:
- `develop` → dev namespace
- `release*` → staging namespace
- `main` → production namespace

---

## 8.4 Branch Build Strategy (Landmark Project)

```
┌─────────────────────────────────────────────────────────────┐
│                    BRANCHING STRATEGY                         │
│                                                             │
│  feature/* ──┐                                              │
│              ├──▶ develop ──▶ release/* ──▶ main            │
│  bugfix/*  ──┘                              ▲               │
│                                             │               │
│                              hotfix/* ──────┘               │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│  Branch         │ CI (test) │ Docker Push │ Deploy To       │
│─────────────────│───────────│─────────────│─────────────────│
│  feature/*      │    ✅     │     ❌      │     —           │
│  develop        │    ✅     │     ✅      │  dev namespace  │
│  release/*      │    ✅     │     ✅      │  staging ns     │
│  main           │    ✅     │     ✅      │  production ns  │
│  hotfix/*       │    ✅     │     ✅      │  production ns  │
└─────────────────────────────────────────────────────────────┘
```

---

## 8.5 Pull Request Builds

With the GitHub Integration plugin, Jenkins can:
- Build every PR automatically
- Report build status back to GitHub (✅ / ❌)
- Block PR merge until build passes

**GitHub → Settings → Branches → Branch protection rules:**
- Require status checks to pass: ✅
- Select: `continuous-integration/jenkins/pr-merge`

---

## Lab: Set Up Multibranch Pipeline with Webhooks

**Time:** 25 minutes

1. Create a **Multibranch Pipeline** job: `landmark-web-app`
2. Configure GitHub as the branch source
3. Point to: `https://github.com/CHAFAH/landmark-web-app.git`
4. Set Script Path: `Jenkinsfile`
5. Save and let Jenkins scan branches
6. Verify it finds `main`, `develop`, `release` (whichever exist)
7. Set up a webhook on GitHub pointing to your Jenkins
8. Push a commit and verify Jenkins triggers automatically

---

## Interview Questions

1. **Q:** How do you trigger Jenkins builds from GitHub?
   **A:** Configure a GitHub webhook pointing to `http://<jenkins>/github-webhook/`. In the Jenkins job, enable "GitHub hook trigger for GITScm polling". On push, GitHub sends a POST request to Jenkins which triggers the build.

2. **Q:** What is a Multibranch Pipeline?
   **A:** A Jenkins job type that automatically discovers all branches in a repository that contain a Jenkinsfile. It creates a separate sub-job for each branch and builds them according to the pipeline defined in the Jenkinsfile.

3. **Q:** How do you implement branch-based deployment in Jenkins?
   **A:** Use a Multibranch Pipeline with `when` conditions in the Jenkinsfile. For example: `when { branch 'main' }` for production, `when { branch 'develop' }` for dev environment.

---

## Summary

- Use PATs for HTTPS authentication; SSH keys for git-over-SSH
- GitHub webhooks provide instant build triggers (no polling delay)
- Multibranch Pipelines auto-discover branches and build each one
- `env.BRANCH_NAME` identifies which branch is building
- Use `when { branch 'pattern' }` for conditional deployments per branch
- Protect branches in GitHub to require passing Jenkins builds
