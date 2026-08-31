# MODULE 5: JENKINS PLUGINS

## Learning Objectives

By the end of this module, students will be able to:
- Explain the Jenkins plugin architecture
- Install and configure essential plugins for CI/CD
- Troubleshoot common plugin issues
- Identify which plugins install tools vs provide DSL only

---

## 5.1 What Are Plugins?

Plugins extend Jenkins functionality. Without plugins, Jenkins can only run shell commands. Plugins add:
- SCM integrations (Git, SVN)
- Build tools (Maven, Node.js, Docker)
- Notifications (Slack, Email)
- Authentication (LDAP, OAuth)
- Pipeline DSL features
- Cloud integrations (AWS, Azure, GCP)

### Plugin Architecture

```
┌─────────────────────────────────────────┐
│           JENKINS CORE                   │
│  (Build engine, UI, API, Scheduling)     │
├─────────────────────────────────────────┤
│         PLUGIN LAYER                     │
│                                         │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐     │
│  │ Git │ │Node │ │Docker│ │ AWS │     │
│  └─────┘ └─────┘ └─────┘ └─────┘     │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐     │
│  │Slack│ │Blue │ │K8s  │ │RBAC │     │
│  │     │ │Ocean│ │ CLI │ │     │     │
│  └─────┘ └─────┘ └─────┘ └─────┘     │
│                                         │
└─────────────────────────────────────────┘
```

### Plugin Lifecycle

| State | Description |
|-------|-------------|
| Available | In the update center, not installed |
| Installed | Downloaded and active |
| Disabled | Installed but not active |
| Requires Restart | Installed but needs Jenkins restart |
| Update Available | Newer version exists |
| Deprecated | Maintained but may be removed |
| Detached | Was part of core, now separate |

### Plugin File Types

- `.jpi` or `.hpi` — Plugin archive files
- Location: `/var/lib/jenkins/plugins/`
- Each plugin has a matching directory: `plugins/git/` (extracted)

---

## 5.2 Essential Plugins for This Project

### Plugins That INSTALL Tools (No Server Install Needed)

| Plugin | Provides | Pipeline Usage |
|--------|----------|----------------|
| **NodeJS** | Node.js binary | `tools { nodejs 'NodeJS-18' }` |
| **Kubernetes CLI** | kubectl binary | `withKubeConfig([credentialsId: '...'])` |

### Plugins That Provide DSL Only (Server Install Still Needed)

| Plugin | Provides | Requires on Server |
|--------|----------|-------------------|
| **Docker Pipeline** | `docker.build()`, `docker.withRegistry()` | Docker daemon |
| **Pipeline: AWS Steps** | `withAWS(credentials: '...')` | AWS CLI binary |

### Complete Plugin List for Landmark Project

| Plugin | Purpose |
|--------|---------|
| **Pipeline** | Core pipeline support |
| **Multibranch Pipeline** | Auto-discover branches |
| **Git** | Git SCM integration |
| **GitHub Integration** | Webhooks & PR status |
| **NodeJS** | Node.js build environment (auto-installs) |
| **Docker Pipeline** | Docker build/push DSL |
| **Docker** | Docker agent support |
| **Kubernetes CLI** | kubectl (auto-installs) |
| **Pipeline: AWS Steps** | AWS credential injection |
| **AWS Credentials** | AWS credential type |
| **Credentials Binding** | Use credentials in pipelines |
| **Blue Ocean** | Modern pipeline UI |
| **Workspace Cleanup** | Clean workspace between builds |
| **Role Strategy** | Role-based access control |

---

## 5.3 Plugin Installation and Configuration

### Git Plugin

**Purpose:** Clone Git repositories in pipelines.

**Install:** Usually included with suggested plugins.

**Usage in Pipeline:**
```groovy
stage('Checkout') {
    steps {
        git branch: 'main',
            url: 'https://github.com/CHAFAH/landmark-web-app.git',
            credentialsId: 'github-token'
    }
}
// OR simply:
stage('Checkout') {
    steps { checkout scm }  // Uses SCM configured in job
}
```

### GitHub Integration Plugin

**Purpose:** Webhooks, commit status updates, PR builds.

**Configuration:**
1. Install plugin
2. **Manage Jenkins → System → GitHub → Add GitHub Server**
3. Add credential (Secret text = GitHub PAT)
4. Test connection

### NodeJS Plugin

**Purpose:** Auto-installs Node.js — no server install needed.

**Configuration:**
1. **Manage Jenkins → Tools → NodeJS installations → Add**
2. Name: `NodeJS-18`
3. Install automatically: ✅
4. Version: 18.x

**Usage:**
```groovy
pipeline {
    tools { nodejs 'NodeJS-18' }
    stages {
        stage('Build') {
            steps { sh 'npm ci && npm run build' }
        }
    }
}
```

### Docker Pipeline Plugin

**Purpose:** Groovy DSL for Docker operations.

**Prerequisite:** Docker must be installed on the server.

**Usage:**
```groovy
stage('Docker Build & Push') {
    steps {
        script {
            docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
                def app = docker.build("chafah/landmark-web-app:${IMAGE_TAG}")
                app.push()
            }
        }
    }
}
```

### Kubernetes CLI Plugin

**Purpose:** Provides kubectl binary and `withKubeConfig` step.

**Usage:**
```groovy
stage('Deploy') {
    steps {
        withKubeConfig([credentialsId: 'kubeconfig']) {
            sh 'kubectl apply -f k8s/'
        }
    }
}
```

### Pipeline: AWS Steps Plugin

**Purpose:** Injects AWS credentials as environment variables.

**Prerequisite:** AWS CLI must be installed on server.

**Usage:**
```groovy
stage('Deploy') {
    steps {
        withAWS(credentials: 'aws-creds', region: 'us-east-1') {
            sh 'aws eks update-kubeconfig --name landmark-eks'
        }
    }
}
```

### Blue Ocean

**Purpose:** Modern visual pipeline editor and UI.

**Access:** Click "Open Blue Ocean" in Jenkins sidebar after installation.

### Credentials Binding Plugin

**Purpose:** Inject credentials into pipeline as environment variables.

**Usage:**
```groovy
withCredentials([usernamePassword(
    credentialsId: 'dockerhub-creds',
    usernameVariable: 'DOCKER_USER',
    passwordVariable: 'DOCKER_PASS'
)]) {
    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
}
```

---

## 5.4 Common Plugin Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Plugin install fails | Network/proxy | Check internet access; configure proxy in Manage Jenkins → Plugins → Advanced |
| Plugin dependency error | Missing required plugin | Install the dependency plugin first |
| Pipeline step not found | Plugin not installed or needs restart | Install plugin and restart Jenkins |
| Plugin breaks after update | Incompatible version | Downgrade: download specific version from plugins.jenkins.io |
| Too many plugins slow Jenkins | Memory usage | Remove unused plugins, increase heap |

---

## Lab: Install Project Plugins

**Time:** 15 minutes

1. Go to **Manage Jenkins → Plugins → Available**
2. Install all plugins from the table in section 5.2
3. Restart Jenkins
4. Configure NodeJS 18 in **Manage Jenkins → Tools**
5. Verify by creating a test pipeline that runs `node --version`

---

## Summary

- Plugins extend Jenkins with integrations, tools, and features
- Some plugins auto-install tools (NodeJS, Kubernetes CLI); others only provide pipeline DSL (Docker Pipeline, AWS Steps)
- For the Landmark project, you need ~14 plugins
- Always restart Jenkins after plugin installation
- Test plugin updates in non-production environments first
