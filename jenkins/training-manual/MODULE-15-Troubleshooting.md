# MODULE 15: TROUBLESHOOTING JENKINS

## Learning Objectives

- Diagnose and fix common Jenkins failures
- Read and interpret Jenkins logs
- Debug pipeline errors
- Resolve agent, Docker, and Git issues

---

## 15.1 Troubleshooting Workflow

```
1. READ the error message carefully
       │
2. CHECK Jenkins logs: journalctl -u jenkins -n 50
       │
3. CHECK build console output
       │
4. IDENTIFY the failing stage/step
       │
5. REPRODUCE locally if possible
       │
6. SEARCH Jenkins issues (issues.jenkins.io) or Stack Overflow
       │
7. FIX and verify
```

---

## 15.2 Common Issues and Solutions

### Jenkins Won't Start

```bash
# Check logs
sudo journalctl -u jenkins --no-pager -n 50

# Common causes:
```

| Cause | Log Message | Fix |
|-------|-------------|-----|
| Java not found | `Failed to start...java: not found` | `sudo dnf install java-17-amazon-corretto -y` |
| Port in use | `Address already in use: 8080` | `sudo fuser -k 8080/tcp` then restart |
| Corrupt plugin | `Failed to load plugin` | Remove plugin: `sudo rm /var/lib/jenkins/plugins/<bad>.jpi` |
| Disk full | `No space left on device` | `df -h`; clean: `docker system prune -f` |
| Permission denied | `Permission denied: /var/lib/jenkins` | `sudo chown -R jenkins:jenkins /var/lib/jenkins` |

### Plugin Conflicts

```bash
# Identify conflicting plugin
sudo journalctl -u jenkins | grep "Failed to load"

# Fix: remove the plugin
sudo rm /var/lib/jenkins/plugins/<plugin-name>.jpi
sudo rm -rf /var/lib/jenkins/plugins/<plugin-name>/
sudo systemctl restart jenkins
```

### Git Authentication Failures

| Error | Cause | Fix |
|-------|-------|-----|
| `Authentication failed` | Bad credentials | Regenerate PAT; update credential in Jenkins |
| `Permission denied (publickey)` | SSH key issue | Verify key added to GitHub; check credential ID |
| `Repository not found` | Repo URL wrong or private | Check URL; ensure token has `repo` scope |

### Docker Permission Errors

```bash
# Error: permission denied while trying to connect to the Docker daemon socket

# Fix:
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

# Verify:
sudo -u jenkins docker ps
```

### Agent Connection Issues

| Error | Fix |
|-------|-----|
| Agent offline | Check SSH connectivity: `ssh -i key jenkins@agent-ip` |
| Java not found on agent | Install Java on agent |
| Host key verification failed | Set "Non verifying" or add host key |
| Connection refused | Check Security Group allows port 22 from controller |

### Pipeline Debugging

```groovy
// Print environment variables
stage('Debug') {
    steps {
        sh 'env | sort'
        sh 'whoami'
        sh 'pwd'
        sh 'ls -la'
    }
}

// Print specific variable
echo "Branch: ${env.BRANCH_NAME}"
echo "Workspace: ${env.WORKSPACE}"
```

### Build Workspace Issues

```bash
# Clean workspace manually
sudo rm -rf /var/lib/jenkins/workspace/<job-name>

# Or in pipeline:
post {
    always { cleanWs() }
}
```

---

## 15.3 Useful Diagnostic Commands

```bash
# Jenkins service status
sudo systemctl status jenkins

# Jenkins Java process
ps aux | grep jenkins

# Memory usage
free -m

# Disk usage
df -h /var/lib/jenkins
du -sh /var/lib/jenkins/*

# Port check
sudo ss -tlnp | grep 8080

# Recent Jenkins logs
sudo journalctl -u jenkins --since "10 minutes ago"

# Check Docker
docker info
docker ps -a
```

---

## Lab: Troubleshooting Scenarios

**Time:** 20 minutes

Instructor will break things; students must fix them:

1. Stop Jenkins and remove a required plugin. Student must identify and reinstall it.
2. Change Docker socket permissions. Student must fix Docker permission error.
3. Invalidate a Git credential. Student must diagnose auth failure and fix.

---

## Summary

- Always start by reading the error message and checking logs
- Most issues fall into: Java, permissions, plugins, network, or disk space
- `journalctl -u jenkins` is your best friend
- Docker issues are almost always permission-related
- Git issues are almost always credential-related
- When stuck: check Console Output → check system logs → search online
