# MODULE 14: JENKINS BACKUP AND DISASTER RECOVERY

## Learning Objectives

- Implement backup strategies for Jenkins
- Automate backups with cron
- Restore Jenkins from backup
- Plan for disaster recovery

---

## 14.1 What to Backup

| Priority | Path | Description |
|----------|------|-------------|
| Critical | `/var/lib/jenkins/secrets/` | Encryption keys (without these, credentials are lost) |
| Critical | `/var/lib/jenkins/credentials.xml` | Encrypted credentials |
| Critical | `/var/lib/jenkins/config.xml` | System configuration |
| High | `/var/lib/jenkins/jobs/*/config.xml` | Job configurations |
| High | `/var/lib/jenkins/users/` | User accounts |
| High | `/var/lib/jenkins/nodes/` | Agent configs |
| Medium | `/var/lib/jenkins/plugins/*.jpi` | Plugin files |
| Low | `/var/lib/jenkins/jobs/*/builds/` | Build history (large) |
| Skip | `/var/lib/jenkins/workspace/` | Regenerated per build |
| Skip | `/var/lib/jenkins/war/` | Extracted from WAR file |

---

## 14.2 Backup Script

```bash
#!/bin/bash
# /opt/scripts/jenkins-backup.sh

set -e

JENKINS_HOME="/var/lib/jenkins"
BACKUP_DIR="/opt/jenkins-backups"
S3_BUCKET="s3://company-jenkins-backups"
DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="jenkins-backup-${DATE}.tar.gz"
RETENTION_DAYS=30

echo "[$(date)] Starting Jenkins backup..."

mkdir -p ${BACKUP_DIR}

# Create tarball excluding workspace and war
tar -czf ${BACKUP_DIR}/${BACKUP_FILE} \
    --exclude="${JENKINS_HOME}/workspace" \
    --exclude="${JENKINS_HOME}/war" \
    --exclude="${JENKINS_HOME}/.cache" \
    --exclude="${JENKINS_HOME}/caches" \
    --exclude="${JENKINS_HOME}/logs" \
    --exclude="*/builds/*/archive" \
    ${JENKINS_HOME}/ 2>/dev/null

BACKUP_SIZE=$(du -h ${BACKUP_DIR}/${BACKUP_FILE} | cut -f1)
echo "[$(date)] Backup created: ${BACKUP_FILE} (${BACKUP_SIZE})"

# Upload to S3 (optional)
if command -v aws &>/dev/null; then
    aws s3 cp ${BACKUP_DIR}/${BACKUP_FILE} ${S3_BUCKET}/${BACKUP_FILE}
    echo "[$(date)] Uploaded to S3"
fi

# Clean old local backups
find ${BACKUP_DIR} -name "jenkins-backup-*.tar.gz" -mtime +${RETENTION_DAYS} -delete
echo "[$(date)] Cleaned backups older than ${RETENTION_DAYS} days"

echo "[$(date)] Backup complete!"
```

### Automate with Cron

```bash
chmod +x /opt/scripts/jenkins-backup.sh

# Run daily at 2 AM
echo "0 2 * * * /opt/scripts/jenkins-backup.sh >> /var/log/jenkins-backup.log 2>&1" | sudo crontab -
```

---

## 14.3 Restore Procedure

```bash
# 1. Stop Jenkins
sudo systemctl stop jenkins

# 2. Restore from backup
sudo tar -xzf /opt/jenkins-backups/jenkins-backup-20260605-020000.tar.gz -C /

# 3. Fix permissions
sudo chown -R jenkins:jenkins /var/lib/jenkins/

# 4. Start Jenkins
sudo systemctl start jenkins

# 5. Verify
sudo systemctl status jenkins
# Access UI and verify jobs/credentials are present
```

---

## 14.4 Disaster Recovery Plan

| Scenario | RTO | RPO | Action |
|----------|-----|-----|--------|
| Plugin corrupt | 5 min | 0 | Restart Jenkins; remove bad plugin |
| Disk full | 15 min | 0 | Expand EBS volume; clean old builds |
| Server crash | 30 min | 24h | Launch new EC2; restore from S3 backup |
| Complete loss | 1 hour | 24h | Rebuild from backup + IaC (Terraform) |

---

## Lab: Backup and Restore

**Time:** 15 minutes

1. Create the backup script
2. Run it manually
3. Verify the tar file contains expected content: `tar -tzf <backup>.tar.gz | head -20`
4. Simulate restore: stop Jenkins, restore from backup, start Jenkins
5. Verify everything works

---

## Summary

- Back up `secrets/`, `credentials.xml`, `config.xml`, and `jobs/*/config.xml` at minimum
- Automate with cron and optionally upload to S3
- Test restores regularly — untested backups are not backups
- Document your disaster recovery plan with RTO/RPO targets
