# MODULE 7: JENKINS PIPELINES

## Learning Objectives

By the end of this module, students will be able to:
- Write Declarative and Scripted pipelines
- Use stages, steps, agents, environment, parameters, when conditions, and post actions
- Understand Pipeline as Code and the Jenkinsfile
- Choose between Declarative and Scripted syntax

---

## 7.1 Pipeline as Code

A **Jenkinsfile** is a text file stored in your repository that defines the entire build pipeline.

**Benefits:**
- Version-controlled alongside application code
- Code-reviewed via pull requests
- Reproducible builds
- Survives Jenkins server replacement
- Self-documenting

**Location:** Root of your repository → `Jenkinsfile` (no extension)

---

## 7.2 Declarative Pipeline (Recommended)

Structured, opinionated syntax. Easier to read, write, and maintain.

### Complete Syntax Reference

```groovy
pipeline {
    agent any                          // WHERE to run

    tools {                            // AUTO-INSTALL tools
        nodejs 'NodeJS-18'
        maven 'Maven-3.9'
    }

    environment {                      // ENVIRONMENT VARIABLES
        APP_NAME = 'landmark-web-app'
        VERSION = '1.0.0'
        DOCKER_REPO = 'chafah/landmark-web-app'
    }

    parameters {                       // USER INPUT
        string(name: 'BRANCH', defaultValue: 'main', description: 'Branch to build')
        choice(name: 'ENV', choices: ['dev', 'staging', 'prod'], description: 'Target env')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip tests?')
    }

    options {                          // JOB OPTIONS
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    triggers {                         // AUTO-TRIGGER
        pollSCM('H/5 * * * *')
        cron('H 2 * * *')
    }

    stages {
        stage('Build') {               // NAMED PHASE
            steps {                    // COMMANDS
                sh 'npm ci'
                sh 'npm run build'
            }
        }

        stage('Test') {
            when {                     // CONDITIONAL
                not { params.SKIP_TESTS }
            }
            steps {
                sh 'npm test'
            }
        }

        stage('Deploy') {
            when {
                branch 'main'          // Only on main branch
            }
            input {                    // MANUAL APPROVAL
                message 'Deploy to production?'
                ok 'Deploy'
            }
            steps {
                sh './deploy.sh'
            }
        }
    }

    post {                             // AFTER ALL STAGES
        always  { cleanWs() }
        success { echo 'Build succeeded!' }
        failure { echo 'Build failed!' }
        unstable { echo 'Build unstable!' }
    }
}
```

---

## 7.3 Scripted Pipeline

Full Groovy scripting. More flexible but harder to read.

```groovy
node {
    def imageTag = ''

    try {
        stage('Checkout') {
            checkout scm
        }

        stage('Build') {
            def nodeHome = tool name: 'NodeJS-18', type: 'nodejs'
            env.PATH = "${nodeHome}/bin:${env.PATH}"
            sh 'npm ci'
            sh 'npm run build'
        }

        stage('Test') {
            sh 'npm test'
        }

        stage('Docker') {
            if (env.BRANCH_NAME == 'main') {
                def branch = env.BRANCH_NAME.replaceAll('/', '-')
                def timestamp = new Date().format('yyyyMMdd-HHmmss')
                imageTag = "${branch}-${timestamp}"

                docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
                    def app = docker.build("chafah/landmark-web-app:${imageTag}")
                    app.push()
                }
            }
        }

        stage('Deploy') {
            if (env.BRANCH_NAME == 'main') {
                sh "kubectl set image deployment/landmark-app landmark-app=chafah/landmark-web-app:${imageTag}"
            }
        }

    } catch (err) {
        currentBuild.result = 'FAILURE'
        throw err
    } finally {
        cleanWs()
    }
}
```

---

## 7.4 Declarative vs Scripted

| Feature | Declarative | Scripted |
|---------|-------------|----------|
| Syntax | Structured blocks | Free-form Groovy |
| Starts with | `pipeline { }` | `node { }` |
| Error handling | `post { failure {} }` | `try/catch/finally` |
| Conditionals | `when { }` directive | `if/else` statements |
| Agent assignment | Per-pipeline or per-stage | `node('label') { }` |
| Restart from stage | ✅ Supported | ❌ Not supported |
| Validation | Linted before execution | No pre-validation |
| Recommended for | 95% of use cases | Complex dynamic logic |

> **Best Practice:** Start with Declarative. Only use Scripted when you need something Declarative cannot express (rare).

---

## 7.5 Key Directives Deep Dive

### agent

```groovy
// Run on any available agent
agent any

// Run on agent with specific label
agent { label 'docker' }

// Run in a Docker container
agent {
    docker { image 'node:18' }
}

// No global agent — define per stage
agent none
```

### environment

```groovy
environment {
    // Static values
    APP_NAME = 'landmark'

    // From credentials
    DOCKER_CREDS = credentials('dockerhub-creds')
    // Creates: DOCKER_CREDS_USR and DOCKER_CREDS_PSW

    // From shell command
    GIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
}
```

### parameters

```groovy
parameters {
    string(name: 'TAG', defaultValue: 'latest', description: 'Docker image tag')
    choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'production'])
    booleanParam(name: 'DEPLOY', defaultValue: true)
    password(name: 'SECRET', description: 'A secret value')
}

// Access in steps:
steps {
    sh "echo Deploying tag: ${params.TAG} to ${params.ENVIRONMENT}"
}
```

### when (Conditions)

