# MODULE 2: INSTALLING JENKINS ON AMAZON LINUX

## Learning Objectives

By the end of this module, students will be able to:
- Provision an EC2 instance suitable for Jenkins
- Configure Security Groups for Jenkins access
- Install Java (JDK 17) on Amazon Linux
- Install and configure Jenkins
- Access Jenkins UI and complete initial setup
- Manage Jenkins as a systemd service

---

## 2.1 Planning the Jenkins Server

### Hardware Requirements

| Environment | Instance Type | vCPU | RAM | Storage |
|-------------|--------------|------|-----|---------|
| Learning/Lab | t3.medium | 2 | 4 GB | 30 GB EBS |
| Small Team (< 10 devs) | t3.large | 2 | 8 GB | 50 GB EBS |
| Medium Team (10-50 devs) | m5.xlarge | 4 | 16 GB | 100 GB EBS |
| Enterprise (50+ devs) | m5.2xlarge+ | 8+ | 32+ GB | 200+ GB EBS |

> **Instructor Note:** For this course, we use `t3.medium`. In production, Jenkins Controller should have at least 8 GB RAM. Builds should run on separate agents.

### Architecture for This Course

```
┌─────────────────────────────────────────────────────────────┐
│                        AWS VPC                               │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                  Public Subnet                         │  │
│  │                                                       │  │
│  │  ┌─────────────────────┐                              │  │
│  │  │   EC2 Instance       │                              │  │
│  │  │   Amazon Linux 2023  │                              │  │
│  │  │   t3.medium          │                              │  │
│  │  │                     │                              │  │
│  │  │   ┌──────────────┐  │     ┌──────────────────┐    │  │
│  │  │   │   Jenkins     │  │◀────│  Your Browser    │    │  │
│  │  │   │   Port 8080   │  │     │  (Port 8080)     │    │  │
│  │  │   └──────────────┘  │     └──────────────────┘    │  │
│  │  │   ┌──────────────┐  │                              │  │
│  │  │   │   Java 17     │  │                              │  │
│  │  │   └──────────────┘  │                              │  │
│  │  │   ┌──────────────┐  │                              │  │
│  │  │   │   Docker      │  │                              │  │
│  │  │   └──────────────┘  │                              │  │
│  │  └─────────────────────┘                              │  │
│  │                                                       │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2.2 Provisioning the EC2 Instance

### Step 1: Launch EC2

1. Go to **AWS Console → EC2 → Launch Instance**
2. Configure:

| Setting | Value |
|---------|-------|
| Name | `jenkins-server` |
| AMI | Amazon Linux 2023 (or Amazon Linux 2) |
| Instance type | t3.medium |
| Key pair | Create or select existing |
| VPC | Default VPC |
| Subnet | Public subnet |
| Auto-assign Public IP | Enable |
| Storage | 30 GB gp3 |

### Step 2: Security Group Configuration

Create a security group named `jenkins-sg`:

| Type | Protocol | Port | Source | Purpose |
|------|----------|------|--------|---------|
| SSH | TCP | 22 | Your IP/32 | SSH access |
| Custom TCP | TCP | 8080 | Your IP/32 | Jenkins UI |
| Custom TCP | TCP | 50000 | 10.0.0.0/16 | Agent connections (JNLP) |

> **Security Best Practice:** Never open port 8080 to 0.0.0.0/0 in production. Use a reverse proxy (ALB/Nginx) with HTTPS.

### Step 3: SSH into the Instance

```bash
chmod 400 your-key.pem
ssh -i your-key.pem ec2-user@<public-ip>
```

**Expected Output:**
```
   ,     #_
   ~\_  ####_        Amazon Linux 2023
  ~~  \_#####\
  ~~     \###|
  ~~       \#/ ___   https://aws.amazon.com/linux/amazon-linux-2023
   ~~       V~' '->
    ~~~         /
      ~~._.   _/
         _/ _/
       _/m/'
[ec2-user@ip-10-0-1-50 ~]$
```

---

## 2.3 Installing Java (JDK 17)

Jenkins requires Java 17 or 21.

### Amazon Linux 2023

```bash
# Install Amazon Corretto 17 (Amazon's JDK distribution)
sudo dnf install java-17-amazon-corretto -y
```

### Amazon Linux 2

```bash
# Install Amazon Corretto 17
sudo amazon-linux-extras install java-openjdk17 -y
```

### Verify Installation

```bash
java -version
```

**Expected Output:**
```
openjdk version "17.0.9" 2023-10-17 LTS
OpenJDK Runtime Environment Corretto-17.0.9.8.1 (build 17.0.9+8-LTS)
OpenJDK 64-Bit Server VM Corretto-17.0.9.8.1 (build 17.0.9+8-LTS, mixed mode, sharing)
```

### Set JAVA_HOME (if needed)

```bash
# Find Java path
which java
# /usr/bin/java

