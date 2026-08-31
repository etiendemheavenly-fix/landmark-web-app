# MODULE 1: INTRODUCTION TO JENKINS

## Learning Objectives

By the end of this module, students will be able to:
- Define Jenkins and explain its role in CI/CD
- Describe the Jenkins architecture (Controller/Agent model)
- Differentiate between CI, CD, and Continuous Deployment
- Explain why Jenkins remains the most widely-used automation server
- Identify common Jenkins use cases in enterprise environments

---

## 1.1 What is Jenkins?

Jenkins is an open-source automation server written in Java. It helps automate the parts of software development related to building, testing, and deploying — facilitating continuous integration and continuous delivery (CI/CD).

**Key Facts:**
- Written in Java (runs on JVM)
- Over 1,800 plugins available
- Used by over 300,000 installations worldwide
- Supports any programming language
- Platform-independent (Linux, Windows, macOS)
- Free and open-source (MIT License)

---

## 1.2 History of Jenkins

| Year | Event |
|------|-------|
| 2004 | Kohsuke Kawaguchi creates **Hudson** at Sun Microsystems |
| 2005 | Hudson released as open-source |
| 2008 | Hudson becomes the leading CI server |
| 2010 | Oracle acquires Sun Microsystems |
| 2011 | Community forks Hudson → **Jenkins** (trademark dispute) |
| 2011 | Jenkins 1.0 released |
| 2016 | Jenkins 2.0 released (Pipeline as Code, Blue Ocean) |
| 2019 | Jenkins X released (cloud-native) |
| 2023 | Jenkins remains #1 CI/CD tool by market share |

> **Instructor Note:** The fork happened because Oracle wanted control over the Hudson trademark. The community voted overwhelmingly (214 vs 14) to rename to Jenkins and move development to GitHub.

---

## 1.3 Jenkins Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        JENKINS CONTROLLER                            │
│                                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │ Web UI   │  │ REST API │  │ Scheduler│  │ Plugin Manager   │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────────┘   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │ Job Mgr  │  │ Security │  │  Queue   │  │ Credential Store │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────────┘   │
│                                                                     │
└───────────────────────────┬─────────────────────────────────────────┘
                            │ (SSH / JNLP / WebSocket)
              ┌─────────────┼─────────────┐
              │             │             │
              ▼             ▼             ▼
     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
     │   AGENT 1    │ │   AGENT 2    │ │   AGENT 3    │
     │ (Linux/Docker)│ │  (Windows)   │ │  (macOS)     │
     │              │ │              │ │              │
     │ Executor 1   │ │ Executor 1   │ │ Executor 1   │
     │ Executor 2   │ │ Executor 2   │ │ Executor 2   │
     └──────────────┘ └──────────────┘ └──────────────┘
```

### Controller (formerly "Master")

The Controller is the central coordination server:

- Hosts the web UI
- Manages configuration
- Schedules builds
- Dispatches builds to agents
- Monitors agents
- Records build results
- Serves the REST API

> **Production Consideration:** Never run builds on the Controller in production. It should only coordinate work.

### Agent (formerly "Slave")

Agents are machines that execute builds:

- Connect to Controller via SSH, JNLP, or WebSocket
- Execute build steps
- Can run on any OS
- Can be static (always running) or dynamic (spun up on demand)
- Have one or more **Executors** (build slots)

### Executor

An executor is a computational resource for running builds:

- Each agent has a configurable number of executors
- One executor = one concurrent build
- Rule of thumb: set executors = number of CPU cores

---

## 1.4 CI vs CD vs Continuous Deployment

```
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                            │
│   Developer      CI Server        Staging          Production              │
│                                                                            │
│   ┌──────┐      ┌────────┐      ┌─────────┐      ┌────────────┐          │
│   │ Code │─────▶│ Build  │─────▶│ Deploy  │─────▶│  Deploy    │          │
│   │ Push │      │ & Test │      │ & Test  │      │  (Manual/  │          │
│   └──────┘      └────────┘      └─────────┘      │  Auto)     │          │
│                                                    └────────────┘          │
│   ◀──── CI ────▶ ◀──── Continuous Delivery ────▶                          │
│   ◀──────────── Continuous Deployment ──────────────────────▶             │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

| Concept | Definition | Manual Gate? |
|---------|-----------|:------------:|
| **Continuous Integration (CI)** | Automatically build and test code on every commit | No gate needed |
| **Continuous Delivery (CD)** | CI + automatically deploy to staging; production deploy requires approval | ✅ Manual approval |
| **Continuous Deployment** | CI + automatically deploy to ALL environments including production | ❌ Fully automated |

