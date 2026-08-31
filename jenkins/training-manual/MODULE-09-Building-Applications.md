# MODULE 9: BUILDING APPLICATIONS WITH JENKINS

## Learning Objectives

By the end of this module, students will be able to:
- Build npm/Node.js applications in Jenkins
- Build Maven/Java applications in Jenkins
- Build Docker images in Jenkins
- Understand build lifecycle and artifact management

---

## 9.1 Building Node.js Applications (Landmark Project)

Our project uses a React frontend + Express backend:

```
landmark-web-app/
├── package.json         ← Frontend (React)
├── src/                 ← React source
├── server/
│   ├── package.json     ← Backend (Express)
│   └── index.js         ← Server entry point
└── Dockerfile           ← Multi-stage build
```

### npm Build Lifecycle

```
npm ci          → Install exact dependencies from lock file (CI-safe)
npm test        → Run test suite
npm run build   → Create production build (React → static files)
```

### Pipeline for Node.js

```groovy
pipeline {
    agent any
    tools { nodejs 'NodeJS-18' }
    stages {
        stage('Install Dependencies') {
            steps {
                sh 'npm ci'                           // Frontend deps
                sh 'cd server && npm ci'              // Backend deps
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
        stage('Build') {
            steps { sh 'npm run build' }             // Creates build/ directory
        }
    }
}
```

> **Note:** Use `npm ci` (not `npm install`) in CI. It's faster, stricter, and reproducible.

---

## 9.2 Building Maven Applications

For Java/Spring Boot projects:

### Maven Build Lifecycle

```
mvn clean           → Remove target/ directory
mvn compile         → Compile source code
mvn test            → Run unit tests
mvn package         → Create JAR/WAR
mvn verify          → Run integration tests
mvn install         → Install to local repo
mvn deploy          → Deploy to remote repo (Nexus/Artifactory)
```

### Pipeline for Maven

```groovy
pipeline {
    agent any
    tools {
        maven 'Maven-3.9'
        jdk 'JDK-17'
    }
    stages {
        stage('Build') {
            steps { sh 'mvn clean compile' }
        }
        stage('Test') {
            steps { sh 'mvn test' }
            post {
                always { junit '**/target/surefire-reports/*.xml' }
            }
        }
        stage('Package') {
            steps { sh 'mvn package -DskipTests' }
        }
        stage('Archive') {
            steps { archiveArtifacts artifacts: 'target/*.jar' }
        }
    }
}
```

---

## 9.3 Building Docker Images

### Landmark Dockerfile (Multi-stage)

```dockerfile
# Stage 1: Build React frontend
FROM node:18-alpine AS frontend
WORKDIR /app
COPY package.json .
RUN npm install
COPY public ./public
COPY src ./src
RUN npm run build

# Stage 2: Production server
FROM node:18-alpine
WORKDIR /app
COPY server/package.json .
RUN npm install
COPY server/index.js .
COPY --from=frontend /app/build ./public
ENV NODE_ENV=production
EXPOSE 5000
CMD ["node", "index.js"]
```

### Docker Build in Pipeline

```groovy
stage('Docker Build & Push') {
    steps {
        script {
            docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
                def app = docker.build("chafah/landmark-web-app:${IMAGE_TAG}")
                app.push()
                app.push('latest')
            }
        }
    }
}
```

---

## Lab: Build the Landmark App

**Time:** 20 minutes

1. Create a Pipeline job
2. Write a pipeline that: checks out code → installs deps → runs tests → builds → docker build
3. Verify the Docker image is created: `docker images | grep landmark`

---

## Summary

- Node.js: `npm ci` → `npm test` → `npm run build`
- Maven: `mvn clean` → `mvn test` → `mvn package`
- Docker: Use multi-stage builds to keep images small
- Use Jenkins `tools` directive to auto-install build tools
- Use `parallel` stages to run frontend/backend tests simultaneously
