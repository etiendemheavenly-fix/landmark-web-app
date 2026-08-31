# MODULE 4: JENKINS SECURITY

## Learning Objectives

By the end of this module, students will be able to:
- Configure Jenkins Security Realm (authentication)
- Set up Authorization Strategies (permissions)
- Implement Role-Based Access Control (RBAC)
- Create users and assign permissions
- Understand LDAP/AD integration concepts
- Apply production security hardening

---

## 4.1 Jenkins Security Model

```
┌─────────────────────────────────────────────────────────────┐
│                   JENKINS SECURITY                            │
│                                                             │
│  ┌─────────────────────┐    ┌──────────────────────────┐   │
│  │  AUTHENTICATION     │    │  AUTHORIZATION            │   │
│  │  (Who are you?)     │    │  (What can you do?)       │   │
│  │                     │    │                          │   │
│  │  • Jenkins DB       │    │  • Anyone can do anything│   │
│  │  • LDAP             │    │  • Logged-in users       │   │
│  │  • Active Directory │    │  • Matrix-based          │   │
│  │  • SAML/SSO         │    │  • Project-based Matrix  │   │
│  │  • GitHub OAuth     │    │  • Role-Based (plugin)   │   │
│  └─────────────────────┘    └──────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

Navigate to **Manage Jenkins → Security**

---

## 4.2 Security Realm (Authentication)

The Security Realm determines HOW users authenticate.

### Option 1: Jenkins' Own User Database (Default)

Best for: Small teams, labs, learning

- Jenkins manages its own user accounts
- Users sign up or admin creates accounts
- Passwords stored in Jenkins (hashed)

**Configuration:**
1. Manage Jenkins → Security
2. Security Realm: **Jenkins' own user database**
3. Check/uncheck "Allow users to sign up"

### Option 2: LDAP

Best for: Enterprise environments with existing directory services

**Configuration:**
1. Install **LDAP** plugin
2. Security Realm: **LDAP**
3. Configure:

| Field | Example Value |
|-------|--------------|
| Server | `ldap://ldap.company.com:389` |
| Root DN | `dc=company,dc=com` |
| User search base | `ou=Users` |
| User search filter | `uid={0}` |
| Manager DN | `cn=admin,dc=company,dc=com` |
| Manager Password | (stored securely) |

### Option 3: Active Directory

Best for: Windows/Microsoft environments

**Configuration:**
1. Install **Active Directory** plugin
2. Security Realm: **Active Directory**
3. Configure domain: `company.com`

### Option 4: GitHub OAuth

Best for: Teams already using GitHub

**Configuration:**
1. Install **GitHub OAuth** plugin
2. Create OAuth App in GitHub (Settings → Developer settings → OAuth Apps)
3. Security Realm: **GitHub Authentication**
4. Enter Client ID and Secret

---

## 4.3 Authorization Strategy (Permissions)

The Authorization Strategy determines WHAT authenticated users can do.

### Strategy 1: Anyone Can Do Anything

⚠️ **Never use in production.** No access control at all.

### Strategy 2: Logged-in Users Can Do Anything

All authenticated users have full admin access. Only suitable for small trusted teams.

### Strategy 3: Matrix-based Security

Fine-grained permission control per user/group.

**Screen students will see:**
```
┌──────────────────────────────────────────────────────────────────┐
│ Matrix-based Security                                            │
├───────────────┬───────┬───────┬───────┬───────┬───────┬────────┤
│ User/Group    │Overall│ Job   │ View  │ Agent │SCM    │Credentials│
│               │ Read  │ Build │ Read  │ Config│       │          │
├───────────────┼───────┼───────┼───────┼───────┼───────┼────────┤
│ admin         │  ✅   │  ✅   │  ✅   │  ✅   │  ✅   │   ✅    │
│ developer     │  ✅   │  ✅   │  ✅   │  ❌   │  ✅   │   ❌    │
│ viewer        │  ✅   │  ❌   │  ✅   │  ❌   │  ❌   │   ❌    │
│ Anonymous     │  ❌   │  ❌   │  ❌   │  ❌   │  ❌   │   ❌    │
└───────────────┴───────┴───────┴───────┴───────┴───────┴────────┘
```

### Strategy 4: Role-Based Strategy (Recommended for Production)

Requires the **Role Strategy** plugin. Most flexible and manageable approach.

---

## 4.4 Implementing Role-Based Access Control (RBAC)

### Step 1: Install the Plugin

**Manage Jenkins → Plugins → Available → Search "Role-based Authorization Strategy" → Install**

### Step 2: Enable the Strategy

1. **Manage Jenkins → Security**
2. Authorization: **Role-Based Strategy**
3. Click **Save**

### Step 3: Manage Roles

Navigate to **Manage Jenkins → Manage and Assign Roles → Manage Roles**

#### Global Roles

| Role | Overall Read | Job Build | Job Read | View Read | Agent | Credentials |
|------|:-----------:|:---------:|:--------:|:---------:|:-----:|:-----------:|
| admin | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| developer | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| viewer | ✅ | ❌ | ✅ | ✅ | ❌ | ❌ |

#### Project Roles (Pattern-based)

| Role | Pattern | Job Build | Job Read | Job Configure |
|------|---------|:---------:|:--------:|:-------------:|
| frontend-dev | `landmark-frontend.*` | ✅ | ✅ | ❌ |
| backend-dev | `landmark-backend.*` | ✅ | ✅ | ❌ |
| devops | `.*` | ✅ | ✅ | ✅ |

