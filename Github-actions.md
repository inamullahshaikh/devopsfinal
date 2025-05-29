# GitHub Actions and Workflows: A Comprehensive Course

GitHub Actions is GitHub’s built-in CI/CD platform that lets you automate build, test, and deployment pipelines directly from your repository.  Workflows are defined in YAML files in the `.github/workflows` directory and are triggered by events (such as code pushes, pull requests, or scheduled timers) or can be run manually.  Each workflow consists of one or more **jobs**, each running on a separate runner (GitHub-hosted or self-hosted machine).  Jobs contain **steps**, which can either run shell commands (`run:`) or invoke actions (`uses:`). The diagram below illustrates a workflow triggered by an event that launches multiple jobs on different runners (each job is made of ordered steps):

&#x20;*Figure: Event triggers GitHub Actions workflows, which run jobs on runners broken into steps.*

Workflows allow you to automate everything from CI tests to issue labeling. For example, you might configure one workflow to run tests on every push, another to deploy code on every merge, and yet another to add labels whenever a new issue is opened.  In summary, GitHub Actions lets developers define **custom automated processes** (workflows) as YAML files right in their repo, so code changes automatically trigger builds, tests, or deployments without external CI tools.

**Example Workflow (Hello World)**

```yaml
name: CI
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Say Hello
        run: echo "Hello, world!"
```

This simple workflow (placed in `.github/workflows/ci.yml`) runs on every push or PR, checks out the repo, and prints a message.

**Q\&A:**

1. **Q:** What directory must GitHub Actions workflow files be stored in?
   **A:** The `.github/workflows` directory.
2. **Q (MC):** Which of these is a valid trigger for a workflow?

   * A. `push` (correct)
   * B. `onStart`
   * C. `executeWorkflow`
   * D. `runAction`
     *(Correct answer: A – GitHub workflows trigger on events like `push`, `pull_request`, etc.)*
3. **Q:** Explain the difference between a workflow, a job, and a step.
   **A:** A *workflow* is an automated process defined in YAML; it contains one or more *jobs*. A *job* is a set of steps run on the same runner (it can depend on other jobs). A *step* is an individual action: either a shell command (`run:`) or an action invocation (`uses:`).

## Workflow YAML Structure and Syntax

Workflows use YAML syntax (file extension `.yml` or `.yaml`). The top-level keys are typically `name`, `on`, and `jobs`.

* `name:` (optional) – A human-readable name shown in the Actions UI; if omitted, GitHub shows the file path.
* `on:` – Specifies the **events** or conditions that trigger the workflow. This can be a single event (like `push`), multiple events (e.g. `[push, pull_request]`), or a schedule (CRON). Triggers can also filter by branches, tags, or file paths.
* `jobs:` – Under this key you define one or more jobs by an arbitrary ID. Each job must specify a `runs-on` platform and contains a list of `steps`.

Inside a workflow YAML file, indentation is critical. For example:

```yaml
name: Example Workflow
on:
  push:
    branches: [ main, release/* ]
jobs:
  build-job:
    runs-on: ubuntu-latest
    env:
      NODE_ENV: production
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - name: Install dependencies
        run: npm install
      - name: Run tests
        run: npm test
```

Here, the workflow is named “Example Workflow”. It runs on every push to `main` or any `release/` branch. There is one job, `build-job`, which runs on an Ubuntu runner. The job has a job-level environment variable `NODE_ENV` and three steps (checking out code and running commands).

**Key points:** Workflow files **must** reside in `.github/workflows` and use valid YAML. You can define global `env` variables at top level or under a specific job or step. The `jobs.<job_id>.steps` array executes in order.

**Q\&A:**

1. **Q:** In a workflow YAML, under which key do you list individual jobs?
   **A:** Under the `jobs:` key, each subkey is a job ID.
2. **Q (MC):** What file extension must workflow files have?

   * A. `.git`
   * B. `.yaml` or `.yml` (correct)
   * C. `.workflow`
   * D. `.json`
     *(Correct answer: B – Workflow files use YAML and must end in `.yml` or `.yaml`.)*