```groovy
// Branch condition
when { branch 'main' }
when { branch pattern: 'release*', comparator: 'GLOB' }

// Environment condition
when { environment name: 'ENV', value: 'production' }

// Expression (Groovy boolean)
when { expression { return params.DEPLOY == true } }

// Multiple conditions (AND)
when {
    allOf {
        branch 'main'
        environment name: 'DEPLOY', value: 'true'
    }
}

// Multiple conditions (OR)
when {
    anyOf {
        branch 'main'
        branch pattern: 'hotfix*', comparator: 'GLOB'
    }
}

// Not condition
when { not { branch 'develop' } }
```

### post

```groovy
post {
    always {
        // Runs regardless of result
        cleanWs()
        junit '**/test-results.xml'
    }
    success {
        echo 'Build succeeded!'
        slackSend color: 'good', message: "Build passed"
    }
    failure {
        echo 'Build failed!'
        slackSend color: 'danger', message: "Build failed"
    }
    unstable {
        echo 'Build unstable (test failures)'
    }
    changed {
        echo 'Build status changed from last build'
    }
}
```

### Parallel Stages

```groovy
stage('Tests') {
    parallel {
        stage('Frontend Tests') {
            steps { sh 'npm test' }
        }
        stage('Server Tests') {
            steps { sh 'cd server && npm test' }
        }
        stage('Lint') {
            steps { sh 'npm run lint' }
        }
    }
}
```

---

## 7.6 Landmark Project Pipeline (Explained)

```groovy
pipeline {
    agent any                              // Run on any available agent
    tools {
        nodejs 'NodeJS-18'                 // Auto-install Node.js 18
    }
    environment {
        DOCKER_REPO = 'chafah/landmark-web-app'
        AWS_REGION = 'us-east-1'
        EKS_CLUSTER = 'landmark-eks'
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }         // Clone the repo
        }
        stage('Install & Test') {
            steps {
                sh 'npm ci'                // Install frontend deps (exact)
                sh 'npm test'              // Run frontend tests
                sh 'cd server && npm ci && npm test'  // Server tests
            }
        }
        stage('Build Frontend') {
            steps { sh 'npm run build' }   // Create production React build
        }
        stage('Generate Image Tag') {
            steps {
                script {
                    // Tag format: branch-YYYYMMDD-HHMMSS
                    def branch = env.BRANCH_NAME.replaceAll('/', '-')
                    def timestamp = new Date().format('yyyyMMdd-HHmmss')
                    env.IMAGE_TAG = "${branch}-${timestamp}"
                }
            }
        }
        stage('Docker Build & Push') {
            when {                         // Only on deploy branches
                anyOf {
                    branch 'develop'
                    branch pattern: 'release*', comparator: 'GLOB'
                    branch 'main'
                    branch pattern: 'hotfix*', comparator: 'GLOB'
                }
            }
            steps {
                script {
                    // Docker plugin handles login/logout automatically
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
                        def app = docker.build("${DOCKER_REPO}:${IMAGE_TAG}")
                        app.push()
                    }
                }
            }
        }
        stage('Deploy to Production') {
            when {
                anyOf {
                    branch 'main'
                    branch pattern: 'hotfix*', comparator: 'GLOB'
                }
            }
            steps {
                withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {
                    sh """
                        aws eks update-kubeconfig --name ${EKS_CLUSTER} --region ${AWS_REGION}
                        sed -i 's/name: landmark/name: production/g' k8s/namespace.yml
                        sed -i 's/namespace: landmark/namespace: production/g' k8s/*.yml
                        sed -i "s|image: landmark-technologies:latest|image: ${DOCKER_REPO}:${IMAGE_TAG}|g" k8s/app-deployment.yml
                        kubectl apply -f k8s/namespace.yml
                        kubectl apply -f k8s/
                    """
                }
            }
        }
    }
    post {
        success { echo 'Pipeline succeeded!' }
        failure { echo 'Pipeline failed!' }
        always  { cleanWs() }              // Clean workspace
    }
}
```

---

## Lab: Write Your First Pipeline

**Time:** 25 minutes

1. Create a new Pipeline job: `landmark-pipeline-test`
2. Choose "Pipeline script" (not SCM)
3. Paste this starter pipeline:

```groovy
pipeline {
    agent any
    tools {
        nodejs 'NodeJS-18'
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/CHAFAH/landmark-web-app.git'
            }
        }
        stage('Install') {
            steps { sh 'npm ci' }
        }
        stage('Test') {
            steps { sh 'npm test' }
        }
        stage('Build') {
            steps { sh 'npm run build' }
        }
    }
    post {
        success { echo '✅ All stages passed!' }
        failure { echo '❌ Pipeline failed!' }
    }
}
```

4. Click **Build Now**
5. View the stage visualization
6. Click into each stage to see console output

---

## Interview Questions

1. **Q:** What is the difference between Declarative and Scripted pipelines?
   **A:** Declarative uses a structured `pipeline {}` block with predefined directives (agent, stages, post). Scripted uses `node {}` with free-form Groovy. Declarative is validated before execution and supports restart-from-stage. Scripted offers more flexibility for complex dynamic logic.

2. **Q:** How do you make a stage run only on the main branch?
   **A:** Use the `when` directive: `when { branch 'main' }`.

3. **Q:** How do you handle credentials in a pipeline?
   **A:** Use `withCredentials()` or `credentials()` in the environment block. Never hardcode secrets.

---

## Summary

- Pipelines are defined in a Jenkinsfile stored in your repository
- Declarative pipelines (`pipeline {}`) are recommended for most use cases
- Key directives: agent, tools, environment, parameters, stages, when, post
- Use `when` for conditional stage execution
- Use `post` for cleanup and notifications
- Use `parallel` for concurrent test execution
