# MODULE 3: JENKINS ADMINISTRATION

## Learning Objectives

By the end of this module, students will be able to:
- Navigate the Jenkins home directory and file structure
- View and interpret Jenkins logs
- Manage plugins (install, update, remove)
- Configure Global Tools (JDK, Node.js, Maven, Git)
- Perform Jenkins backups and restores
- Upgrade Jenkins safely

---

## 3.1 Jenkins Home Directory

The Jenkins home directory is the heart of your Jenkins installation. Everything Jenkins stores lives here.

**Default location:** `/var/lib/jenkins/`

```
/var/lib/jenkins/
├── config.xml                    # Main Jenkins configuration
├── credentials.xml               # Encrypted credentials
├── hudson.model.UpdateCenter.xml # Update center config
├── identity.key.enc              # Instance identity
├── jenkins.telemetry.Correlator.xml
├── jobs/                         # All job configurations and build history
│   ├── landmark-web-app/
│   │   ├── config.xml            # Job configuration
│   │   ├── builds/               # Build history
│   │   │   ├── 1/
│   │   │   │   ├── build.xml
│   │   │   │   ├── log
│   │   │   │   └── changelog.xml
│   │   │   ├── 2/
│   │   │   └── lastSuccessfulBuild -> 2
│   │   └── nextBuildNumber
│   └── another-job/
├── logs/                         # Jenkins system logs
├── nodes/                        # Agent configurations
│   └── agent-01/
│       └── config.xml
├── plugins/                      # Installed plugins (.jpi files)
│   ├── git.jpi
│   ├── docker-workflow.jpi
│   └── ...
├── secrets/                      # Encryption keys and secrets
│   ├── initialAdminPassword
│   ├── master.key
│   └── hudson.util.Secret
├── users/                        # User accounts
│   └── admin_123456/
│       └── config.xml
├── workspace/                    # Build workspaces (temporary)
│   └── landmark-web-app/
│       ├── src/
│       ├── server/
│       └── Jenkinsfile
├── updates/                      # Plugin update metadata
├── war/                          # Extracted WAR file
└── userContent/                  # Static files served at /userContent/
```

### Key Directories Explained

| Directory | Purpose | Backup? |
|-----------|---------|:-------:|
| `config.xml` | Main system config | ✅ |
| `jobs/` | All job configs and build history | ✅ |
| `plugins/` | Installed plugin files | ✅ |
| `secrets/` | Encryption keys | ✅ Critical |
| `credentials.xml` | Stored credentials (encrypted) | ✅ Critical |
| `users/` | User account data | ✅ |
| `nodes/` | Agent configurations | ✅ |
| `workspace/` | Temporary build files | ❌ (regenerated) |
| `war/` | Extracted Jenkins WAR | ❌ (regenerated) |

---

## 3.2 Jenkins Logs

### Viewing System Logs

```bash
# Via systemd journal (real-time)
sudo journalctl -u jenkins -f

# Via systemd journal (last 100 lines)
sudo journalctl -u jenkins --no-pager -n 100

# Via Jenkins log file
sudo tail -f /var/log/jenkins/jenkins.log

# Grep for errors
sudo journalctl -u jenkins | grep -i error
```

### Log Locations

| Log | Location |
|-----|----------|
| System log | `/var/log/jenkins/jenkins.log` |
| Systemd journal | `journalctl -u jenkins` |
| Build logs | `/var/lib/jenkins/jobs/<job>/builds/<number>/log` |
| Plugin logs | Jenkins UI → Manage Jenkins → System Log |

### Jenkins UI Logs

Navigate to **Manage Jenkins → System Log**:
- **All Jenkins Logs** — everything
- **Create custom log recorders** for specific components

---

## 3.3 Plugin Management

### Installing Plugins via UI

1. **Manage Jenkins → Plugins → Available plugins**
2. Search for the plugin name
3. Check the box and click **Install**
4. Restart Jenkins if required

### Installing Plugins via CLI

```bash
# Download plugin
sudo wget https://updates.jenkins.io/latest/git.hpi -O /var/lib/jenkins/plugins/git.hpi

# Fix permissions
sudo chown jenkins:jenkins /var/lib/jenkins/plugins/git.hpi

# Restart Jenkins
sudo systemctl restart jenkins
```

### Installing Plugins via Jenkins CLI