3. **Q:** How do you define an environment variable `FOO=bar` for just one job?
   **A:** Under that job’s `env` key. For example:

   ```yaml
   jobs:
     example-job:
       runs-on: ubuntu-latest
       env:
         FOO: bar
       steps:
         - run: echo "$FOO"
   ```

.

## Defining Jobs and Steps

A **job** is a set of steps that execute on the same runner. Each job must specify a `runs-on` label (e.g. `ubuntu-latest`, `windows-latest`, `self-hosted`, etc.). Steps inside a job are executed sequentially in the order listed. Because all steps of a job run on the same runner, they share the runner’s file system: for example, a build step can create an artifact that a later step tests.  You can also configure dependencies between jobs using `needs:` so one job waits for another to finish.

A **step** can either run a shell command or invoke an action:

* **`run:`** executes a command in the runner shell (bash, PowerShell, etc.).
* **`uses:`** calls an action (pre-packaged reusable functionality) by specifying its repository and version, e.g. `actions/checkout@v4`.

For example, here’s a job with steps that use both `run` and `uses`:

```yaml
jobs:
  my-job:
    runs-on: ubuntu-latest
    steps:
      - name: Check out code
        uses: actions/checkout@v4
      - name: Use Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
      - name: Install deps
        run: npm install
      - name: Run script
        run: npm run lint
```

This job’s steps will check out the repository, set up Node.js, install dependencies, and run a lint script. Note how each step has a `name:` (for readability in logs).

Because each step runs in the same workspace, later steps can use files produced by earlier ones. You can also set a `continue-on-error` or `timeout-minutes` on steps if needed.

**Q\&A:**

1. **Q:** What are the two ways to define a step in a GitHub Actions job?
   **A:** Using `run:` to execute a shell command, or `uses:` to invoke an action (a reusable piece of code).
2. **Q (MC):** If `jobs.build.runs-on: ubuntu-latest`, what does this specify?

   * A. The operating system for the runner (correct)
   * B. The version of Ubuntu to install
   * C. That the job should only run on self-hosted Linux runners
   * D. The container image to use
     *(Correct answer: A – `runs-on: ubuntu-latest` tells GitHub to use a GitHub-hosted runner with Ubuntu Linux.)*
3. **Q:** How do you make one job wait for another before running?
   **A:** Use the `needs:` keyword. For example:

   ```yaml
   jobs:
     build:
       runs-on: ubuntu-latest
       steps: { … }
     deploy:
       needs: build
       runs-on: ubuntu-latest
       steps: { … }
   ```

   This ensures `deploy` runs only after `build` completes.

## Events and Triggers

Workflows are triggered by **events**. Common events include `push`, `pull_request`, `schedule` (cron), `workflow_dispatch` (manual trigger), `release`, `issue`, and many more. For example, setting `on: push` triggers a workflow on any push to any branch. You can also specify sub-options: for example, `on: push: branches: [main]` triggers only when `main` is pushed to. The documentation lists all possible events and their syntax.

**Example Triggers:**

```yaml
on:
  push:
    branches: [main, release/*]
  pull_request:
    types: [opened, synchronize, reopened]
  schedule:
    - cron: '0 0 * * *'
```

This configuration means: run on pushes to `main` or any `release/` branch, run on pull requests being opened/synced/reopened, and also run daily at midnight UTC (cron).

**Custom Events:** You can also trigger workflows by workflow\_dispatch (manual), or repository events like `issues` or `issue_comment`. Additionally, GitHub provides triggers for external events like repository dispatch or webhook events from integrated apps. Because GitHub Actions is fully integrated with GitHub, you can use any GitHub event (or even a webhook from an external app) as a trigger.

**Q\&A:**

1. **Q:** Which keyword in the workflow YAML specifies the events that trigger the workflow?
   **A:** The `on:` key.
