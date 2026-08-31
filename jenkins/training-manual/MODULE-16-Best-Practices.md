# MODULE 16: JENKINS BEST PRACTICES

## Learning Objectives

- Apply production-grade security practices
- Design for high availability and scaling
- Implement governance and compliance
- Optimize pipeline performance

---

## 16.1 Security Best Practices

| Practice | Priority |
|----------|----------|
| HTTPS via reverse proxy (Nginx/ALB) | Critical |
| Controller executors = 0 | Critical |
| RBAC with Role Strategy plugin | Critical |
| Disable user sign-up | High |
| Credential scoping (folder-level) | High |
| Audit log monitoring | High |
| Plugin vulnerability scanning | Medium |
| Regular credential rotation | Medium |
| Network segmentation (private subnet for controller) | Medium |

---

## 16.2 High Availability

```
                    ┌─────────────┐
                    │  Load       │
                    │  Balancer   │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼────┐ ┌────▼─────┐ ┌───▼──────┐
        │ Jenkins  │ │ Jenkins  │ │ Jenkins  │
        │ (Active) │ │ (Standby)│ │ (Agents) │
        └─────┬────┘ └────┬─────┘ └──────────┘
              │            │
              └─────┬──────┘
                    │
              ┌─────▼─────┐
              │ Shared    │
              │ Storage   │
              │ (EFS/NFS) │
              └───────────┘
```

**Options:**
- **Active/Passive:** One controller active, one on standby with shared storage
- **CloudBees HA:** Commercial Jenkins distribution with built-in HA
- **Kubernetes:** Run Jenkins on K8s with persistent volumes

---

## 16.3 Scaling

| Strategy | When to Use |
|----------|-------------|
| Add static agents | Predictable workload |
| EC2 Fleet plugin | Variable workload, cost optimization |
| Docker agents | Fast spin-up, isolated builds |
| Kubernetes agents | Cloud-native, auto-scaling |
| Split into multiple controllers | Org-level isolation (team per controller) |

---

## 16.4 Pipeline Best Practices

```groovy
// ✅ DO: Use tools directive
tools { nodejs 'NodeJS-18' }

// ❌ DON'T: Install tools in shell
sh 'curl -fsSL https://deb.nodesource.com/setup_18.x | bash -'

// ✅ DO: Use credentials binding
withCredentials([...]) { sh 'deploy' }

// ❌ DON'T: Hardcode secrets
sh 'docker login -u admin -p password123'

// ✅ DO: Use parallel stages
parallel { stage('A') {...} stage('B') {...} }

// ❌ DON'T: Run tests sequentially when independent

// ✅ DO: Clean workspace
post { always { cleanWs() } }

// ❌ DON'T: Leave artifacts accumulating

// ✅ DO: Use specific agent labels
agent { label 'docker && linux' }

// ❌ DON'T: Run everything on the controller
```

---

## 16.5 Governance and Compliance

| Practice | Implementation |
|----------|---------------|
| Pipeline as Code | All Jenkinsfiles in version control |
| Shared Libraries | Enforce standard pipeline patterns |
| Approval gates | `input` step before production deploy |
| Audit trails | Jenkins audit log + CloudTrail |
| Secret scanning | Prevent credentials in code |
| Build retention | Discard old builds (keep last 10-20) |
| Change management | Require PR approval for Jenkinsfile changes |

---

## 16.6 Performance Tuning

```bash
# Increase JVM heap (edit service override)
sudo systemctl edit jenkins

[Service]
Environment="JAVA_OPTS=-Xms2g -Xmx4g -XX:+UseG1GC"

sudo systemctl daemon-reload
sudo systemctl restart jenkins
```

| Tuning | Action |
|--------|--------|
| Slow UI | Increase heap, reduce # of builds kept |
| Long queues | Add more agents/executors |
| Slow Git checkout | Use shallow clone: `checkout([$class: 'GitSCM', extensions: [[$class: 'CloneOption', shallow: true]]])` |
| Large workspaces | Use `.dockerignore`, `cleanWs()` |
| Plugin bloat | Remove unused plugins |

---

## 16.7 Production Checklist

```
PRE-DEPLOYMENT:
☐ Jenkins behind HTTPS (ALB/Nginx)
☐ Controller in private subnet
☐ Security groups restrict access
☐ RBAC configured
☐ Automated backups to S3
☐ Monitoring/alerting (CloudWatch)
☐ At least 2 build agents

PIPELINE:
☐ All pipelines defined in Jenkinsfile (no UI-configured jobs)
☐ Credentials in Jenkins store (never in code)
☐ Build logs retained for 30 days minimum
☐ Notifications on failure (Slack/email)
☐ Health checks after deployment
☐ Rollback procedure documented

MAINTENANCE:
☐ Weekly plugin update review
☐ Monthly credential rotation
☐ Quarterly Jenkins version upgrade
☐ Backup restore tested
```

---

## Interview Questions

1. **Q:** How do you secure Jenkins in an enterprise environment?
   **A:** HTTPS via reverse proxy, RBAC, controller executors set to 0, network segmentation, credential scoping, regular plugin updates, disable CLI over remoting, audit logging, and automated backups.

2. **Q:** How do you scale Jenkins?
   **A:** Add agents (static EC2, Docker, or Kubernetes-based). Use labels to route builds. For very large orgs, use multiple controllers per team.

3. **Q:** What are your Jenkins pipeline best practices?
   **A:** Pipeline as Code in Jenkinsfile, use `tools` directive, parallel test stages, credential binding (never hardcode), clean workspace, meaningful stage names, health checks after deploy, shared libraries for reusable logic.

---

## Summary

- Security: HTTPS, RBAC, no builds on controller, credential scoping
- Scale: Agents (static, Docker, K8s), not bigger controllers
- Performance: JVM tuning, build retention, shallow clones
- Governance: Pipeline as Code, shared libraries, approval gates
- Always have a tested backup and documented DR plan
- Follow the production checklist before going live

---

## Course Wrap-Up

Congratulations! You've completed the Jenkins CI/CD Masterclass.

**What you've learned:**
- Jenkins architecture and installation
- Administration, security, and plugin management
- Freestyle projects and Pipeline as Code
- GitHub integration with webhooks
- Building Node.js, Maven, and Docker applications
- Agent architecture and scaling
- Complete CI/CD pipeline with deployment
- Credentials, backups, troubleshooting, and best practices

**Next steps:**
- Set up Jenkins for your own project
- Implement the complete pipeline from Module 12
- Explore Jenkins Shared Libraries
- Consider Jenkins on Kubernetes for cloud-native workloads
