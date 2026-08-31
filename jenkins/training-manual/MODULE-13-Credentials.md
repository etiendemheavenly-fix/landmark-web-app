# MODULE 13: JENKINS CREDENTIALS MANAGEMENT

## Learning Objectives

- Store and manage secrets securely in Jenkins
- Use different credential types
- Inject credentials into pipelines
- Implement credential rotation

---

## 13.1 Credential Types

| Type | Use Case | Example |
|------|----------|---------|
| **Secret text** | API tokens, single secrets | GitHub PAT |
| **Username with password** | Registry logins | Docker Hub |
| **SSH Username with private key** | Server access | Deployment SSH |
| **AWS Credentials** | AWS services | EKS/ECR access |
| **Secret file** | Config files | kubeconfig |
| **Certificate** | TLS/SSL | Client certificates |

---

## 13.2 Adding Credentials

**Path:** Manage Jenkins → Credentials → System → Global credentials → Add Credentials

### Secret Text (GitHub PAT)
```
Kind: Secret text
Scope: Global
Secret: ghp_xxxxxxxxxxxx
ID: github-token
Description: GitHub Personal Access Token
```

### Username/Password (Docker Hub)
```
Kind: Username with password
Scope: Global
Username: chafah
Password: dckr_pat_xxxxxxxxxxxx
ID: dockerhub-creds
Description: Docker Hub Access Token
```

### SSH Key (Deploy Server)
```
Kind: SSH Username with private key
Scope: Global
Username: ec2-user
Private Key: Enter directly → paste private key content
ID: deploy-server-ssh
Description: Deployment server SSH key
```

### AWS Credentials
```
Kind: AWS Credentials
Scope: Global
Access Key ID: AKIAXXXXXXXXXXXXXXXX
Secret Access Key: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
ID: aws-creds
Description: AWS IAM for EKS
```

---

## 13.3 Using Credentials in Pipelines

### withCredentials Block

```groovy
// Username/Password
withCredentials([usernamePassword(
    credentialsId: 'dockerhub-creds',
    usernameVariable: 'USER',
    passwordVariable: 'PASS'
)]) {
    sh 'echo $PASS | docker login -u $USER --password-stdin'
}

// Secret text
withCredentials([string(credentialsId: 'github-token', variable: 'TOKEN')]) {
    sh 'curl -H "Authorization: token $TOKEN" https://api.github.com/user'
}

// SSH key
sshagent(['deploy-server-ssh']) {
    sh 'ssh ec2-user@10.0.1.100 "docker ps"'
}

// AWS
withAWS(credentials: 'aws-creds', region: 'us-east-1') {
    sh 'aws sts get-caller-identity'
}
```

### Environment Block (Auto-binding)

```groovy
environment {
    DOCKER_CREDS = credentials('dockerhub-creds')
    // Creates: DOCKER_CREDS_USR and DOCKER_CREDS_PSW
}
```

---

## 13.4 Security Best Practices

- Never print credentials in logs (`echo $PASSWORD` → masked automatically)
- Use credential scoping (folder-level vs global)
- Rotate tokens regularly
- Use short-lived credentials where possible (IAM roles > access keys)
- Audit credential usage via Jenkins logs
- Limit who can manage credentials via RBAC

---

## Summary

- Jenkins encrypts all credentials at rest using `secrets/master.key`
- Use `withCredentials` to inject secrets only where needed
- Credentials are masked in console output automatically
- Prefer IAM roles over access keys for AWS
- Rotate credentials periodically