2. **Q (MC):** Which of these is NOT a valid event trigger for GitHub Actions?

   * A. `push`
   * B. `pull_request`
   * C. `workflow_call`
   * D. `onReady` (correct)
     *(Correct answer: D – there is no `onReady` event. `workflow_call` is used for reusable workflows.)*
3. **Q:** How would you trigger a workflow only on tags?
   **A:** Use `on: push: tags: [ 'v*' ]`. For example:

   ```yaml
   on:
     push:
       tags:
         - 'v*'  # triggers on tag names starting with "v"
   ```

## Reusable Workflows

You can factor out complex sets of jobs into **reusable workflows** and call them from other workflows. A reusable workflow is simply another workflow file that uses `on: workflow_call` as its trigger. In the reusable workflow’s `action.yml`, you declare any `inputs:` and `secrets:` it expects. For example:

```yaml
# .github/workflows/reusable-workflow.yml
on:
  workflow_call:
    inputs:
      config-path:
        required: true
        type: string
    secrets:
      token:
        required: true
jobs:
  run-action:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/labeler@v4
        with:
          repo-token: ${{ secrets.token }}
          configuration-path: ${{ inputs.config-path }}
```

This workflow can be *called* by another workflow. In the caller workflow, you use a job with `uses:` pointing to the reusable workflow file and version, and you supply the inputs/secrets under `with:` and `secrets:`:

```yaml
name: Caller Workflow
on: [push]
jobs:
  call-reusable:
    uses: octo-org/example-repo/.github/workflows/reusable-workflow.yml@main
    with:
      config-path: .github/labeler.yml
    secrets:
      token: ${{ secrets.PERSONAL_TOKEN }}
```

Here, the job `call-reusable` uses the reusable workflow `reusable-workflow.yml` from `example-repo` at the `main` branch. It passes the `config-path` input and the `token` secret. When this runs, it executes all the jobs/steps defined in the reusable workflow as if they were in the caller. This lets you reuse common CI/CD logic across repos.

**Key points:** Use `on: workflow_call` in the *called* workflow and a `uses:` reference in the *caller* workflow. The called workflow gets its `github` context from the **caller** and can access the passed inputs/secrets. Outputs can also be defined to pass data back to the caller.

**Q\&A:**

1. **Q:** What keyword must a reusable workflow use under `on:` to indicate it can be called by other workflows?
   **A:** `workflow_call`.
2. **Q:** How do you invoke a reusable workflow in a job?
   **A:** In a job, use `uses: owner/repo/.github/workflows/name.yml@ref`, and provide any `with:` inputs and `secrets:` parameters.
3. **Q:** (Open-ended) What happens if a reusable workflow calls `actions/checkout`? Which repository is checked out?
   **A:** It checks out the repository of the caller (because actions in the called workflow run in the context of the caller).

## Using Marketplace Actions

GitHub Marketplace hosts thousands of pre-built **actions** that you can incorporate into your workflows. To use an action, add a step with the `uses:` syntax specifying its repository and version/tag. For example:

```yaml
- uses: actions/checkout@v4
- uses: actions/setup-node@v4
- uses: actions/javascript-action@v1.0.1
```

These use official GitHub actions to check out code and set up Node.js. Always specify a version or tag (or full commit SHA) to pin to a specific release, ensuring reproducible builds.

You can find actions via the GitHub Marketplace (search by function, e.g. “deploy”, “docker”, “lint”) or by browsing the Actions sidebar in the workflow editor. Marketplace listings include the exact `uses:` line to copy. Actions can be from the same repo (use `uses: ./.github/actions/my-action`), from any public repo (`uses: owner/repo/path@version`), or even Docker containers (`uses: docker://imagename:tag`). For example, you might use a Docker action like `uses: docker://alpine:3.8` to run a container directly.

**Example:**

