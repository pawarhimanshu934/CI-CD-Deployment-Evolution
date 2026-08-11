# CI/CD Deployment Evolution: Manual Deployment to GitOps

This document explains the transition from a traditional **manual production deployment process** to a **GitOps-based deployment model** using tools such as **Argo CD or Flux**.

The primary goal is to eliminate the manual handoff between QA and DevOps while ensuring that the **exact artifact tested and approved by QA is promoted to production**.

---

## 1. Approach #1: Manual Deployment Through CloudBees

### Deployment Flow

```text
Developer
   │
   ▼
Push Code to Git
   │
   ▼
CloudBees
   │
   ├── Build
   ├── Scan
   └── Create Docker Image
   │
   ▼
Container Registry
   │
   ▼
Deploy Image to TEST
   │
   ▼
QA Testing
   │
   ▼
QA Approval
   │
   ▼
DevOps Manually Processes/Runs CloudBees Pipeline
   │
   ▼
PRODUCTION
```

### Example

Suppose CloudBees builds the following image:

```text
myapp:abc123
```

The image is deployed to the TEST environment.

QA validates the application and confirms:

```text
Image abc123 is approved for production.
```

At this point, DevOps needs to manually process the production deployment.

The typical handoff looks like:

```text
QA
 │
 │ "Image abc123 is approved"
 ▼
DevOps
 │
 ├── Identify the correct pipeline/job
 ├── Select/process the approved build
 ├── Trigger production deployment
 ▼
PRODUCTION
```

---

## 2. Problems With the Manual Approach

The biggest issue is the **manual handoff between QA and DevOps**.

After QA approves an image, someone from DevOps still needs to perform the production deployment.

### Typical Manual Steps

1. QA informs DevOps that the image has been approved.
2. QA provides the relevant build, pipeline, or job information.
3. DevOps identifies the correct deployment pipeline.
4. DevOps manually triggers or processes the deployment.
5. The application is deployed to production.

### Key Problems

#### 1. Deployment Bottleneck

Production deployment depends on DevOps availability.

If DevOps is unavailable, the deployment may have to wait.

```text
QA Approval
     │
     ▼
Waiting for DevOps
     │
     ▼
Manual Deployment
     │
     ▼
Production
```

#### 2. Longer Deployment Time

There is additional waiting time between QA approval and production deployment because of the manual handoff and execution.

#### 3. Human Error Risk

Manual intervention introduces the possibility of selecting:

* The wrong pipeline
* The wrong build
* The wrong image
* The wrong environment
* An incorrect deployment parameter

For example:

```text
QA approved:
myapp:abc123

DevOps accidentally deploys:
myapp:abc122
```

This can result in production running an artifact that was not approved by QA.

#### 4. Limited Automation

Even though CloudBees handles the CI/build process, production deployment still requires manual intervention.

```text
CI = Automated
CD = Partially Manual
```

#### 5. Scalability Issues

As deployment frequency increases, the manual DevOps handoff becomes increasingly difficult to manage.

For example:

```text
5 deployments/day
     ↓
Manageable manually

50 deployments/day
     ↓
DevOps becomes a bottleneck

100+ deployments/day
     ↓
Manual deployment becomes inefficient
```

#### 6. Fragmented Auditability

The QA approval and production deployment can exist as separate activities.

For example:

```text
QA:
"abc123 approved"

       +

DevOps:
"Production deployment triggered"

       ↓

Audit trail is split across communication
and pipeline activity.
```

---

# 3. Approach #2: GitOps-Based Deployment

The GitOps approach removes the manual production deployment handoff.

The core idea is:

> **QA approves the exact image, automation updates the deployment configuration, and GitOps automatically deploys the approved version.**

### Deployment Flow

```text
Developer
   │
   ▼
Push Code
   │
   ▼
CloudBees
   │
   ├── Build
   ├── Scan
   └── Create Docker Image
   │
   ▼
Container Registry
   │
   ▼
Image: myapp:abc123
   │
   ▼
TEST Environment
   │
   ▼
QA Testing
   │
   ▼
QA Approval
   │
   ▼
Automation Updates Deployment Configuration
   │
   ▼
Git Commit / Pull Request
   │
   ▼
GitOps Controller
(Argo CD / Flux)
   │
   ▼
Kubernetes
   │
   ▼
PRODUCTION
```

---

# 4. What Changes in the GitOps Approach?

Suppose production is currently configured to run:

```yaml
image: myapp:old-version
```

QA tests the new image:

```text
myapp:abc123
```

and approves it.

Automation updates the production deployment configuration:

```yaml
image: myapp:abc123
```

That change is committed to the Git repository containing the deployment configuration.

For example:

```text
Deployment Repository
│
└── production/
    └── deployment.yaml
```

The change might look like:

```diff
- image: myapp:old-version
+ image: myapp:abc123
```

The change is then committed to Git.

---

# 5. How GitOps Deploys the Application

A GitOps controller such as **Argo CD** continuously monitors the deployment repository.

It compares:

```text
Git Desired State
        │
        │
        ▼
   Argo CD / Flux
        │
        │
        ▼
Kubernetes Actual State
```

Git says:

```text
Production should run:

myapp:abc123
```

The GitOps controller ensures that Kubernetes matches that desired state.

```text
Git Repository
      │
      │ Desired State
      ▼
   Argo CD
      │
      │ Reconciliation
      ▼
 Kubernetes
      │
      ▼
myapp:abc123
```

The important concept is that **Git becomes the source of truth for the desired production state**.

---

