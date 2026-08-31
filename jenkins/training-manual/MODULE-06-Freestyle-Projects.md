# MODULE 6: JENKINS FREESTYLE PROJECTS

## Learning Objectives

By the end of this module, students will be able to:
- Create and configure Freestyle jobs
- Set up build triggers (polling, webhooks, manual)
- Integrate Git/GitHub as SCM
- Configure post-build actions
- Understand when to use Freestyle vs Pipeline

---

## 6.1 What is a Freestyle Project?

A Freestyle project is Jenkins' original job type. It provides a web-based form to configure builds without writing code.

**When to use Freestyle:**
- Simple build-and-deploy tasks
- Running shell scripts on a schedule
- Quick prototypes
- Teams not ready for Pipeline as Code

**When to use Pipeline instead:**
- Complex multi-stage builds
- Version-controlled build config
- Parallel execution
- Conditional logic
- Code review of build changes

---

## 6.2 Creating a Freestyle Job

### Step-by-Step

1. **Jenkins Dashboard → New Item**
2. Enter name: `landmark-freestyle-build`
3. Select **Freestyle project** → OK

### General Configuration

| Field | Value |
|-------|-------|
| Description | "Build and test Landmark web app" |
| Discard old builds | ✅ Max # of builds to keep: 10 |
| GitHub project | `https://github.com/CHAFAH/landmark-web-app/` |

### Source Code Management

| Field | Value |
|-------|-------|
| Git | ✅ |
| Repository URL | `https://github.com/CHAFAH/landmark-web-app.git` |
| Credentials | github-token |
| Branch | `*/main` |

### Build Triggers

| Trigger | Description | Configuration |
|---------|-------------|---------------|
| Poll SCM | Jenkins checks Git on a schedule | `H/5 * * * *` (every 5 min) |
| GitHub hook trigger | GitHub pushes trigger build | Requires webhook setup |
| Build periodically | Cron schedule regardless of changes | `H 2 * * *` (daily at 2am) |
| Trigger remotely | API call triggers build | Set authentication token |

**Poll SCM Schedule Syntax (Cron):**
```
H/5 * * * *    # Every 5 minutes
H * * * *      # Every hour
H 2 * * *      # Daily at 2 AM
H 2 * * 1-5    # Weekdays at 2 AM
```

> **Note:** `H` is a Jenkins hash that spreads load. `H/5` means "every 5 minutes but at a consistent offset."

### Build Environment

| Option | Purpose |
|--------|---------|
| Delete workspace before build | Fresh start each time |
| Provide Node & npm bin/folder to PATH | Adds NodeJS (requires NodeJS plugin) |
| Use secret text(s) or file(s) | Inject credentials |

Select: **Provide Node & npm bin/folder to PATH** → NodeJS Installation: `NodeJS-18`

### Build Steps

Click **Add build step → Execute shell**:

```bash
#!/bin/bash
echo "=== Installing Dependencies ==="
npm ci

echo "=== Running Tests ==="
npm test

echo "=== Building Frontend ==="
npm run build

echo "=== Build Complete ==="
echo "Build Number: ${BUILD_NUMBER}"
echo "Branch: ${GIT_BRANCH}"
echo "Commit: ${GIT_COMMIT}"
```

### Post-Build Actions

| Action | Purpose |
|--------|---------|
| Archive the artifacts | Save `build/` directory |
| Email notification | Notify on failure |
| Publish JUnit test results | Display test results |
| Build other projects | Trigger downstream jobs |

**Archive artifacts:** Pattern: `build/**/*`

---

## 6.3 Built-in Environment Variables

Available in any Freestyle job:

| Variable | Description | Example |
|----------|-------------|---------|
| `BUILD_NUMBER` | Current build number | `42` |
| `BUILD_ID` | Build ID | `42` |
| `JOB_NAME` | Name of the job | `landmark-freestyle-build` |
| `WORKSPACE` | Absolute path of workspace | `/var/lib/jenkins/workspace/landmark-freestyle-build` |
| `JENKINS_URL` | Jenkins server URL | `http://localhost:8080/` |
| `GIT_BRANCH` | Git branch being built | `origin/main` |
| `GIT_COMMIT` | Git commit hash | `a1b2c3d4...` |
| `BUILD_URL` | Full URL to this build | `http://localhost:8080/job/landmark/42/` |

---

## 6.4 Chaining Freestyle Jobs

You can chain jobs together:

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Build   │────▶│   Test   │────▶│  Deploy  │
│  Job     │     │   Job    │     │   Job    │
└──────────┘     └──────────┘     └──────────┘
```

**Configuration:**
- In "Build" job → Post-build Actions → **Build other projects**
- Projects to build: `test-job`
- Trigger only if build is stable: ✅

> **Limitation:** This is why Pipelines exist — chaining Freestyle jobs is brittle, hard to maintain, and not version-controlled.

---

## 6.5 Freestyle vs Pipeline Comparison

| Feature | Freestyle | Pipeline |
|---------|-----------|----------|
| Configuration | Web UI forms | Code (Jenkinsfile) |
| Version control | ❌ | ✅ |
| Complex logic | ❌ | ✅ |
| Parallel stages | ❌ | ✅ |
| Code review | ❌ | ✅ (PR on Jenkinsfile) |
| Visualization | Basic | Blue Ocean |
| Restart from stage | ❌ | ✅ |
| Shared libraries | ❌ | ✅ |
| Learning curve | Low | Medium |

---

## Lab: Create a Freestyle Job for Landmark App

**Time:** 20 minutes

1. Create a new Freestyle job: `landmark-build-test`
2. Configure Git SCM: `https://github.com/CHAFAH/landmark-web-app.git`
3. Set Build Environment: Provide Node & npm → NodeJS-18
4. Add Execute shell step:
   ```bash
   npm ci
   npm test
   npm run build
   echo "SUCCESS: Build #${BUILD_NUMBER}"
   ```
5. Set Post-build: Archive artifacts `build/**/*`
6. Set Build Trigger: Poll SCM `H/5 * * * *`
7. Click **Build Now**
8. Verify the console output shows tests passing

---

## Interview Questions

1. **Q:** What is the difference between Freestyle and Pipeline jobs?
   **A:** Freestyle jobs are configured via the web UI and are limited to simple sequential builds. Pipeline jobs are defined in code (Jenkinsfile), support complex logic, parallel stages, conditionals, and are version-controlled.

2. **Q:** When would you still use a Freestyle job?
   **A:** For simple scheduled tasks (backups, cleanup scripts), quick prototypes, or teams transitioning to Jenkins who aren't ready for Pipeline as Code yet.

---

## Summary

- Freestyle projects are UI-configured jobs for simple builds
- They support Git integration, build triggers, shell steps, and post-build actions
- Pipelines are preferred for production CI/CD due to version control and flexibility
- Freestyle jobs are still useful for learning Jenkins fundamentals and simple tasks