```yaml
steps:
  - name: Checkout
    uses: actions/checkout@v4     # Checks out repository code:contentReference[oaicite:46]{index=46}
  - name: Setup Python
    uses: actions/setup-python@v4
    with:
      python-version: '3.9'
  - name: Run ESLint
    run: npm run lint
```

In this snippet, two marketplace actions are used (`checkout` and `setup-python`) followed by a lint step.

**Q\&A:**

1. **Q (MC):** How do you specify the version of an action in a workflow?

   * A. `version: v2` under `jobs`
   * B. As part of the `uses:` string, e.g. `@v4` (correct)
   * C. In a separate `uses-version:` key
   * D. GitHub automatically uses the latest version
     *(Correct answer: B – include the version or tag after `@` in the `uses` line.)*
2. **Q:** Why should you pin to a specific action version or commit SHA rather than `@latest`?
   **A:** Pinning prevents unintended updates. If the action’s code changes or is deleted, workflows pinned by SHA still run the same code.
3. **Q (Open):** Where can you find actions to use in your workflows?
   **A:** On the GitHub Marketplace, or via the workflow editor’s Marketplace sidebar.

## Writing Custom Actions (JavaScript and Docker)

If existing actions don’t meet your needs, you can **create custom actions**. There are two main types:

* **JavaScript (Node.js) Actions:** These use Node.js code with the GitHub Actions toolkit. You write a JavaScript program and include an `action.yml` metadata file. The `action.yml` declares inputs, outputs, and specifies `runs.using: 'node16'` (for example) and the entrypoint JavaScript file. Here’s a minimal example of an `action.yml` for a JavaScript action:

  ```yaml
  name: 'Hello World'
  description: 'Greet someone and output the time'
  inputs:
    who-to-greet:
      description: 'Name of the person to greet'
      required: true
      default: 'World'
  outputs:
    time:
      description: 'The time that the greeting was sent'
  runs:
    using: 'node20'
    main: 'index.js'
  ```

  This file defines an input `who-to-greet` and an output `time`, and tells Actions to run `index.js` with Node 20. You would include an `index.js` that reads the input, prints a greeting, and writes the `time` to the `GITHUB_OUTPUT` file.

* **Docker Container Actions:** These package your code in a Docker image. You include a `Dockerfile` and an `action.yml` with `runs.using: 'docker'`. For example:

  **Dockerfile:**

  ```dockerfile
  FROM alpine:3.10
  COPY entrypoint.sh /entrypoint.sh
  ENTRYPOINT ["/entrypoint.sh"]
  ```

  **action.yml:**

  ```yaml
  name: 'Hello Docker'
  description: 'Greet someone from Docker'
  inputs:
    who-to-greet:
      description: 'Name of the person to greet'
      required: true
      default: 'World'
  outputs:
    time:
      description: 'The time the greeting occurred'
  runs:
    using: 'docker'
    image: 'Dockerfile'
    args:
      - ${{ inputs.who-to-greet }}
  ```

  Here, the `action.yml` tells GitHub to build the Docker image from `Dockerfile` and pass the `who-to-greet` input as an argument to the container. Inside `entrypoint.sh`, you might echo “Hello \${1}” and then output the time. The example above shows one input, one output, and uses a Docker container to run the action.

After writing your action code and `action.yml`, you push it to a GitHub repo (often in the same organization or a dedicated actions repo) and tag a release. Then other workflows can use it via `uses: user/repo/path/to/action@v1`.

**Q\&A:**

1. **Q:** In a JavaScript action’s `action.yml`, what does the `runs.using: 'node20'` field specify?
   **A:** It tells GitHub to run the action with Node.js 20, using the Node runtime on the runner.
2. **Q (MC):** Which of these fields is NOT valid in an action’s `action.yml`?

   * A. `inputs:`
   * B. `runs:`
   * C. `permissions:` (correct)
   * D. `outputs:`
     *(Correct answer: C – `permissions:` is not part of action metadata; it belongs in workflow files.)*
