# MODULE 12: COMPLETE CI/CD PROJECT

## Learning Objectives

By the end of this module, students will be able to:
- Implement a production-grade CI/CD pipeline end-to-end
- Integrate all previous modules into a working deployment
- Deploy to Amazon EC2 using Docker
- Verify deployment health

---

## 12.1 Project Overview

We will build a complete pipeline for the Landmark Technologies web app:

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Checkout │───▶│ Install  │───▶│  Test    │───▶│  Build   │───▶│  Push    │───▶│  Deploy  │
│   Code   │    │  Deps    │    │          │    │  Docker  │    │  Docker  │    │  to EC2  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
```

---

## 12.2 Prerequisites Setup

### Credentials Required in Jenkins

| ID | Type | Purpose |
|----|------|---------|
| `github-token` | Secret text | GitHub PAT for checkout |
| `dockerhub-creds` | Username/Password | Docker Hub push |
| `deploy-server-ssh` | SSH Username with private key | SSH to deployment server |

### Deployment Server (EC2)

Provision a separate EC2 instance for deployment:

```bash
# On the deployment server
sudo dnf install docker -y
sudo systemctl start docker && sudo systemctl enable docker
sudo usermod -aG docker ec2-user
```

---

## 12.3 Complete Production Jenkinsfile

```groovy
pipeline {
    agent any
    tools {
        nodejs 'NodeJS-18'
    }
    environment {
        DOCKER_REPO = 'chafah/landmark-web-app'
        DEPLOY_SERVER = '10.0.1.100'  // Deployment EC2 private IP
        APP_PORT = '8080'
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    def branch = env.BRANCH_NAME.replaceAll('/', '-')
                    def timestamp = new Date().format('yyyyMMdd-HHmmss')
                    env.IMAGE_TAG = "${branch}-${timestamp}-${env.GIT_SHORT}"
                }
                echo "Building: ${DOCKER_REPO}:${IMAGE_TAG}"
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
                sh 'cd server && npm ci'
            }
        }

        stage('Run Tests') {
            parallel {
                stage('Frontend Tests') {
                    steps { sh 'npm test' }
                }
                stage('Backend Tests') {
                    steps { sh 'cd server && npm test' }
                }
            }
        }

        stage('Build Frontend') {
            steps { sh 'npm run build' }
        }

        stage('Docker Build & Push') {
            when {
                anyOf {
                    branch 'develop'
                    branch pattern: 'release*', comparator: 'GLOB'
                    branch 'main'
                }
            }
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
                        def app = docker.build("${DOCKER_REPO}:${IMAGE_TAG}")
                        app.push()
                        app.push('latest')
                    }
                }
            }
        }

        stage('Deploy to EC2') {
            when { branch 'main' }
            steps {
                sshagent(['deploy-server-ssh']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ec2-user@${DEPLOY_SERVER} << 'EOF'
                            # Pull latest image
                            docker pull ${DOCKER_REPO}:${IMAGE_TAG}

                            # Stop existing container
                            docker stop landmark-app || true
                            docker rm landmark-app || true

                            # Run new container
                            docker run -d \\
                                --name landmark-app \\
                                -p ${APP_PORT}:5000 \\
                                -e MONGO_URI=mongodb://mongo:27017/landmark \\
                                --restart unless-stopped \\
                                ${DOCKER_REPO}:${IMAGE_TAG}

                            # Verify
                            sleep 5
                            docker ps | grep landmark-app
EOF
                    """
                }
            }
        }

        stage('Health Check') {
            when { branch 'main' }
            steps {
                script {
                    def maxRetries = 5
                    def success = false
                    for (int i = 0; i < maxRetries; i++) {
                        def status = sh(
                            script: "curl -s -o /dev/null -w '%{http_code}' http://${DEPLOY_SERVER}:${APP_PORT}/api/students",
                            returnStdout: true
                        ).trim()
                        if (status == '200') {
                            echo "Health check passed! HTTP ${status}"
                            success = true
                            break
                        }
                        echo "Attempt ${i+1}/${maxRetries}: HTTP ${status}. Retrying..."
                        sleep 10
                    }
                    if (!success) {
                        error("Health check failed after ${maxRetries} attempts")
                    }
                }
            }
        }
    }

    post {
        success {
            echo """
            ✅ Pipeline Succeeded!
            Image: ${DOCKER_REPO}:${IMAGE_TAG}
            App URL: http://${DEPLOY_SERVER}:${APP_PORT}
            """
        }
        failure {
            echo '❌ Pipeline Failed! Check console output for details.'
        }
        always {
            cleanWs()
            sh "docker rmi ${DOCKER_REPO}:${IMAGE_TAG} || true"
        }
    }
}
```

---

## 12.4 Deployment Script (Alternative: Script-based Deploy)

Create `scripts/deploy.sh` in your repo:

```bash
#!/bin/bash
set -e

IMAGE=$1
APP_PORT=${2:-8080}

echo "=== Deploying ${IMAGE} ==="

# Pull image
docker pull ${IMAGE}

# Stop old container
docker stop landmark-app 2>/dev/null || true
docker rm landmark-app 2>/dev/null || true

# Start new container
docker run -d \
    --name landmark-app \
    -p ${APP_PORT}:5000 \
    -e MONGO_URI=mongodb://mongo:27017/landmark \
    --restart unless-stopped \
    ${IMAGE}

# Wait and verify
sleep 5
if docker ps | grep -q landmark-app; then
    echo "✅ Deployment successful!"
    docker logs --tail 5 landmark-app
else
    echo "❌ Deployment failed!"
    docker logs landmark-app
    exit 1
fi
```

---

## 12.5 Webhook Setup

1. GitHub repo → Settings → Webhooks → Add webhook
2. Payload URL: `http://<jenkins-ip>:8080/github-webhook/`
3. Content type: `application/json`
4. Events: Just the push event

---

## 12.6 Verification Checklist

After the pipeline runs successfully:

```bash
# On deployment server
docker ps                                    # Container running
curl http://localhost:8080/api/students       # API responds
curl http://localhost:8080                    # Frontend loads

# From your browser
# Open http://<deploy-server-public-ip>:8080
```

---

## Lab: End-to-End CI/CD

**Time:** 45 minutes

1. Set up all credentials in Jenkins (GitHub, Docker Hub, SSH)
2. Create a Multibranch Pipeline pointing to your fork
3. Configure the webhook on GitHub
4. Push a code change to `main`
5. Watch the pipeline execute all stages
6. Verify the application is accessible on the deployment server
7. Make another change and verify auto-deployment

---

## Interview Questions

1. **Q:** Walk me through your CI/CD pipeline.
   **A:** On every push, Jenkins checks out code, installs dependencies, runs parallel tests (frontend + backend), builds the React frontend, builds a Docker image tagged with branch-timestamp-commit, pushes to Docker Hub, then deploys by SSH-ing into the server, pulling the image, and running it. A health check verifies the deployment succeeded.

2. **Q:** How do you handle rollbacks?
   **A:** Each image has a unique tag (branch-timestamp-commit). To rollback, re-deploy the previous tag. With Docker, it's: `docker stop app && docker run <previous-tag>`.

3. **Q:** How do you verify a deployment is healthy?
   **A:** Implement a health check stage that curls an endpoint (e.g., `/api/students`) and retries with backoff. If it fails after N attempts, mark the build as failed and trigger a rollback.

---

## Summary

- A production CI/CD pipeline: checkout → test → build → push → deploy → verify
- Use parallel stages for concurrent testing
- Tag images uniquely (branch + timestamp + commit) for traceability
- Deploy via SSH + Docker pull/run for EC2 deployments
- Always include a health check after deployment
- Use `sshagent` for secure SSH key injection
