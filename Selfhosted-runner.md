# Course: Mastering GitHub Self-Hosted Runners 📚

## Overview

This course guides you through understanding, deploying, and maintaining GitHub self-hosted runners. Each module includes concept discussions, hands-on examples, and Q\&A to reinforce learning.

### Who Should Take This Course?

* DevOps engineers seeking control over CI/CD infrastructure
* Developers aiming to optimize GitHub Actions
* System administrators responsible for secure and scalable build environments

---

## Module 1: Introduction to GitHub Runners

### 1.1 What Are Runners?

* **GitHub-hosted runners** vs **Self-hosted runners**
* Benefits and trade-offs of self-hosted

### 1.2 Use Cases

* Building on custom hardware (GPUs, ARM)
* Accessing private networks
* Cost optimization

#### Example 1.1: Comparing Runners

```markdown
| Feature        | GitHub-hosted | Self-hosted     |
|----------------|---------------|-----------------|
| Maintenance    | Managed       | Self-managed    |
| Custom hardware| ❌            | ✅              |
| Free minutes   | 2,000/month   | Unlimited       |
| Security scope | Shared        | Isolated on-prem|
```

**Q\&A**

1. **Q:** Why choose a self-hosted runner over GitHub-hosted?
   **A:** For custom environments, private network access, and unlimited minutes.

---

## Module 2: Architecture & Components

### 2.1 Runner Architecture

* Runner application, GitHub Actions service, Workflows
* Communication over HTTPS

### 2.2 Supported Platforms

* Windows, Linux, macOS
* ARM vs x86

#### Example 2.1: Runner Components Diagram

```plaintext
[GitHub Service] <--HTTPS--> [Runner Service] --> [Build Container/Host]
```

**Q\&A**

1. **Q:** How does the runner communicate with GitHub?
   **A:** Over secure HTTPS using a registration token.

---

## Module 3: Installing a Self-Hosted Runner

### 3.1 Prerequisites

* GitHub repository/organization permissions
* Supported OS and hardware
* Docker (optional)

### 3.2 Registration Process

1. Generate a registration token in repo/org settings.
2. Download runner package.
3. Configure with token.
4. Run `./config.sh` and `./run.sh`.

#### Example 3.1: Linux Runner Setup

```bash
# Download
curl -O -L https://github.com/actions/runner/releases/download/v2.305.0/actions-runner-linux-x64-2.305.0.tar.gz
# Extract
tar xzf actions-runner-linux-x64-2.305.0.tar.gz
# Configure
./config.sh --url https://github.com/owner/repo --token <TOKEN>
# Start
./run.sh
```

**Q\&A**

1. **Q:** Where do you get the registration token?
   **A:** From Settings → Actions → Runners → New self-hosted runner.

---

## Module 4: Configuring Runner Labels & Scaling

### 4.1 Labels

* Defining `--labels` during config
* Using labels in workflows

#### Example 4.1: Labeling a Runner

```bash
./config.sh --url https://github.com/owner/repo --token <TOKEN> --labels selfhosted,Linux,high-mem
```

### 4.2 Autoscaling Patterns

* Using Docker Machine, Kubernetes, or third-party tools
* GitHub's own autoscaling runner offerings

**Q\&A**

1. **Q:** How do labels affect runner selection?
   **A:** Workflows specify `runs-on: [selfhosted, Linux]` to target matching runners.

---

## Module 5: Integrating with GitHub Actions Workflows

### 5.1 Workflow Syntax

* `runs-on` keyword
* Matrix builds with self-hosted

#### Example 5.1: Basic Workflow Using Self-Hosted

```yaml
name: CI
on: [push]
jobs:
  build:
    runs-on: [selfhosted, Linux]
    steps:
      - uses: actions/checkout@v3
      - name: Build
        run: make all
```

### 5.2 Matrix Strategy

```yaml
strategy:
  matrix:
    os: [selfhosted]
    version: [8, 11]
runs-on: ${{ matrix.os }}
```

**Q\&A**

1. **Q:** Can you mix GitHub-hosted and self-hosted in one workflow?
   **A:** Yes, using multiple jobs with different `runs-on` values.

---

## Module 6: Security & Maintenance

### 6.1 Hardening Runners

* Least-privilege principle
* Running in isolated containers

### 6.2 Updating Runners

* Keeping runner software up to date
* Automating updates via cron or systemd timers

#### Example 6.1: Auto-Update Script (Linux)

```bash
#!/bin/bash
cd /path/to/actions-runner
./svc.sh stop
git fetch --tags
LATEST=$(git tag | sort -V | tail -1)
wget https://github.com/actions/runner/releases/download/$LATEST/actions-runner-linux-x64-$LATEST.tar.gz
tar xzf actions-runner-linux-x64-$LATEST.tar.gz
./svc.sh install
./svc.sh start
```

**Q\&A**

1. **Q:** How often should you update self-hosted runners?
   **A:** At least monthly or when security patches are released.

---

## Module 7: Advanced Examples

### 7.1 GPU-accelerated Builds

* Installing NVIDIA drivers & Docker runtime

#### Example 7.1: CUDA Build Workflow

```yaml
runs-on: [selfhosted, gpu]
steps:
  - uses: actions/checkout@v3
  - name: Build with CUDA
    run: |
      nvcc --version
      make cuda
```

### 7.2 Kubernetes-based Runners

* Deploying runner-controller on a k8s cluster
* Dynamic provisioning

**Q\&A**

1. **Q:** What benefits do k8s-runner controllers provide?
   **A:** Scalability and isolation per pod.

---

## Module 8: Monitoring & Troubleshooting

### 8.1 Logs and Diagnostics

* Accessing runner logs in `_diag` folder
* Verbose logging mode `--debug`

#### Example 8.1: Enabling Debug Logs

```bash
./run.sh --once --debug
```

### 8.2 Common Issues

* Token expiration
* Network connectivity
* Permission errors

**Q\&A**

1. **Q:** Why does my runner say "job not found"?
   **A:** Likely label mismatch or expired token.

---

## Final Q\&A

1. **Q:** What command lists all active self-hosted runners in an org?
   **A:** `gh api orgs/{org}/actions/runners` with GitHub CLI.

2. **Q:** How to unregister a runner?
   **A:** `./config.sh remove --token <TOKEN>` or via Settings UI.

3. **Q:** How to rotate tokens?
   **A:** Generate a new token in Settings; decommission old.
