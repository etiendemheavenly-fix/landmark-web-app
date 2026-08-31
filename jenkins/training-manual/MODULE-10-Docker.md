# MODULE 10: DOCKER WITH JENKINS

## Learning Objectives

By the end of this module, students will be able to:
- Install Docker on a Jenkins server
- Configure Docker permissions for Jenkins
- Use the Docker Pipeline plugin
- Build, tag, and push images in pipelines
- Use Docker containers as build agents

---

## 10.1 Installing Docker on Jenkins Server

```bash
# Install Docker
sudo dnf install docker -y            # Amazon Linux 2023
# OR
sudo yum install docker -y            # Amazon Linux 2

# Start and enable
sudo systemctl start docker
sudo systemctl enable docker

# Add Jenkins user to docker group
sudo usermod -aG docker jenkins

# CRITICAL: Restart Jenkins to pick up group change
sudo systemctl restart jenkins

# Verify
sudo -u jenkins docker ps
```

> **Common Error:** If you skip `usermod -aG docker jenkins`, pipelines will fail with `permission denied while trying to connect to the Docker daemon socket`.

---

## 10.2 Docker Pipeline Plugin

The plugin provides Groovy DSL — it does NOT install Docker.

### Key Methods

| Method | Purpose |
|--------|---------|
| `docker.build(image:tag)` | Build an image |
| `docker.withRegistry(url, credId)` | Login/logout around a block |
| `image.push()` | Push image to registry |
| `image.push('tag')` | Push with specific tag |
| `docker.image(name).pull()` | Pull an image |
| `docker.image(name).run(args)` | Run a container |

---

## 10.3 Complete Docker Pipeline (Landmark Project)

```groovy
pipeline {
    agent any
    tools { nodejs 'NodeJS-18' }
    environment {
        DOCKER_REPO = 'chafah/landmark-web-app'
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Test') {
            steps {
                sh 'npm ci && npm test'
                sh 'cd server && npm ci && npm test'
            }
        }
        stage('Generate Tag') {
            steps {
                script {
                    def branch = env.BRANCH_NAME.replaceAll('/', '-')
                    def timestamp = new Date().format('yyyyMMdd-HHmmss')
                    env.IMAGE_TAG = "${branch}-${timestamp}"
                    echo "Image tag: ${env.IMAGE_TAG}"
                }
            }
        }
        stage('Docker Build & Push') {
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
        stage('Cleanup') {
            steps {
                sh "docker rmi ${DOCKER_REPO}:${IMAGE_TAG} || true"
                sh "docker rmi ${DOCKER_REPO}:latest || true"
            }
        }
    }
}
```

---

## 10.4 Docker as Build Agent

Run your entire pipeline inside a Docker container:

```groovy
pipeline {
    agent {
        docker {
            image 'node:18'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'node --version'
                sh 'npm ci'
                sh 'npm run build'
            }
        }
    }
}
```

**Benefits:**
- No need to install Node.js on Jenkins server
- Consistent build environment
- Easy to change versions (just change image tag)

---

## 10.5 Setting Up Docker Hub Credentials

1. **Manage Jenkins → Credentials → Add**
2. Kind: **Username with password**
3. Username: Docker Hub username
4. Password: Docker Hub access token (from https://hub.docker.com/settings/security)
5. ID: `dockerhub-creds`

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `permission denied...docker.sock` | `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins` |
| `docker: command not found` | Install Docker: `sudo dnf install docker -y` |
| `unauthorized: authentication required` | Check Docker Hub credentials in Jenkins |
| Disk full after many builds | Add cleanup stage: `docker system prune -f` |
| Build slow | Use Docker layer caching, `.dockerignore` |

---

## Lab: Build and Push Docker Image

**Time:** 20 minutes

1. Ensure Docker is installed and Jenkins has permission
2. Create Docker Hub credentials in Jenkins
3. Create a pipeline that builds the Landmark app Docker image
4. Push it to your Docker Hub account
5. Verify on Docker Hub that the image exists

---

## Summary

- Docker must be installed on the server — the plugin only provides DSL
- `sudo usermod -aG docker jenkins` is critical for permission
- `docker.withRegistry()` handles login/logout automatically
- Use meaningful image tags (branch + timestamp)
- Clean up local images to save disk space
- Docker containers can also serve as build agents
