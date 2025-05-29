# Course: Mastering GitHub Self-Hosted Runners 📚

Below is a fully fleshed-out version of each module, expanding on every bullet point with concepts, examples, and practical guidance.

---

## Module 1: Introduction to GitHub Runners

### 1.1 What Are Runners?

**GitHub-hosted runners**

* **Definition:** Virtual machines managed by GitHub, pre-configured with common build tools and languages.
* **Pros:** Zero maintenance, fast start-up, automatically patched.
* **Cons:** Limited free minutes, cannot install custom software or access private networks.

**Self-hosted runners**

* **Definition:** Servers (physical or cloud) that *you* spin up, register with your repo/org, and manage yourself.
* **Pros:** Unlimited minutes, full control over hardware/software, ability to access internal resources.
* **Cons:** You’re responsible for security updates, scaling, and reliability.

### 1.2 Use Cases

1. **Custom hardware**

   * Training machine-learning models on GPUs or building embedded-ARM binaries.
   * Example: A data-science team uses a self-hosted runner with multiple NVIDIA GPUs to parallelize model training.

2. **Private network access**

   * Running integration tests against databases or services behind a corporate VPN.
   * Example: A banking CI pipeline uses self-hosted runners inside the corporate network to test against on-prem systems.

3. **Cost optimization**

   * Avoiding per-minute charges by re-using idle capacity on long-lived servers.
   * Example: An open-source project spins up spot instances overnight to process nightly builds at a fraction of normal cost.

---

## Module 2: Architecture & Components

### 2.1 Runner Architecture

* **GitHub Actions service:** Central orchestrator that stores workflows and schedules jobs.
* **Runner application:** Agent process you install on your machine; it polls GitHub for jobs, executes them, and returns results.
* **Workflows and jobs:** YAML files in your repo that define steps; each job is dispatched to a runner.

```plaintext
[GitHub Actions service] <-- HTTPS --> [Self-hosted runner] --> [Build container or host OS]
```

### 2.2 Supported Platforms

* **Windows, Linux, macOS:** Officially supported; choose based on your build requirements.
* **ARM vs. x86:** ARM runners are ideal for cross-compiling ARM binaries or testing ARM-specific code.

---

## Module 3: Installing a Self-Hosted Runner

### 3.1 Prerequisites

* **Permissions:** You need Admin access to the GitHub repo or org.
* **OS and hardware:** Ensure your OS version is supported (e.g., Ubuntu 20.04, Windows Server 2019).
* **Docker (optional):** For containerized execution, install Docker and grant the runner user permission to control it.

### 3.2 Registration Process

1. **Generate token:** In **Settings → Actions → Runners → New self-hosted runner**, click “Generate token”.
2. **Download package:** Obtain the runner binary (Linux x64, ARM, Windows ZIP, etc.).
3. **Configure:**

   ```bash
   ./config.sh --url https://github.com/owner/repo \
               --token YOUR_TOKEN
   ```
4. **Run:**

   ```bash
   ./run.sh
   ```

   Optionally install as a service for auto-start.

---

## Module 4: Configuring Runner Labels & Scaling

### 4.1 Labels

* When configuring, you can add comma-separated labels that describe capabilities:

  ```bash
  ./config.sh --url ... --token ... --labels selfhosted,Linux,high-mem
  ```
* In workflows, `runs-on: [selfhosted, high-mem]` ensures only runners with that label pick up the job.

### 4.2 Autoscaling Patterns