3. **Q:** How does a Docker action pass inputs into the container?
   **A:** You define inputs in `action.yml` and list them under `args:`. The inputs become `$1`, `$2`, etc. inside the container. (In the example, `${{ inputs.who-to-greet }}` is passed as an arg.)

## Environment Variables and Secrets

Workflows have access to **environment variables** and **secrets**. GitHub provides default environment variables (like `GITHUB_SHA`, `GITHUB_REF`, etc.) for each workflow run. You can also define your own variables with the `env:` key at the workflow, job, or step level. For instance:

```yaml
env:
  GLOBAL_VAR: hello
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      JOB_VAR: world
    steps:
      - run: |
          echo "Global is $GLOBAL_VAR, Job is $JOB_VAR"
```

This prints “Global is hello, Job is world”. Step-level `env:` works similarly. Custom variables can store non-sensitive configuration data. (Remember: variables set in workflows are not masked – avoid putting secrets in regular `env` variables.)

For **secrets**, GitHub offers a secure store. You create repository/organization secrets (e.g. in repo Settings). In a workflow, secrets are accessed via the `secrets` context: `${{ secrets.MY_SECRET }}`. For example:

```yaml
steps:
  - name: Deploy
    run: deploy.sh
    env:
      API_KEY: ${{ secrets.API_KEY }}
```

This sets the environment variable `API_KEY` inside the runner to the secret’s value. Secrets are never exposed in logs; if a secret is missing, `${{ secrets.X }}` evaluates to empty string. Note that secrets are not automatically passed to workflows from forks, and must be explicitly provided to reusable workflows.

**Accessing Contexts:** You can also use contexts like `${{ github.actor }}` or `${{ github.ref }}` in expressions (these use default env vars under the hood).

**Q\&A:**

1. **Q:** How do you reference a GitHub Actions secret named `TOKEN` in a workflow step?
   **A:** Use `${{ secrets.TOKEN }}`, either in `env:` or in action inputs.
2. **Q (MC):** True or False: Secrets defined in a repository are automatically available in a reusable workflow called by your workflow.

   * A. True
   * B. False (correct)
     *(Correct answer: B – secrets are **not** automatically passed to reusable workflows; they must be passed explicitly or inherited.)*
3. **Q:** How can you limit the permissions of the `GITHUB_TOKEN` to read-only for a workflow?
   **A:** In the workflow YAML, specify:

   ```yaml
   permissions:
     contents: read
   ```

   This uses least privilege instead of the default full write permissions.

## Matrix Builds and Strategy

GitHub Actions supports **matrix strategies** to run a job multiple times with different configurations. A matrix defines variables and their possible values; GitHub then runs the job for every combination of those values. This is ideal for testing across multiple OSes, language versions, etc.

For example:

```yaml
jobs:
  test-matrix:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [14, 16, 18]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm test
```

This will run the `test-matrix` job 9 times (3 OS choices × 3 Node versions), each combination spawning a separate parallel job. The example in \[17] shows a similar matrix: versions 10/12/14 and os ubuntu/windows, producing 6 jobs.

Matrix supports more advanced controls: you can **exclude** certain combinations or **include** additional ones. You can also limit parallelism with `max-parallel` or disable fast-fail with `fail-fast` to let all matrix jobs complete even if one fails. By default, GitHub runs as many matrix jobs in parallel as runners allow.

&#x20;*Figure: Example of a matrix build – the workflow above produces multiple parallel build jobs (Ubuntu/Windows/mac) and then sequential deploy steps (Staging, Review, Production). The matrix jobs run in parallel across platforms.*

**Q\&A:**

1. **Q:** How would you define a matrix to run on Python 3.8 and 3.9 on both Ubuntu and Windows?
   **A:**

   ```yaml
   strategy:
     matrix:
       python-version: [3.8, 3.9]
       os: [ubuntu-latest, windows-latest]
   ```

   This yields 4 combinations.
2. **Q (MC):** What is the maximum number of jobs a matrix can generate in a single workflow run?

   * A. 100
   * B. 256 (correct)
   * C. 512
   * D. Unlimited
     *(Correct answer: B – up to 256 jobs per workflow run.)*