```bash
# Download Jenkins CLI
wget http://localhost:8080/jnlpJars/jenkins-cli.jar

# Install plugin
java -jar jenkins-cli.jar -s http://localhost:8080/ -auth admin:<password> install-plugin git docker-workflow nodejs

# Safe restart
java -jar jenkins-cli.jar -s http://localhost:8080/ -auth admin:<password> safe-restart
```

### Updating Plugins

1. **Manage Jenkins → Plugins → Updates**
2. Select plugins to update
3. Click **Download now and install after restart**
4. Check "Restart Jenkins when installation is complete"

### Removing Plugins

1. **Manage Jenkins → Plugins → Installed plugins**
2. Find the plugin and click **Uninstall** (if available)
3. Or manually: `sudo rm /var/lib/jenkins/plugins/<plugin>.jpi`
4. Restart Jenkins

> **Best Practice:** Always test plugin updates in a non-production environment first. Plugin updates can break existing pipelines.

---

## 3.4 Global Tool Configuration

Navigate to **Manage Jenkins → Tools**

### JDK Configuration

| Field | Value |
|-------|-------|
| Name | `JDK-17` |
| Install automatically | ✅ |
| Version | OpenJDK 17 |

Or point to existing installation:
| Field | Value |
|-------|-------|
| Name | `JDK-17` |
| Install automatically | ❌ |
| JAVA_HOME | `/usr/lib/jvm/java-17-amazon-corretto.x86_64` |

### NodeJS Configuration

Requires the **NodeJS** plugin.

| Field | Value |
|-------|-------|
| Name | `NodeJS-18` |
| Install automatically | ✅ |
| Version | NodeJS 18.x |

### Maven Configuration

| Field | Value |
|-------|-------|
| Name | `Maven-3.9` |
| Install automatically | ✅ |
| Version | 3.9.x |

### Git Configuration

| Field | Value |
|-------|-------|
| Name | `Default` |
| Path to Git executable | `/usr/bin/git` |

### Using Tools in a Pipeline

```groovy
pipeline {
    agent any
    tools {
        jdk 'JDK-17'
        nodejs 'NodeJS-18'
        maven 'Maven-3.9'
    }
    stages {
        stage('Verify Tools') {
            steps {
                sh 'java -version'
                sh 'node --version'
                sh 'mvn --version'
            }
        }
    }
}
```

---

## 3.5 System Configuration

Navigate to **Manage Jenkins → System**

### Key Settings

| Setting | Purpose | Default |
|---------|---------|---------|
| Jenkins URL | Base URL for links in emails/notifications | `http://localhost:8080/` |
| System Admin e-mail | From address for notifications | (empty) |
| # of executors | Build slots on Controller | 2 (set to 0 in production) |
| Quiet period | Seconds to wait before starting a build | 5 |
| SCM checkout retry count | Retries for git checkout | 0 |

> **Production Recommendation:** Set Controller executors to **0** and run all builds on agents.

### Environment Variables

Set global environment variables at **Manage Jenkins → System → Global properties → Environment variables**:

| Name | Value |
|------|-------|
| `DOCKER_REPO` | `chafah/landmark-web-app` |
| `AWS_REGION` | `us-east-1` |

---

## 3.6 Backup Strategies

### What to Backup

```bash
# Critical (must backup)
/var/lib/jenkins/config.xml
/var/lib/jenkins/credentials.xml
/var/lib/jenkins/secrets/
/var/lib/jenkins/jobs/*/config.xml
/var/lib/jenkins/users/
/var/lib/jenkins/nodes/

# Important (should backup)
/var/lib/jenkins/plugins/*.jpi

# Skip (can be regenerated)
/var/lib/jenkins/workspace/
/var/lib/jenkins/war/
/var/lib/jenkins/jobs/*/builds/  (optional — build history is large)
```

### Simple Backup Script

```bash
#!/bin/bash
# jenkins-backup.sh

BACKUP_DIR="/opt/jenkins-backups"
JENKINS_HOME="/var/lib/jenkins"
DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/jenkins-backup-${DATE}.tar.gz"

mkdir -p ${BACKUP_DIR}

# Stop Jenkins for consistent backup (optional)
# sudo systemctl stop jenkins

tar -czf ${BACKUP_FILE} \
  --exclude="${JENKINS_HOME}/workspace" \
  --exclude="${JENKINS_HOME}/war" \
  --exclude="${JENKINS_HOME}/.cache" \
  --exclude="${JENKINS_HOME}/jobs/*/builds/*/archive" \
  ${JENKINS_HOME}/

# Restart if stopped
# sudo systemctl start jenkins

# Keep only last 7 backups
find ${BACKUP_DIR} -name "jenkins-backup-*.tar.gz" -mtime +7 -delete

echo "Backup created: ${BACKUP_FILE}"
echo "Size: $(du -h ${BACKUP_FILE} | cut -f1)"
```