* **Docker Machine:** Launch new Docker hosts on demand (e.g., AWS EC2)—perfect for ephemeral runners.
* **Kubernetes:** Use the [actions-runner-controller](https://github.com/actions-runner-controller) to spin up runner pods per job.
* **Third-party tools:** Solutions like HashiCorp Nomad or custom Terraform scripts.
* **GitHub’s autoscaling offering:** In private beta, GitHub’s own hosted autoscaling runners.

---

## Module 5: Integrating with GitHub Actions Workflows

### 5.1 Workflow Syntax

* **`runs-on`** keyword selects the runner; an array lets you combine labels.

  ```yaml
  jobs:
    build:
      runs-on: [selfhosted, Linux]
      steps:
        - uses: actions/checkout@v3
        - run: make all
  ```

### 5.2 Matrix Strategy

* Define multiple axes (e.g., JDK versions, OS variants) while still targeting self-hosted runners:

  ```yaml
  strategy:
    matrix:
      jdk: [8, 11]
  runs-on: [selfhosted, Linux]
  steps:
    - name: Set up JDK ${{ matrix.jdk }}
      uses: actions/setup-java@v3
      with:
        java-version: ${{ matrix.jdk }}
  ```

**Mixing runner types:**

* You can have one job on GitHub-hosted, another on self-hosted:

  ```yaml
  jobs:
    test:
      runs-on: ubuntu-latest
      steps: ...
    deploy:
      runs-on: [selfhosted, deploy]
      steps: ...
  ```

---

## Module 6: Security & Maintenance

### 6.1 Hardening Runners

* **Least-privilege principle:** Create a dedicated system user for the runner with minimal rights.
* **Isolation:** Run each job inside a container or VM to limit blast radius.
* **Network controls:** Restrict egress/ingress so runners can’t reach unauthorized services.

### 6.2 Updating Runners

* **Manual updates:** Download the latest release from GitHub and restart the service.
* **Automated updates:**

  ```bash
  #!/bin/bash
  cd /opt/actions-runner
  ./svc.sh stop
  LATEST=$(curl -s https://api.github.com/repos/actions/runner/releases/latest \
           | jq -r .tag_name)
  wget https://github.com/actions/runner/releases/download/$LATEST/...
  tar xzf ...
  ./svc.sh install
  ./svc.sh start
  ```
* **Frequency:** At least monthly, or immediately upon critical security advisories.

---

## Module 7: Advanced Examples

### 7.1 GPU-accelerated Builds

* **Drivers & runtime:** Install NVIDIA drivers and [nvidia-container-toolkit](https://github.com/NVIDIA/nvidia-docker).
* **Workflow snippet:**

  ```yaml
  runs-on: [selfhosted, gpu]
  steps:
    - uses: actions/checkout@v3
    - name: Verify CUDA
      run: |
        nvidia-smi
        nvcc --version
    - name: Build CUDA app
      run: make cuda
  ```

### 7.2 Kubernetes-based Runners

* **actions-runner-controller:**

  * Deploys a Kubernetes Operator that watches Runner custom resources.
  * Automatically creates and scales runner pods per workload.
* **Benefits:**

  * Pod‐per‐job isolation.
  * Effortless horizontal scaling.
  * Leverages existing k8s cluster security and RBAC.

---

## Module 8: Monitoring & Troubleshooting

### 8.1 Logs and Diagnostics

* **Logs location:** The runner’s `_diag` folder contains JSON logs and crash dumps.
* **Verbose mode:**

  ```bash
  ./run.sh --once --debug
  ```

### 8.2 Common Issues

1. **Expired token:** Registration tokens are short-lived; re-generate if you see authentication failures.
2. **Label mismatch:** If a job stays queued, verify that `runs-on` labels exactly match a runner’s labels.
3. **Network errors:** Ensure the runner machine can reach `https://api.github.com` over HTTPS and that firewalls allow egress.

---

## Final Q\&A

1. **List all active runners in an org:**

   ```bash
   gh api orgs/{org}/actions/runners --paginate
   ```
2. **Unregister a runner:**

   * CLI: `./config.sh remove --token <TOKEN>`
   * UI: **Settings → Actions → Runners**, select and remove.
3. **Rotate tokens:**

   * Generate a new token in **Settings → Actions → Runners → New token**, then update your runner configs; decommission the old token immediately.