### Step 4: Assign Roles

Navigate to **Manage Jenkins → Manage and Assign Roles → Assign Roles**

| User | Global Role | Project Role |
|------|-------------|--------------|
| admin | admin | — |
| john | developer | frontend-dev |
| jane | developer | backend-dev |
| intern | viewer | — |

---

## 4.5 Creating Users

### Via UI

1. **Manage Jenkins → Users → Create User**
2. Fill in: Username, Password, Full Name, Email
3. Click **Create User**

### Via Script Console (Bulk)

Navigate to **Manage Jenkins → Script Console**:

```groovy
import jenkins.model.*
import hudson.security.*

def instance = Jenkins.getInstance()
def realm = instance.getSecurityRealm()

// Create users
def users = [
    ['john', 'P@ssw0rd!', 'John Smith', 'john@company.com'],
    ['jane', 'P@ssw0rd!', 'Jane Doe', 'jane@company.com'],
    ['bob', 'P@ssw0rd!', 'Bob Builder', 'bob@company.com']
]

users.each { user ->
    def existingUser = hudson.model.User.getById(user[0], false)
    if (!existingUser) {
        realm.createAccount(user[0], user[1])
        println "Created user: ${user[0]}"
    } else {
        println "User already exists: ${user[0]}"
    }
}
```

---

## 4.6 Production Security Hardening

### Checklist

```
✅ Disable "Allow users to sign up"
✅ Set Authorization to Role-Based Strategy
✅ Set Controller executors to 0 (no builds on controller)
✅ Enable CSRF Protection (enabled by default in Jenkins 2.x)
✅ Enable Agent → Controller Access Control
✅ Use HTTPS (via reverse proxy)
✅ Restrict network access (Security Groups / firewall)
✅ Rotate credentials regularly
✅ Audit user access periodically
✅ Keep Jenkins and plugins updated
✅ Disable Jenkins CLI over remoting
✅ Enable build log masking for secrets
```

### Putting Jenkins Behind a Reverse Proxy (HTTPS)

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│   Browser    │──HTTPS──▶│   Nginx/ALB  │──HTTP──▶│   Jenkins    │
│              │  :443    │              │  :8080   │              │
└──────────────┘         └──────────────┘         └──────────────┘
```

**Nginx config example:**

```nginx
server {
    listen 443 ssl;
    server_name jenkins.company.com;

    ssl_certificate /etc/ssl/certs/jenkins.crt;
    ssl_certificate_key /etc/ssl/private/jenkins.key;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

Update Jenkins URL: **Manage Jenkins → System → Jenkins URL:** `https://jenkins.company.com/`

---

## 4.7 Disabling Dangerous Features

### Disable Script Console for Non-Admins

The Script Console allows arbitrary Groovy execution. Ensure only admins have `Overall/Administer` permission.

### Disable CLI Over Remoting

**Manage Jenkins → Security → CLI:**
- Uncheck "Enable CLI over Remoting"

### Disable Agent → Controller Access

**Manage Jenkins → Security → Agent → Controller Security:**
- Set rules to deny by default

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Locked out after enabling security | Edit `/var/lib/jenkins/config.xml`, set `<useSecurity>false</useSecurity>`, restart |
| LDAP connection failed | Check server URL, port, and bind credentials. Test with `ldapsearch` |
| Users can't see jobs | Check Role assignments — users need both Global Role (Overall/Read) AND Project Role |
| CSRF errors on API calls | Include crumb in API requests: `curl -s http://jenkins/crumbIssuer/api/json` |

---

## Lab: Configure RBAC

**Time:** 20 minutes

1. Install the **Role-based Authorization Strategy** plugin
2. Enable it in **Manage Jenkins → Security**
3. Create three Global Roles: `admin`, `developer`, `viewer`
4. Create three users: `devops-admin`, `dev-john`, `intern-bob`
5. Assign roles:
   - `devops-admin` → admin role
   - `dev-john` → developer role
   - `intern-bob` → viewer role
6. Log in as each user and verify permissions

**Success Criteria:**
- `devops-admin` can configure jobs and system
- `dev-john` can build jobs but not configure system
- `intern-bob` can only view jobs

---

## Interview Questions

1. **Q:** How do you implement RBAC in Jenkins?
   **A:** Install the Role-based Authorization Strategy plugin, enable it in Security settings, define Global Roles and Project Roles with specific permissions, then assign users to roles.

2. **Q:** What do you do if you're locked out of Jenkins?
   **A:** Edit `/var/lib/jenkins/config.xml`, set `<useSecurity>false</useSecurity>`, restart Jenkins, reconfigure security, then restart again.

3. **Q:** How do you secure Jenkins in production?
   **A:** Use HTTPS via reverse proxy, enable RBAC, disable sign-up, set controller executors to 0, restrict network access via security groups, keep plugins updated, and rotate credentials.

---

## Summary

- Authentication (Security Realm) = who can log in
- Authorization (Strategy) = what logged-in users can do
- Use Role-Based Strategy plugin for production RBAC
- Global Roles control system-wide access; Project Roles control per-job access
- Always put Jenkins behind HTTPS in production
- Disable unnecessary features (CLI over remoting, script console for non-admins)