### Automated Backup with Cron

```bash
# Add to crontab
sudo crontab -e

# Daily at 2 AM
0 2 * * * /opt/scripts/jenkins-backup.sh >> /var/log/jenkins-backup.log 2>&1
```

### Restore from Backup

```bash
# Stop Jenkins
sudo systemctl stop jenkins

# Restore
sudo tar -xzf /opt/jenkins-backups/jenkins-backup-20260605-020000.tar.gz -C /

# Fix permissions
sudo chown -R jenkins:jenkins /var/lib/jenkins/

# Start Jenkins
sudo systemctl start jenkins
```

---

## 3.7 Upgrading Jenkins

### Check Current Version

```bash
java -jar /usr/share/java/jenkins.war --version
# Or: Manage Jenkins → (bottom of page shows version)
```

### Upgrade Process

```bash
# 1. Backup first!
sudo /opt/scripts/jenkins-backup.sh

# 2. Update package
sudo dnf update jenkins -y    # Amazon Linux 2023
# OR
sudo yum update jenkins -y    # Amazon Linux 2

# 3. Restart
sudo systemctl restart jenkins

# 4. Verify
sudo systemctl status jenkins
```

### Safe Upgrade Procedure (Production)

1. Backup Jenkins home
2. Review release notes for breaking changes
3. Update plugins first (compatible with new version)
4. Schedule maintenance window
5. Stop Jenkins
6. Update Jenkins package
7. Start Jenkins
8. Verify all jobs work
9. Monitor logs for errors

---

## Troubleshooting

| Problem | Command | Solution |
|---------|---------|----------|
| Jenkins won't start | `sudo journalctl -u jenkins -n 50` | Check logs for Java errors |
| Disk full | `df -h /var/lib/jenkins` | Clean old builds: Manage Jenkins → (job) → discard old builds |
| Too many plugins | `ls /var/lib/jenkins/plugins/ \| wc -l` | Remove unused plugins |
| Slow UI | Check RAM: `free -m` | Increase instance size or tune JVM |
| Lost admin password | See procedure below | Reset security |

### Reset Admin Password

```bash
# Stop Jenkins
sudo systemctl stop jenkins

# Edit config to disable security temporarily
sudo sed -i 's/<useSecurity>true</<useSecurity>false</' /var/lib/jenkins/config.xml

# Start Jenkins
sudo systemctl start jenkins

# Access without login, go to Manage Jenkins → Security
# Re-enable security and set new password

# Restart
sudo systemctl restart jenkins
```

---

## Lab: Jenkins Administration

**Time:** 20 minutes

1. SSH into your Jenkins server
2. Explore the `/var/lib/jenkins/` directory
3. Install the **NodeJS** plugin via the UI
4. Configure NodeJS 18 in **Manage Jenkins → Tools**
5. Create a backup script and run it
6. View Jenkins logs with `journalctl`
7. Check disk usage: `du -sh /var/lib/jenkins/*`

---

## Interview Questions

1. **Q:** Where does Jenkins store its configuration?
   **A:** `/var/lib/jenkins/` — specifically `config.xml` for system config, `jobs/` for job configs, `plugins/` for plugins, and `secrets/` for encryption keys.

2. **Q:** How do you backup Jenkins?
   **A:** Tar the Jenkins home directory (excluding `workspace/` and `war/`). Critical files: `config.xml`, `credentials.xml`, `secrets/`, `jobs/*/config.xml`, and `users/`.

3. **Q:** How do you upgrade Jenkins safely?
   **A:** Backup first, review release notes, update plugins for compatibility, schedule a maintenance window, update the package, restart, and verify.

---

## Summary

- Jenkins home (`/var/lib/jenkins/`) contains all configuration, jobs, and plugins
- Use `journalctl -u jenkins -f` to view real-time logs
- Configure Global Tools (JDK, Node.js, Maven) in Manage Jenkins → Tools
- Backup regularly — at minimum: `config.xml`, `credentials.xml`, `secrets/`, and job configs
- Never skip backups before upgrades
- Set Controller executors to 0 in production