# 6. Manual vs GitOps Deployment

### Manual Deployment

```text
QA
 │
 │ Approval
 ▼
DevOps
 │
 │ Manual handoff
 ▼
CloudBees Pipeline
 │
 │ Manual execution
 ▼
PRODUCTION
```

### GitOps Deployment

```text
QA
 │
 │ Approval
 ▼
Automation
 │
 │ Update image version
 ▼
Git Repository
 │
 │ Desired state
 ▼
Argo CD / Flux
 │
 │ Reconciliation
 ▼
Kubernetes
 │
 ▼
PRODUCTION
```

The manual process:

```text
QA → DevOps → Manually Run Pipeline → PROD
```

becomes:

```text
QA → Automation → Git → GitOps → Kubernetes → PROD
```

DevOps no longer needs to manually execute every production deployment.

---

# 7. Why GitOps Is Better

## 7.1 Faster Deployment

There is no need to wait for a DevOps engineer to manually process the deployment.

```text
QA Approval
     ↓
Automation
     ↓
Git Update
     ↓
GitOps
     ↓
Production
```

This reduces the time between QA approval and production deployment.

---

## 7.2 Less Human Intervention

The deployment process becomes automated after the required approval.

Instead of:

```text
QA → DevOps → Manual Pipeline Execution → PROD
```

the process becomes:

```text
QA → Automation → GitOps → PROD
```

---

## 7.3 Better Reliability

The exact image approved by QA can be promoted to production.

For example:

```text
QA tested:
myapp:abc123

QA approved:
myapp:abc123

Production:
myapp:abc123
```

This reduces the risk of accidentally deploying a different build.

---

## 7.4 Better Auditability

Git provides a history of deployment configuration changes.

For example:

```text
Commit:
old-version → abc123

Author:
Automation

Timestamp:
2026-08-12 00:30

Repository:
deployment-config
```

This makes it easier to determine:

* What version was deployed?
* When was it deployed?
* What changed?
* Who or what initiated the change?
* What version was previously running?

---

## 7.5 Better Scalability

With GitOps, increasing deployment frequency does not require a corresponding increase in manual DevOps intervention.

```text
5 deployments/day
        ↓
50 deployments/day
        ↓
100 deployments/day
```

The deployment mechanism remains automated.

DevOps can focus on:

* Platform engineering
* Infrastructure
* Reliability
* Security
* Observability
* Deployment tooling
* Automation improvements

rather than manually triggering every deployment.

---

# 8. Clear Separation of Responsibilities

GitOps also creates a cleaner separation of responsibilities.

```text
┌──────────────────────────────────────┐
│                QA                    │
│                                      │
│ Validate application and approve     │
│ the release candidate                │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│             Automation               │
│                                      │
│ Promote the approved image by        │
│ updating deployment configuration     │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│               Git                    │
│                                      │
│ Source of truth for desired state    │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│          Argo CD / Flux              │
│                                      │
│ Reconcile Git state with Kubernetes  │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│             Kubernetes               │
│                                      │
│ Run the desired application version  │
└──────────────────────────────────────┘
```

The responsibilities become:

| Component          | Responsibility                                                     |
| ------------------ | ------------------------------------------------------------------ |
| Developer          | Develop and commit application code                                |
| CloudBees          | Build, scan, and publish the application image                     |
| Container Registry | Store the immutable image                                          |
| QA                 | Test and approve the image                                         |
| Automation         | Promote the approved image by updating Git                         |
| Git                | Store the desired deployment state                                 |
| Argo CD / Flux     | Reconcile Git state with Kubernetes                                |
| Kubernetes         | Run the application                                                |
| DevOps             | Maintain the platform, infrastructure, automation, and reliability |

---

# 9. Most Important GitOps Concept

Don't think of the process as:

> **"Code gets pushed directly to production."**

Think of it as:

> **"The same immutable artifact that passed QA is promoted to production."**

The complete lifecycle is:

```text
Source Code
    │
    ▼
Build
    │
    ▼
Immutable Image
myapp:abc123
    │
    ▼
Container Registry
    │
    ▼
TEST Environment
    │
    ▼
QA Testing
    │
    ▼
QA Approval
    │
    ▼
Promote abc123
    │
    ▼
Git Deployment Configuration
    │
    ▼
GitOps Controller
    │
    ▼
PRODUCTION
```

The important principle is:

```text
BUILD ONCE
    ↓
TEST
    ↓
APPROVE
    ↓
PROMOTE THE SAME ARTIFACT
```

You do **not** rebuild the application separately for production.

For example:

```text
TEST:
myapp:abc123

PROD:
myapp:abc123
```

not:

```text
TEST:
myapp:abc123

PROD:
myapp:xyz789
```

The production environment should run the same artifact that was tested and approved.

---

# 10. Key Takeaways

### Traditional Model

```text
Build
  ↓
Test
  ↓
QA Approval
  ↓
Manual DevOps Handoff
  ↓
Manual Production Deployment
```

Main weakness:

> **Production deployment depends on manual DevOps intervention.**

### GitOps Model

```text
Build
  ↓
Test
  ↓
QA Approval
  ↓
Automated Promotion
  ↓
Git Update
  ↓
GitOps Reconciliation
  ↓
Production
```

Main advantage:

> **The desired production state is stored in Git, and GitOps automatically reconciles Kubernetes to that state.**

### Core Principle

> **Build once, test once, approve once, and promote the same immutable artifact across environments.**

This improves **automation, reliability, auditability, scalability, and deployment speed** while reducing unnecessary manual intervention.
