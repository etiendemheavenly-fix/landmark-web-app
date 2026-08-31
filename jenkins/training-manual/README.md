# Jenkins CI/CD Masterclass for DevOps Engineers

## Training Manual — 3-Day Bootcamp

**Project:** Landmark Technologies Web Application
**Repository:** https://github.com/CHAFAH/landmark-web-app

---

## Course Structure

| Day | Modules | Topics |
|-----|---------|--------|
| Day 1 | 1-6 | Introduction, Installation, Administration, Security, Plugins, Freestyle |
| Day 2 | 7-11 | Pipelines, GitHub Integration, Building Apps, Docker, Agents |
| Day 3 | 12-16 | Complete CI/CD Project, Credentials, Backup, Troubleshooting, Best Practices |

---

## Modules

| # | Module | File |
|---|--------|------|
| 1 | Introduction to Jenkins | [MODULE-01-Introduction.md](MODULE-01-Introduction.md) |
| 2 | Installing Jenkins on Amazon Linux | [MODULE-02-Installation.md](MODULE-02-Installation.md) |
| 3 | Jenkins Administration | [MODULE-03-Administration.md](MODULE-03-Administration.md) |
| 4 | Jenkins Security | [MODULE-04-Security.md](MODULE-04-Security.md) |
| 5 | Jenkins Plugins | [MODULE-05-Plugins.md](MODULE-05-Plugins.md) |
| 6 | Freestyle Projects | [MODULE-06-Freestyle-Projects.md](MODULE-06-Freestyle-Projects.md) |
| 7 | Jenkins Pipelines | [MODULE-07-Pipelines.md](MODULE-07-Pipelines.md) |
| 8 | Integrating GitHub with Jenkins | [MODULE-08-GitHub-Integration.md](MODULE-08-GitHub-Integration.md) |
| 9 | Building Applications | [MODULE-09-Building-Applications.md](MODULE-09-Building-Applications.md) |
| 10 | Docker with Jenkins | [MODULE-10-Docker.md](MODULE-10-Docker.md) |
| 11 | Jenkins Agents | [MODULE-11-Agents.md](MODULE-11-Agents.md) |
| 12 | Complete CI/CD Project | [MODULE-12-Complete-CICD-Project.md](MODULE-12-Complete-CICD-Project.md) |
| 13 | Credentials Management | [MODULE-13-Credentials.md](MODULE-13-Credentials.md) |
| 14 | Backup and Disaster Recovery | [MODULE-14-Backup-DR.md](MODULE-14-Backup-DR.md) |
| 15 | Troubleshooting | [MODULE-15-Troubleshooting.md](MODULE-15-Troubleshooting.md) |
| 16 | Best Practices | [MODULE-16-Best-Practices.md](MODULE-16-Best-Practices.md) |

---

## Prerequisites

Students should already know:
- Linux Administration
- Shell Scripting
- Git and GitHub
- Docker
- Maven
- npm
- Basic AWS Concepts

---

## Lab Environment

- Jenkins Server: t3.medium EC2 (Amazon Linux 2023)
- Build Agent: t3.medium EC2 (Amazon Linux 2023)
- Deployment Server: t3.small EC2 (Amazon Linux 2023)
- All instances in the same VPC

---

## Practical Project

The **Landmark Technologies Web Application** is used throughout:
- Full-stack: React frontend + Express/Node.js backend + MongoDB
- Dockerized with multi-stage builds
- Deployed to AWS (EC2 or EKS)
- CI/CD with branch-based deployment strategy