3. **Q:** How do you prevent a matrix job from failing fast on the first error?
   **A:** Set `strategy.fail-fast: false` under the job. This lets remaining matrix jobs run even if one fails.

## CI/CD Pipelines Using GitHub Actions

GitHub Actions excels at implementing full **CI/CD pipelines**. A typical pipeline might build code, run tests, and then deploy on success. You can define multiple jobs for each phase, and use `needs:` to sequence them. For example:

```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: make build
      - name: Archive build
        uses: actions/upload-artifact@v3
        with:
          name: build-artifact
          path: ./build
  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v3
        with:
          name: build-artifact
      - run: make test
  deploy:
    needs: [build, test]
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Production
        uses: some/deploy-action@v1
        with:
          artifact: ./build
```

In this example, a push to `main` triggers three jobs. `build` compiles the code and uploads it as an artifact. `test` waits for `build`, downloads the artifact, and runs tests. `deploy` waits for both build and test to succeed, then uses a deployment action. This implements a simple CI (build + test) and CD (deploy) pipeline.

As \[42] explains, a CI pipeline “runs when code changes and should make sure all changes work... compile your code, run tests, and check that it’s functional,” while a CD pipeline “deploys the built code into production”. With GitHub Actions, you define all this via workflow YAML – no separate CI server is needed.

**Q\&A:**

1. **Q:** What is the purpose of the `needs:` keyword in a multi-job workflow?
   **A:** It makes a job wait for specified other jobs to complete, controlling the execution order (often used to sequence build→test→deploy jobs).
2. **Q (MC):** According to GitHub’s CI/CD philosophy, which of these does a CI pipeline *not* typically do?

   * A. Run tests on code changes.
   * B. Compile code on commit.
   * C. Automatically deploy to production on every commit (correct).
   * D. Validate functionality of changes.
     *(Correct answer: C – deployment is part of CD, not continuous integration.)*
3. **Q:** Why might you want to have separate workflows for “build” vs. “deploy”?
   **A:** To decouple concerns. For example, you may run build/tests on every PR, but only trigger deploy on pushes to main or after PR merges. Also, separate workflows can run on different event triggers or schedules.

## Common Use Cases

**Code Linting:** A common workflow step is linting source code. You might add a job that runs linters (ESLint, flake8, etc.) on your files. Many linting actions exist (e.g. `github/super-linter`). For example:

```yaml
- name: Run ESLint
  uses: github/super-linter@v5
  with:
    filter_pattern: "src/**/*.{js,jsx}"
    config_file: ./.eslintrc.json
```

This uses the community Super-Linter action to check JavaScript files. Alternatively, you can just run a shell command (`run: npm run lint`).

**Unit Testing:** Another staple is running unit tests. For a Node project:

```yaml
- name: Install deps
  run: npm ci
- name: Run tests
  run: npm test
```

For Python:

```yaml
- name: Setup Python
  uses: actions/setup-python@v4
  with:
    python-version: '3.x'
- name: Install requirements
  run: pip install -r requirements.txt
- name: Run pytest
  run: pytest
```

Tests should usually be placed in a `test` job after `build`.

**Deploying to Cloud:** Workflows often include deployment. For example, to deploy a Docker app to AWS ECS (Elastic Container Service), GitHub provides an example workflow. A `on: push` to `main` can build and push to ECR and update ECS:

```yaml
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v2
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-west-2
- name: Build and Push Docker image
  run: |
    docker build -t myapp:$GITHUB_SHA .
    docker tag myapp:$GITHUB_SHA 123456789012.dkr.ecr.us-west-2.amazonaws.com/myapp:latest
    $(aws ecr get-login --no-include-email)
    docker push 123456789012.dkr.ecr.us-west-2.amazonaws.com/myapp:latest
- name: Deploy to ECS
  uses: aws-actions/amazon-ecs-deploy-task-definition@v1
  with:
    task-definition: myapp-task
    container-name: myapp
    image: 123456789012.dkr.ecr.us-west-2.amazonaws.com/myapp:latest
```

