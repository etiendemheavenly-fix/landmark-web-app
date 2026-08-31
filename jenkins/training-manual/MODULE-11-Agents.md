# MODULE 11: JENKINS AGENTS

## Learning Objectives

By the end of this module, students will be able to:
- Explain the Controller-Agent architecture
- Set up static SSH agents on EC2
- Configure labels and executors
- Use Docker-based dynamic agents
- Choose the right agent strategy

---

## 11.1 Why Use Agents?

```
WITHOUT AGENTS (Bad):                WITH AGENTS (Good):
┌──────────────────┐                 ┌──────────────────┐
│    Controller     │                 │    Controller     │
│  (Web UI + Builds)│                 │  (Web UI only)   │
│                  │                 │  Executors: 0    │
│  ⚠️ Overloaded   │                 └────────┬─────────┘
│  ⚠️ Insecure     │                          │
│  ⚠️ Single point │                 ┌────────┼────────┐
│    of failure    │                 │        │        │
└──────────────────┘                 ▼        ▼        ▼
                                  ┌──────┐ ┌──────┐ ┌──────┐
                                  │Agent1│ │Agent2│ │Agent3│
                                  │Linux │ │Docker│ │macOS │
                                  └──────┘ └──────┘ └──────┘
```

**Benefits of agents:**
- Distribute load across multiple machines
- Keep controller secure (no build code execution)
- Different OS/tool combinations per agent
- Scale horizontally by adding agents
- Isolate builds from each other

---

## 11.2 Static SSH Agent (EC2)

### Step 1: Provision Agent EC2 Instance

Launch an EC2 instance:
- AMI: Amazon Linux 2023
- Type: t3.medium
- Security Group: Allow SSH (port 22) from Jenkins Controller IP

### Step 2: Prepare the Agent

SSH into the agent instance:

```bash
# Install Java (required for Jenkins agent)
sudo dnf install java-17-amazon-corretto -y

# Install tools needed for builds
sudo dnf install git docker -y
sudo systemctl start docker && sudo systemctl enable docker

# Create Jenkins user
sudo useradd -m -s /bin/bash jenkins
sudo usermod -aG docker jenkins

# Create workspace directory
sudo mkdir -p /home/jenkins/agent
sudo chown jenkins:jenkins /home/jenkins/agent
```

### Step 3: Set Up SSH Key

On the **Controller**:
```bash
# Generate key pair
sudo -u jenkins ssh-keygen -t ed25519 -f /var/lib/jenkins/.ssh/agent_key -N ""

# Copy public key to agent
sudo -u jenkins ssh-copy-id -i /var/lib/jenkins/.ssh/agent_key.pub jenkins@<agent-ip>
```

### Step 4: Add SSH Credential in Jenkins

1. **Manage Jenkins → Credentials → Add**
2. Kind: **SSH Username with private key**
3. Username: `jenkins`
4. Private Key: Enter directly (paste content of `agent_key`)
5. ID: `agent-ssh-key`

### Step 5: Add Node in Jenkins

1. **Manage Jenkins → Nodes → New Node**
2. Name: `build-agent-01`
3. Type: Permanent Agent

| Field | Value |
|-------|-------|
| # of executors | 2 |
| Remote root directory | `/home/jenkins/agent` |
| Labels | `linux docker` |
| Launch method | Launch agents via SSH |
| Host | `<agent-private-ip>` |
| Credentials | `agent-ssh-key` |
| Host Key Verification | Non verifying (for lab) |

4. Click **Save**

### Step 6: Verify Connection

The agent should show "online" in **Manage Jenkins → Nodes**.

---

## 11.3 Labels and Agent Selection

Labels let you direct builds to specific agents:

```groovy
// Run on any agent with 'docker' label
pipeline {
    agent { label 'docker' }
    stages { ... }
}

// Run on agent with multiple labels
pipeline {
    agent { label 'linux && docker' }
    stages { ... }
}

// Different agents per stage
pipeline {
    agent none
    stages {
        stage('Build') {
            agent { label 'linux' }
            steps { sh 'npm run build' }
        }
        stage('Test macOS') {
            agent { label 'macos' }
            steps { sh 'npm test' }
        }
    }
}
```

---

## 11.4 Docker Agents (Dynamic)

Run each build in a fresh Docker container:

```groovy
pipeline {
    agent {
        docker {
            image 'node:18'
            label 'docker'    // Run on agent that has Docker
            args '-v /tmp:/tmp'
        }
    }
    stages {
        stage('Build') {
            steps { sh 'npm ci && npm run build' }
        }
    }
}
```

**How it works:**
1. Jenkins launches `docker run node:18` on an agent with Docker
2. Mounts the workspace into the container
3. Executes all steps inside the container
4. Container is removed after the build

---

## 11.5 Executors

| Setting | Meaning |
|---------|---------|
| Executors = 1 | One build at a time |
| Executors = 2 | Two concurrent builds |
| Executors = 4 | Four concurrent builds |

**Rule of thumb:** Set executors = number of CPU cores on the agent.

**Controller executors should be 0 in production** (no builds on controller).

---

## Lab: Set Up an SSH Agent

**Time:** 30 minutes

1. Launch a second EC2 instance (agent)
2. Install Java and Docker on it
3. Create the jenkins user and SSH key
4. Add the agent in Jenkins UI
5. Create a pipeline that runs on the agent using `agent { label 'docker' }`
6. Verify the build runs on the agent (check Console Output for "Running on build-agent-01")

---

## Interview Questions

1. **Q:** Why should you not run builds on the Jenkins Controller?
   **A:** Security risk (build code can access Jenkins secrets), performance impact on the UI, and single point of failure. In production, set controller executors to 0.

2. **Q:** How do you add a new build agent to Jenkins?
   **A:** Install Java on the agent machine, create a jenkins user, set up SSH keys, then in Jenkins UI: Manage Jenkins → Nodes → New Node. Configure the SSH connection, labels, and executors.

3. **Q:** What are labels used for?
   **A:** Labels tag agents with capabilities (e.g., `docker`, `linux`, `gpu`). Pipelines use `agent { label 'docker' }` to run on agents with specific tools installed.

---

## Summary

- Controller should coordinate only; agents execute builds
- Static agents: always running EC2 instances connected via SSH
- Docker agents: ephemeral containers spun up per build
- Labels direct builds to agents with specific capabilities
- Set executors based on CPU cores available
- Production: Controller executors = 0