### Real-World Example (Landmark Technologies Project)

In our project repository:
- **CI:** Every push triggers `npm ci`, `npm test`, and `docker build`
- **CD:** Pushes to `develop` auto-deploy to dev namespace; pushes to `release*` auto-deploy to staging
- **Continuous Deployment:** Pushes to `main` auto-deploy to production (could add manual approval)

---

## 1.5 Why Jenkins Remains Relevant

Despite newer tools (GitHub Actions, GitLab CI, CircleCI), Jenkins remains dominant because:

1. **Self-hosted** — Full control over infrastructure, data, and security
2. **Plugin ecosystem** — 1,800+ plugins for any integration
3. **Flexibility** — Not opinionated; adapts to any workflow
4. **Enterprise adoption** — Massive existing install base
5. **On-premise compliance** — Required for regulated industries (banking, healthcare)
6. **Multi-platform** — Builds for Linux, Windows, macOS, embedded
7. **Cost** — Free for unlimited users, jobs, and build minutes
8. **Maturity** — Battle-tested for 15+ years

### When NOT to Use Jenkins

- Small teams with simple workflows → GitHub Actions
- GitLab-native shops → GitLab CI
- Fully cloud-native with no on-prem requirements → Managed CI services

---

## 1.6 Jenkins Use Cases

| Use Case | Description |
|----------|-------------|
| CI/CD Pipelines | Build, test, deploy applications |
| Infrastructure as Code | Run Terraform, CloudFormation, Ansible |
| Scheduled Jobs | Nightly builds, database backups, report generation |
| Multi-branch Builds | Build every branch and PR automatically |
| Release Management | Manage versioning, tagging, changelog generation |
| Security Scanning | Integrate SAST/DAST tools into pipelines |
| Performance Testing | Run JMeter, Gatling tests |
| Machine Learning | ML model training pipelines |

---

## 1.7 Jenkins Ecosystem

```
┌──────────────────────────────────────────────────────┐
│                 JENKINS ECOSYSTEM                      │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Core:        Jenkins Controller + Agents            │
│  UI:          Classic UI, Blue Ocean                 │
│  Config:      Jenkinsfile (Pipeline as Code)         │
│  Plugins:     1,800+ (Git, Docker, K8s, AWS...)     │
│  API:         REST API, CLI                          │
│  Cloud:       Jenkins X (Kubernetes-native)          │
│  Community:   jenkins.io, GitHub, Gitter             │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## Quiz Questions

1. What language is Jenkins written in?
2. What was Jenkins originally called before 2011?
3. What is the difference between the Controller and an Agent?
4. How many executors should an agent typically have?
5. What is the difference between Continuous Delivery and Continuous Deployment?
6. Name three reasons Jenkins remains relevant despite newer CI/CD tools.
7. What protocol(s) can agents use to connect to the Controller?

---

## Interview Questions

1. **Q:** Explain the Jenkins architecture.
   **A:** Jenkins uses a Controller-Agent architecture. The Controller manages configuration, scheduling, and the UI. Agents are worker machines that execute builds. They connect via SSH, JNLP, or WebSocket. Each agent has executors (build slots) that run concurrent jobs.

2. **Q:** When would you choose Jenkins over GitHub Actions?
   **A:** When you need self-hosted infrastructure for compliance, when you have complex multi-platform builds, when you need deep enterprise integrations, or when you want to avoid per-minute billing.

3. **Q:** What is Pipeline as Code?
   **A:** Defining your CI/CD pipeline in a Jenkinsfile stored in version control alongside your application code. This enables versioning, code review, and reproducibility of your build process.

---

## Summary

- Jenkins is a free, open-source automation server for CI/CD
- It uses a Controller-Agent architecture for distributed builds
- CI = build/test on every commit; CD = deploy to staging automatically; Continuous Deployment = deploy to production automatically
- Jenkins remains relevant due to its flexibility, plugin ecosystem, and enterprise adoption
- Our project (Landmark Technologies) uses Jenkins with a multi-branch pipeline deploying to Kubernetes

---

## Homework

1. Research three companies that use Jenkins in production and document their use case
2. Draw a diagram showing how Jenkins would fit into your organization's software delivery process
3. Compare Jenkins with one cloud-native CI tool (GitHub Actions or GitLab CI) — list 5 pros and 5 cons of each