This script demonstrates using AWS Actions and the AWS CLI. Likewise, GitHub has guides for Azure Static Web Apps, Google Cloud, and more. Many marketplaces actions exist for cloud platforms (e.g. Azure/static-web-apps-deploy, GoogleCloudPlatform/github-actions, etc.).

**Q\&A:**

1. **Q:** Where would you store an API token used for deployment so that it’s not exposed?
   **A:** In GitHub Secrets. Then reference it with `${{ secrets.MY_TOKEN }}` in the workflow.
2. **Q (MC):** What is the advantage of using `actions/upload-artifact` in a build job?

   * A. It sends logs to the GitHub UI.
   * B. It allows later jobs to download and use the build output (correct).
   * C. It encrypts your code.
   * D. It stores artifacts only in Docker.
     *(Correct answer: B – uploading artifacts lets later jobs download them (e.g. compile once, test later).)*
3. **Q:** Give an example of a GitHub Action used for deployment (open-ended).
   **A:** Answers may vary. Example: `azure/static-web-apps-deploy@v1` for Azure Static Web Apps, or `aws-actions/amazon-ecs-deploy-task-definition@v1` for AWS ECS.

## Best Practices

* **Pin versions:** Specify exact versions (tags or SHAs) for actions and runners. Pin actions to a commit SHA or version tag to ensure stability. For example, use `actions/checkout@v4` instead of `@master`. GitHub recommends even pinning major version numbers.

* **Limit triggers:** Only trigger workflows on the events you need (e.g., `on: push` only on main or specific branches). This avoids unnecessary runs and saves time.

* **Set timeouts:** Workflows default to a 6-hour limit. Set a shorter `timeout-minutes` for jobs that shouldn't run that long (e.g. 30 minutes) so hung jobs don’t waste runner capacity.

* **Restrict permissions:** Apply the principle of least privilege for the default `GITHUB_TOKEN`. If the workflow only needs read access, set `permissions: contents: read` at the top of the YAML.

* **Concurrent runs:** Use `concurrency` to ensure that only one run of a workflow (or job) executes at a time for a given branch or tag. This prevents race conditions when multiple pushes happen quickly (see \[47] for concurrency examples).

* **Avoid secrets in logs:** Never `echo` secrets directly. Use environment variables or `${{ secrets.X }}` safely. GitHub will mask secrets in logs, but it’s best to minimize showing them. Prefer environment variables over passing on the command line.

* **Organize workflows:** Keep workflows focused. Put distinct tasks in separate files (e.g., one workflow for CI on PRs, another for CD on merges). Reuse common code via reusable workflows or composite actions when possible.

* **Use `actions/cache` for dependencies:** Caching dependencies (npm, pip, etc.) speeds up builds. GitHub Actions provides a `actions/cache` action to save and restore caches between runs.

Following these practices helps create reliable, secure, and maintainable workflows.

**Q\&A:**

1. **Q:** Why should you pin a GitHub Action to a specific SHA or tag?
   **A:** To ensure the action code doesn’t change unexpectedly; it makes builds reproducible.
2. **Q:** (MC) What does adding `timeout-minutes: 20` to a job do?

   * A. Retries the job up to 20 times.
   * B. Ensures the job runs for at least 20 minutes.
   * C. Cancels the job if it runs longer than 20 minutes (correct).
   * D. Sets a 20-minute delay before starting.
     *(Correct answer: C – it limits the job to 20 minutes, after which GitHub will kill it if not complete.)*
3. **Q:** Give two ways to speed up workflow runs involving dependency installs.
   **A:** Use `actions/cache` to cache `node_modules` or `~/.m2/repository`, or use pre-built Docker containers with dependencies, or run on more powerful runners. Caching significantly reduces install time on subsequent runs.