# Find actual binary
readlink -f $(which java)
# /usr/lib/jvm/java-17-amazon-corretto.x86_64/bin/java

# Set JAVA_HOME
echo 'export JAVA_HOME=/usr/lib/jvm/java-17-amazon-corretto.x86_64' | sudo tee -a /etc/profile.d/java.sh
source /etc/profile.d/java.sh
echo $JAVA_HOME
```

---

## 2.4 Installing Jenkins

### Step 1: Add Jenkins Repository

```bash
# Download Jenkins repo file
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo

# Import Jenkins GPG key
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
```

### Step 2: Install Jenkins

```bash
# Amazon Linux 2023
sudo dnf install jenkins -y

# Amazon Linux 2
sudo yum install jenkins -y
```

**Expected Output:**
```
Installed:
  jenkins-2.xxx.x-1.1.noarch

Complete!
```

### Step 3: Start and Enable Jenkins

```bash
# Start Jenkins service
sudo systemctl start jenkins

# Enable Jenkins to start on boot
sudo systemctl enable jenkins

# Check status
sudo systemctl status jenkins
```

**Expected Output:**
```
● jenkins.service - Jenkins Continuous Integration Server
     Loaded: loaded (/usr/lib/systemd/system/jenkins.service; enabled; preset: disabled)
     Active: active (running) since Thu 2026-06-05 10:30:00 UTC; 5s ago
   Main PID: 12345 (java)
      Tasks: 42 (limit: 4674)
     Memory: 512.0M
        CPU: 15.234s
     CGroup: /system.slice/jenkins.service
             └─12345 /usr/bin/java -Djava.awt.headless=true -jar /usr/share/java/jenkins.war...
```

### Step 4: Verify Jenkins is Listening

```bash
# Check port 8080
sudo ss -tlnp | grep 8080
```

**Expected Output:**
```
LISTEN 0      50                 *:8080            *:*    users:(("java",pid=12345,fd=12))
```

---

## 2.5 Firewall Configuration (if applicable)

Amazon Linux 2023 typically doesn't have firewalld running. Verify:

```bash
sudo systemctl status firewalld
```

If firewalld is active:

```bash
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --add-port=50000/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

---

## 2.6 Accessing Jenkins UI

### Step 1: Get Initial Admin Password

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

**Expected Output:**
```
a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
```

> **Instructor Note:** This password is auto-generated and unique per installation. Students should copy it.

### Step 2: Open Browser

Navigate to:
```
http://<your-ec2-public-ip>:8080
```

### Step 3: Unlock Jenkins

**Screen students will see:**
```
┌──────────────────────────────────────────────────────┐
│                                                      │
│              Getting Started                          │
│                                                      │
│    Unlock Jenkins                                    │
│                                                      │
│    To ensure Jenkins is securely set up by the       │
│    administrator, a password has been written to     │
│    the log and this file on the server:              │
│                                                      │
│    /var/lib/jenkins/secrets/initialAdminPassword     │
│                                                      │
│    Administrator password: [________________]        │
│                                                      │
│                    [Continue]                         │
│                                                      │
└──────────────────────────────────────────────────────┘
```

Paste the password from Step 1 and click **Continue**.

### Step 4: Install Plugins

**Screen students will see:**
```
┌──────────────────────────────────────────────────────┐
│                                                      │
│    Customize Jenkins                                 │
│                                                      │
│    ┌────────────────┐    ┌────────────────────┐     │
│    │   Install       │    │   Select plugins   │     │
│    │   suggested     │    │   to install       │     │
│    │   plugins       │    │                    │     │
│    └────────────────┘    └────────────────────┘     │
│                                                      │
└──────────────────────────────────────────────────────┘
```

Select **Install suggested plugins**. Wait 2-5 minutes.

### Step 5: Create Admin User

Fill in:
- Username: `admin`
- Password: (choose a strong password)
- Full name: Your Name
- E-mail: your@email.com

### Step 6: Instance Configuration

Set Jenkins URL:
```
http://<your-ec2-public-ip>:8080/
```

Click **Save and Finish** → **Start using Jenkins**

---

## 2.7 Validation Steps

Run these commands to confirm everything is working:

```bash
# Jenkins service running
sudo systemctl is-active jenkins
# Output: active

# Jenkins version
java -jar /usr/share/java/jenkins.war --version
# Output: 2.xxx.x

# Jenkins home directory exists
ls /var/lib/jenkins/
# Output: config.xml  jobs  logs  nodes  plugins  secrets  ...

# Jenkins user exists
id jenkins
# Output: uid=995(jenkins) gid=993(jenkins) groups=993(jenkins)

# Java version
java -version

# Jenkins listening on 8080
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/login
# Output: 200
```

---

## 2.8 Jenkins Service Management

### Common systemctl Commands

```bash
# Start Jenkins
sudo systemctl start jenkins

# Stop Jenkins
sudo systemctl stop jenkins

# Restart Jenkins
sudo systemctl restart jenkins

# Check status
sudo systemctl status jenkins

# View logs
sudo journalctl -u jenkins -f

# Enable on boot
sudo systemctl enable jenkins

# Disable on boot
sudo systemctl disable jenkins
```

### Jenkins Service File Location

```bash
cat /usr/lib/systemd/system/jenkins.service
```

Key settings in the service file:
- `User=jenkins` — Jenkins runs as its own user
- `Environment="JENKINS_PORT=8080"` — Default port
- `Environment="JAVA_OPTS=-Djava.awt.headless=true"` — Headless Java

### Changing Jenkins Port

```bash
# Edit the service override
sudo systemctl edit jenkins

# Add:
[Service]
Environment="JENKINS_PORT=9090"

# Reload and restart
sudo systemctl daemon-reload
sudo systemctl restart jenkins
```

---

## 2.9 IAM Considerations

For a Jenkins server that will deploy to AWS services:

### Option A: IAM Instance Profile (Recommended)

Attach an IAM Role to the EC2 instance with required permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:ListClusters",
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "*"
    }
  ]
}
```

### Option B: Access Keys in Jenkins Credentials

Store AWS Access Key + Secret Key in Jenkins credential store. Less secure but simpler for labs.

> **Production Recommendation:** Always use IAM Instance Profiles or IRSA (IAM Roles for Service Accounts) instead of access keys.

---

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| Cannot access port 8080 | Security Group | Add inbound rule for 8080 from your IP |
| Jenkins won't start | Java not found | Install JDK 17: `sudo dnf install java-17-amazon-corretto -y` |
| Jenkins won't start | Port in use | Check: `sudo ss -tlnp \| grep 8080` and kill the process |
| Permission denied | SELinux | Check: `getenforce` and set: `sudo setenforce 0` |
| Out of memory | Insufficient RAM | Upgrade instance type or add swap |
| Slow startup | First boot | Normal — Jenkins downloads and initializes plugins |

### Check Jenkins Logs

```bash
# Systemd journal
sudo journalctl -u jenkins --no-pager -n 50

# Jenkins log file
sudo tail -f /var/log/jenkins/jenkins.log
```

---

## Lab: Install Jenkins on Amazon Linux

**Time:** 30 minutes

**Objective:** Install a working Jenkins server from scratch.

**Steps:**
1. Launch a t3.medium EC2 instance with Amazon Linux 2023
2. SSH into the instance
3. Install Java 17
4. Install Jenkins
5. Start and enable the Jenkins service
6. Access the Jenkins UI from your browser
7. Complete initial setup (install suggested plugins, create admin user)
8. Verify Jenkins is accessible at `http://<ip>:8080`

**Success Criteria:**
- Jenkins dashboard loads in browser
- Can create a new Freestyle job
- `sudo systemctl status jenkins` shows "active (running)"

---

## Interview Questions

1. **Q:** What are the system requirements for a Jenkins server?
   **A:** Minimum 2 GB RAM (recommended 4+ GB), Java 17 or 21, 50+ GB disk. For production, use dedicated agents for builds and give the controller at least 8 GB RAM.

2. **Q:** How do you install Jenkins on a Linux server?
   **A:** Add the Jenkins yum/apt repository, import the GPG key, install the package, start and enable the systemd service, then access port 8080 for initial configuration.

3. **Q:** What port does Jenkins use by default?
   **A:** 8080 for the web UI, 50000 for JNLP agent connections.

4. **Q:** Where is the Jenkins home directory?
   **A:** `/var/lib/jenkins/` — contains all configuration, jobs, plugins, and build data.

---

## Summary

- Jenkins requires Java 17+ and runs on port 8080 by default
- Installation on Amazon Linux: add repo → install package → start service
- First-time setup requires the initial admin password from `/var/lib/jenkins/secrets/initialAdminPassword`
- Manage Jenkins with `systemctl` (start, stop, restart, status)
- Security Groups must allow inbound traffic on port 8080
- For AWS deployments, prefer IAM Instance Profiles over access keys
