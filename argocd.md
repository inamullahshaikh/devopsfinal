# Argo CD & GitOps – Comprehensive Course

## 1. GitOps Foundations

**Concept:** GitOps is an operational paradigm where **Git is the single source of truth** for declarative infrastructure and application definitions.  All desired system state (manifests, configs) is stored in Git, and an automated agent (like Argo CD) continuously reconciles the live cluster state to match Git. Core principles include:

* **Declarative configuration:** Everything (apps, infra) is defined declaratively (YAML/JSON/Kustomize/Helm templates).
* **Version control:** Git stores all config with history, enabling audit trails and rollbacks.
* **Automated reconciliation:** An agent detects drift (live ≠ desired) and applies corrective changes.

Argo CD *implements* GitOps for Kubernetes. As the docs state, “Argo CD follows the GitOps pattern of using Git repositories as the source of truth for defining the desired application state”. When developers push updates to Git, Argo CD automatically (or manually) synchronizes the Kubernetes cluster to that new state, ensuring deployments are **auditable and repeatable**.

**Key Takeaways:** GitOps brings benefits like repeatable deployments, observable history, and self-healing clusters.

<details>
<summary><strong>Practice Q&A</strong></summary>

* **Q:** *What are the three core principles of GitOps?*
  **A:** Declarative config, Git as version-controlled source of truth, and automated agents for reconciliation.

* **Q:** *How does Argo CD embody GitOps?*
  **A:** Argo CD treats Git as the single source of truth for app state; it continuously monitors Git and the live cluster, and syncs any changes back to desired state.

* **Q:** *Why is GitOps beneficial?*
  **A:** It provides **declarativity** (all config in Git), **auditability** (history of changes), and **automation** (agents auto-sync), leading to more reliable and secure deployments.

</details>

## 2. Argo CD Overview & Architecture

Argo CD is *declarative GitOps CD* for Kubernetes. It runs as a set of microservices (controllers) inside its own Kubernetes namespace (usually `argocd`). The main components are:

* **API Server:** Exposes a gRPC/REST API for the UI, CLI, and webhooks. It handles application CRUD, user operations (sync, rollback), repository credentials, cluster credentials (stored as K8s Secrets), and authentication/authorization. It also listens for Git webhook events to trigger syncs.
* **Repository Server:** Maintains a cache of Git repos. Given a repo URL, revision, path, and any templating parameters, it renders the Kubernetes manifests and returns them. This offloads manifest generation (Helm/Kustomize templating) from the controller.
* **Application Controller:** The “operator” that continually watches each Application’s live state in the cluster and compares it to the desired Git state. If an Application is *OutOfSync* (live ≠ target), it can automatically or manually apply changes. It invokes user-defined hooks (PreSync, Sync, PostSync) during sync operations.

&#x20;*Figure: High-level Argo CD architecture (API Server, Repo Server, Application Controller).*

In summary, Argo CD’s **controller-based architecture** separates concerns (API/UI vs. repo templating vs. reconciliation). This modularity makes it flexible and scalable.

<details>
<summary><strong>Practice Q&A</strong></summary>

* **Q:** *Name Argo CD’s three core components and their responsibilities.*
  **A:** (1) **API Server** – serves the UI/CLI API, handles app management, sync/rollback commands, and auth/RBAC. (2) **Repository Server** – clones/caches Git repos and renders manifests (Helm/Kustomize) for given revisions. (3) **Application Controller** – watches apps in the cluster and performs syncs or hooks if live state drifts from the Git-defined state.

* **Q:** *What Argo CD component applies Kubernetes manifests to the cluster?*
  **A:** The **Application Controller**, which runs as a K8s controller, applies or prunes resources to match the desired state.

* **Q:** *How does the API server integrate with Git webhooks?*
  **A:** The API server includes a webhook listener/forwarder that can receive Git provider webhooks. This allows immediate syncs on Git events.

* **Tip:** Remember that Argo CD runs in its own namespace (`argocd`) by default. Pods like `argocd-server`, `argocd-repo-server`, `argocd-application-controller` and `argocd-dex-server` (for SSO) will be created on installation.

</details>

## 3. Installing Argo CD

**Quickstart (Kubernetes):**

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This official manifest installs all Argo CD components into the `argocd` namespace. After deployment, check pods:

```bash
kubectl get pods -n argocd
```

Key pods include: `argocd-server`, `argocd-repo-server`, `argocd-dex-server`, `argocd-application-controller`, and `argocd-redis`.

**Accessing Argo CD:**

* **Web UI:** By default, `argocd-server` is a ClusterIP service. Expose it via port-forward or LoadBalancer. Example:

  ```bash
  kubectl port-forward svc/argocd-server -n argocd 8080:443
  ```

  Then browse `https://localhost:8080`.
* **CLI Login:** Install the `argocd` CLI (download from GitHub releases). Log in with:

  ```
  argocd login <ARGOCD_SERVER> --username admin --password <PASSWORD>
  ```

  The initial **admin** password is the name of a secret:

  ```bash
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
  ```

  Once logged in, you can change the admin password or create new users. (It’s recommended to disable the default `admin` after adding SSO or other users).

**Configuration:** The `argocd-cm` ConfigMap holds settings (config management, SSO connectors, etc), while `argocd-rbac-cm` holds RBAC policies. You can also install via Helm chart or Operator for advanced setups. Always ensure you install compatible CRDs before the main chart if using Helm.

<details>
<summary><strong>Practice Q&A</strong></summary>

* **Q:** *Which namespace is Argo CD installed into by default?*
  **A:** The `argocd` namespace (you create it before applying the install manifest).

* **Q:** *How do you find the initial admin password?*
  **A:** It’s stored in the `argocd-initial-admin-secret` K8s Secret. Use `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d"`.

* **Q:** *What pod names should you see in the `argocd` namespace?*
  **A:** `argocd-server`, `argocd-repo-server`, `argocd-dex-server`, `argocd-application-controller`, `argocd-redis` (and optionally `argocd-applicationset-controller` if installed).

* **Tip:** If you expose Argo CD via Ingress, enable TLS and authentication. Also, by default Argo allows all Ingresses; secure it for production.

</details>

## 4. Declarative GitOps Workflow with Argo CD

With Argo CD, you **declare applications as Kubernetes manifests** in Git. The central CRD is `Application` (or `ApplicationSet` for multi-app). An `Application` manifest looks like this:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/repo.git
    path: k8s/production
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  # Optional: sync policy
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Key fields:

* `project`: Argo **AppProject** name (scopes the app).
* `source.repoURL`: Git repo URL containing manifests/templates.
* `source.path`: directory inside repo (or chart name if using Helm).
* `source.targetRevision`: Git branch, tag, or commit (e.g. `main` or `HEAD`).
* `destination.server`: cluster API (use `"https://kubernetes.default.svc"` for in-cluster or a cluster name).
* `destination.namespace`: target namespace in the cluster.
* `syncPolicy`: optional. If `automated` is enabled, Argo will auto-sync changes.

Argo supports multiple config types: plain YAML, **Kustomize** (if a `kustomization.yaml` is present), **Helm charts** (specify `chart` and `helm` fields), **Jsonnet**, or custom config-management plugins. For example, a Helm app might be:

```yaml
source:
  chart: nginx
  repoURL: https://charts.bitnami.com/bitnami
  targetRevision: 15.9.0
  helm:
    values: |
      replicaCount: 2
      service:
        type: LoadBalancer
```

In this case, Argo uses `helm template` under the hood (it never calls `helm install`); “Helm is only used to inflate charts… the lifecycle is handled by Argo CD”.

### AppProjects (Applications Group)

An **AppProject** groups applications and **scopes permissions**. It can restrict: allowed Git repos (`sourceRepos`), destination clusters/namespaces (`destinations`), and resource kinds. The default project (`default`) is fully permissive. In production, create specific projects. Example CLI to create a project that only allows one repo and namespace:

```bash
argocd proj create team-alpha \
  --description "Team Alpha apps" \
  --dest https://kubernetes.default.svc,team-alpha-namespace \
  --src https://github.com/my-org/team-alpha-apps.git
```

Or as YAML:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-alpha
  namespace: argocd
spec:
  sourceRepos: ["https://github.com/my-org/team-alpha-apps.git"]
  destinations:
  - server: https://kubernetes.default.svc
    namespace: team-alpha-namespace
```

This ensures apps in **team-alpha** project can only use that repo and deploy to that namespace. (Argo enforces these restrictions at sync time.)

<details>
<summary><strong>Practice Q&A</strong></summary>

* **Q:** *What fields must an Argo `Application` include to define a deployment?*
  **A:** It needs `spec.source` (with `repoURL`, `path` or `chart`, `targetRevision`) and `spec.destination` (with `server` and `namespace`), plus a `project` reference.

* **Q:** *How do you enable automatic sync on an Application?*
  **A:** Set `spec.syncPolicy.automated` in the Application manifest or use `argocd app set <app> --sync-policy automated`. Use additional flags `--auto-prune` and `--self-heal` to prune deleted resources and re-sync on drift.

* **Q:** *What is the purpose of an AppProject?*
  **A:** It scopes and namespaces applications. Projects restrict which Git repos, clusters, namespaces, and resource kinds apps can use. Each app must belong to one project.

* **Q:** *Give an example of a source type Argo CD can use.*
  **A:** Argo CD can use Kustomize overlays (detects `kustomization.yaml` in the path), Helm charts (specifying `chart` and repo), plain YAML directories, or Jsonnet, among others.

* **Tip:** Store each `Application` manifest in Git too (declarative setup). You can manage Argo CD config via GitOps by placing the Argo namespace resources (Applications, Projects, ConfigMaps) in a repo and applying them with `kubectl apply`.

</details>

## 5. Syncing, Health, and Status

Argo CD continuously compares the **desired state** (Git) and the **live state** (cluster). The **sync status** can be **Synced** (live matches Git) or **OutOfSync** (drift detected). Per docs: “A deployed application whose live state deviates from the target state is considered `OutOfSync`. Argo CD reports & visualizes the differences… and provides facilities to automatically or manually sync”. The UI and CLI (`argocd app get <app>`) will show:

* **Sync Status:** Synced/OutOfSync, and the Git revision applied.
* **Health Status:** Healthy (all resources meet health checks), Degraded (one or more failing), Missing (e.g. deleted in cluster), or Unknown. Argo has built-in health logic: e.g., Deployments are Healthy when `.status.updatedReplicas = .status.replicas`.

**Syncing:** You can sync an app manually (`argocd app sync <app>`) or automatically with `automated` policy. When syncing:

* Argo will **apply** (or update) the manifests in the cluster.
* If pruning is enabled (`prune: true`), it will also **delete** resources not in Git.
* Sync **waves** and **hooks** control the order (see next section).
* After sync, `argocd app get` should show **Synced** and **Healthy** if all resources stabilized.

**Health/Status:** The Application summary displays counts of resources (Pods, Services, etc) and their statuses. Common troubleshooting statuses:

* *OutOfSync with no diff:* Possibly your Git path has changed branches without changes to tracked files, or ignore-differences is set. Running `argocd app diff <app>` can show what’s different.
* *Degraded:* At least one resource failed readiness or custom health check. Check events or logs for that resource.
* *Missing:* Resource was deleted in Git; if auto-prune is off, it remains in cluster. Enabling prune will remove it.

<details>
<summary><strong>Practice Q&A</strong></summary>

* **Q:** *What does “OutOfSync” mean in Argo CD?*
  **A:** The live state in the cluster differs from the Git-declared desired state (e.g. a manifest changed in Git, or someone manually changed the cluster).

* **Q:** *How does Argo CD determine an app is “Healthy”?*
  **A:** It runs built-in health checks for each resource type (e.g. Deployment’s `.status.updatedReplicas` equals desired replicas, Service has an assigned LB IP, etc). If all resource checks pass, the app is Healthy.

* **Q:** *When will automated sync NOT run for an app?*
  **A:** If the app is already Synced, or if the latest Git SHA was already applied successfully (except if `selfHeal` is enabled). Also, rollback isn’t allowed on an automanaged app.

* **Tip:** Always click **Refresh** in the UI or use `argocd app get` after syncing. Sometimes sync completes but UI may lag. If something looks wrong, `argocd app diff` or `kubectl get pods,...` can reveal the issue.

</details>

## 6. Application Management (Updates & Rollbacks)

Once an `Application` is set up, **updates happen via Git**: pushing a new commit will make the app go OutOfSync. You can then sync the app (or let Argo auto-sync) to apply changes. Key commands:

* `argocd app list` – list all apps and their status.
* `argocd app get <app>` – show detailed status (sync/health) and last 10 revisions.
* `argocd app history <app>` – list deploy history (commit SHAs, sync times).
* `argocd app rollback <app> <revision>` – revert to a previous Git commit or rollout.

Argo CD **records history** of deployments. On rollback, it re-applies the specified revision (if still present in Git). The UI provides a timeline view.

**Example:** Update flow

1. Dev pushes an updated manifest to Git (e.g. increases a Deployment’s replica count).
2. Argo CD detects new revision (via polling or webhook) and marks the app OutOfSync.
3. If `syncPolicy.automated` is enabled, Argo immediately performs the sync; otherwise an operator clicks “Sync” in the UI or runs `argocd app sync`.
4. The controller applies the new manifest (scale-up), and the UI shows the app as Synced (at new commit) and Healthy (once pods are running).

**Common tasks:**

* **Ignoring Differences:** You can configure Argo to ignore specific fields (like `status` or annotations) via `argocd-cm` settings.
* **App Actions:** Custom actions can be defined (e.g. `promote`, `restart`) by Argo's App of Apps pattern or extension.
* **Automated Promotions:** Integrate branch strategies (e.g. merging from `staging` to `production` triggers separate apps).

<details>
<summary><strong>Practice Q&A</strong></summary>

* **Q:** *How do you manually sync an Argo CD app?*
  **A:** Either click **Sync** in the UI or run `argocd app sync <app-name>` in the CLI.

* **Q:** *How do you roll back an application to the previous version?*
  **A:** Use `argocd app history <app>`, find the desired `REVISION`, then run `argocd app rollback <app> <REVISION>`.

* **Q:** *Which command shows the difference between live and Git states?*
  **A:** `argocd app diff <app>` shows a diff of resources between the live cluster and the target revision in Git.

* **Tip:** Keep your Git repo immutable; do not push hotfixes directly to a cluster. Instead, always update Git and let Argo CD apply changes. This ensures the source of truth (Git) stays accurate.

</details>

## 7. Advanced Sync Features (Auto-Sync, Hooks, Waves)

* **Auto-Sync:** Enabling `syncPolicy.automated` lets Argo auto-deploy any Git change. Example:

  ```bash
  argocd app set my-app --sync-policy automated
  argocd app set my-app --auto-prune
  argocd app set my-app --self-heal
  ```

  Or in YAML:

  ```yaml
  syncPolicy:
    automated: 
      prune: true      # delete removed resources
      selfHeal: true   # automatically re-sync on external drift
  ```

  When automated sync is on, ArgoCD will only sync when an application is *OutOfSync*. It will not repeatedly apply the same commit unless `selfHeal` is true (which attempts a re-sync after a short delay). Automated sync *cannot* perform rollbacks.

* **Sync Hooks (Phases):** Use **resource hooks** to run jobs at lifecycle points. Common annotations on K8s resources:

  ```yaml
  metadata:
    annotations:
      argocd.argoproj.io/hook: PreSync    # or PostSync, Sync, PreDelete, etc.
  ```

  For example, a database migration Job annotated `PreSync` will run before the main sync begins. This lets you prepare or clean up resources around deployments.

* **Sync Waves (Ordering):** To control resource application order, annotate resources with `sync-wave`. All resources default to wave **0**. Lower waves apply first. Example:

  ```yaml
  apiVersion: v1
  kind: Namespace
  metadata:
    name: todo
    annotations:
      argocd.argoproj.io/sync-wave: "-1"
  ```

  Sets this Namespace to wave -1 (applied before wave 0). Argo orders resources by (1) *Phase* (hooks phases first), then (2) *sync-wave* (lowest to highest), then *K8s kind order*, then name. Waves can be negative or positive. This is useful, e.g., to create a namespace or CRDs first, then dependent resources.

* **Sync Options:** The CLI and manifest support options such as:

  * `--dry-run` / skip hooks for testing.
  * `--force` or `Replace: true` to overwrite immutable fields.
  * `--timeout` settings for slow resources.
  * RespectIgnoreDifferences: if true, Argo will not auto-sync if only ignored fields changed.

<details>
<summary><strong>Practice Q&A</strong></summary>

* **Q:** *What does `spec.syncPolicy.automated: {}` do?*
  **A:** It tells Argo CD to automatically sync the app whenever it detects OutOfSync (i.e., a new commit or drift).

* **Q:** *How do you run a database migration before deploying an app?*
  **A:** Annotate the migration Job with `argocd.argoproj.io/hook: PreSync`. Argo will execute it before syncing the rest of the manifests.

* **Q:** *What is a sync wave?*
  **A:** A numeric annotation (`argocd.argoproj.io/sync-wave`) on a resource that sets its apply order. Lower numbers apply first. Waves allow ordered rollouts within the same App.

* **Tip:** Be careful with automated pruning. Only enable `auto-prune` if you’re sure resources removed from Git should be deleted. Also, `selfHeal` can cause reconciling loops if misconfigured.

</details>

## 8. Access Control: RBAC and SSO

Argo CD has its own RBAC system (separate from Kubernetes RBAC). By default, only the `admin` user exists (full permissions). To add users, you must configure **SSO or local accounts**. Common steps: enable an external ID provider (GitHub, OIDC, SAML) or define local users in `argocd-cm`, then set RBAC policies in `argocd-rbac-cm`.

**Built-in Roles:** Argo defines `role:admin` (superuser) and `role:readonly` by default. The `policy.default` entry in `argocd-rbac-cm` maps authenticated users to a role (often `role:readonly` in examples).

**Policy Syntax:** Policies follow a Casbin-style CSV. Example in the ConfigMap:

```yaml
data:
  policy.csv: |
    p, my-org:team-alpha, applications, sync, my-project/*, allow
    g, my-org:team-beta, role:admin
    g, user@example.org, role:admin
  policy.default: role:readonly
```

* `p, SUBJECT, RESOURCE, ACTION, OBJECT, EFFECT` defines permissions. E.g. `my-org:team-alpha` (an OIDC group) can `sync` any app in `my-project`.
* `g, SUBJECT, role:NAME` binds a user/group to a role.

**Example:** To grant a group `devs@company` the ability to sync apps in project `devs`:

```
p, devs@company, applications, sync, devs/*, allow
```

This line in `policy.csv` says members of `devs@company` can perform `sync` on any app named `devs/*`. See the RBAC docs for more granular options (e.g. update, delete).

**Local Accounts:** You can also create static user accounts. Example snippet to disable the built-in admin once you have other users:

```yaml
apiVersion: v1
kind: ConfigMap
metadata: 
  name: argocd-cm 
  namespace: argocd
data:
  admin.enabled: "false"
```

. Then use `argocd account` commands to create users or set passwords.

**SSO (Single Sign-On):** Argo CD embeds Dex (an OIDC identity service) for SSO. In `argocd-cm`, you add a `dex.config` section with connectors (GitHub OAuth, LDAP, OIDC, SAML, etc). Alternatively, skip Dex and use an existing OIDC provider (Okta, Google, etc). For example, a GitHub connector config might include `type: github`, your GitHub app’s `clientID` and `clientSecret`. After configuring, users can log in via that identity provider. Groups from the ID token can be mapped in RBAC (via `scopes: [groups]` in `argocd-rbac-cm`).

<details>
<summary><strong>Practice Q&A</strong></summary>

* **Q:** *Where are Argo CD’s RBAC policies defined?*
  **A:** In the `argocd-rbac-cm` ConfigMap (key `policy.csv`) and optionally in AppProject roles.

* **Q:** *What does `role:readonly` allow?*
  **A:** It is a built-in role giving read-only access to all Argo CD resources. It lets you view apps/status but not sync or change them.

* **Q:** *How do you grant a user the right to sync apps in a project?*
  **A:** Add a policy line like `p, alice@example.com, applications, sync, my-project/*, allow` in `argocd-rbac-cm`. This allows Alice to sync any app in `my-project`.

* **Q:** *What do you need to do before enforcing Argo CD RBAC?*
  **A:** Configure user accounts via SSO or local accounts. Argo CD has no built-in user DB (besides admin). Once SSO/local users are set, disable the default `admin` to avoid backdoor access.

* **Tip:** Always restrict `policy.default` as narrowly as possible (e.g. to `role:readonly`) and then explicitly allow needed actions. Use AppProjects to isolate teams. Remember that **Kubernetes RBAC** still controls what Argo’s service account can do in each cluster.

</details>

## 9. Secrets Management

Argo CD does **not** natively encrypt or store application secrets. By default, it treats secrets like any other Kubernetes object (in plaintext YAML). Best practice is to **offload secrets management** to specialized tools. Two main approaches:

* **Cluster-side secrets:** Use operators on the target cluster that handle secrets. For example, [Bitnami Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) or [External Secrets Operator](https://github.com/external-secrets/kubernetes-external-secrets) or [Secret Store CSI driver](https://github.com/kubernetes-sigs/secrets-store-csi-driver) or HashiCorp [Vault Secrets Operator](https://developer.hashicorp.com/vault/docs/platform/k8s/operator). The idea is that you store an **encrypted/sealed secret manifest** in Git (which is safe to commit), and the operator on the cluster decrypts it into a real Secret. Argo CD simply deploys the sealed/encrypted object as usual. Benefits: Argo CD never sees the raw secret, and you can update secrets independently of app releases.

* **Manifest-generation plugins:** Argo supports [config management plugins](https://argo-cd.readthedocs.io/en/stable/operator-manual/config-management-plugins/) that can inject secrets at sync time. A common example is the [argocd-vault-plugin](https://github.com/argoproj-labs/argocd-vault-plugin) which fetches secrets from Vault or AWS Secrets Manager and injects them into templates. **Warning:** This means Argo CD (and its Redis cache) will see the plaintext secrets and store them temporarily. The Argo docs caution against this due to security risks (secrets in logs/redis).

**Example (Sealed Secret):** You might have in Git:

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-password
spec:
  encryptedData:
    password: AgByZ...==
```

Argo CD will apply this, and on the cluster the Sealed Secrets controller will create the real Secret.

In summary, **do not put raw secret values in Git**. Use encryption (SealedSecrets/SOPS) or external secret stores. If using generation plugins, isolate the Argo CD cluster and secure its data (NetworkPolicy, separate cluster) as recommended.

<details>
<summary><strong>Practice Q&A</strong></summary>

* **Q:** *What is the recommended way to manage secrets with Argo CD?*
  **A:** Use cluster-side secret controllers like Sealed Secrets or External Secrets. Commit only the encrypted (sealed) secrets to Git.

* **Q:** *Why is injecting secrets at sync time (via a plugin) risky?*
  **A:** Because Argo CD caches rendered manifests (including secrets) in plaintext in Redis. Anyone with access to Argo’s cache could see the secrets.

* **Tip:** If you must use `argocd-vault-plugin` or similar, ensure only trusted operators have access to the Argo namespace or Redis. Consider running Argo in a separate secure cluster to minimize exposure.

</details>

## 10. Multi-Cluster Support

One Argo CD instance can manage **multiple Kubernetes clusters**. By default, Argo manages the “in-cluster” where it’s installed (`https://kubernetes.default.svc`). To add others:

* **Register clusters:** Use the CLI to add a cluster context:

  ```bash
  argocd cluster add <kubectl-context>
  ```

  This command takes credentials from your kubeconfig and creates a `Secret` with cluster info and a service account for Argo in the target cluster. You’ll need permissions to install on that cluster. Repeat for each cluster.

* **Destination in Applications:** In an Application manifest, set `destination.server` to the API URL or the cluster’s name as registered. For example, `server: https://54.34.22.1` or name given in `argocd cluster add`. This tells Argo which cluster to deploy to.

* **AppProject Restrictions:** In `AppProject.spec.destinations`, you can list allowed clusters. Example YAML to restrict a project to a specific cluster and namespace:

  ```yaml
  spec:
    destinations:
    - server: https://cluster1.example.com
      namespace: team-blue-namespace
    - server: https://cluster2.example.com
      namespace: team-blue-namespace
  ```

  If you try to deploy an app outside these, Argo will deny it.

* **Sharding (HA):** In large setups, you can have multiple Argo instances: one “root” for UI/RBAC and separate for each cluster (using ApplicationSet with Cluster self-registration). This is an advanced pattern (see [Dynamic Cluster Distribution](https://argo-cd.readthedocs.io/en/stable/operator-manual/high_availability/)).

<details>
<summary><strong>Practice Q&A</strong></summary>

* **Q:** *How do you add a new cluster to Argo CD?*
  **A:** Run `argocd cluster add <context>`. This registers the cluster and lets Argo CD deploy apps there.

* **Q:** *What does the special cluster name “in-cluster” refer to?*
  **A:** It refers to the Kubernetes cluster where Argo CD itself is running. This cluster is registered automatically and cannot be removed via `argocd cluster rm`.

* **Tip:** Ensure the Argo CD service account in each cluster has enough RBAC (usually cluster-admin) to create all resources. If a cluster is unreachable or has insufficient creds, syncs will fail with connection errors.

</details>

## 11. Integrations (Helm, Kustomize, CI/CD)

* **Helm:** Argo CD natively supports deploying Helm charts as shown. It uses `helm template` to render resources, then treats them as if they were static YAML. You can specify chart repo, name, version (`targetRevision`), and values files or set parameters in the `Application` spec. Private Helm repos can be configured via repository credentials in Argo or in the `argocd-repo-server` settings.

* **Kustomize:** If your repo path contains a `kustomization.yaml`, Argo automatically runs Kustomize build on it. You can also set Kustomize options (patches, images overrides) in the `Application.spec.source` according to the Kustomize docs.

* **Jsonnet:** Argo can interpret Jsonnet files (if `*.jsonnet` or `.libsonnet` found) using builtin jsonnet engine or plugins.

* **CI/CD Pipelines:** Argo CD is the **CD** part of CI/CD. A typical integration: a CI system (Jenkins, GitHub Actions, GitLab CI, etc.) builds and tests code, and on success **commits configuration to Git** (e.g., updates a manifest or Helm values). Argo CD then detects this Git change and deploys. Alternatively, a pipeline can directly call Argo CD’s API/CLI: for example, after pushing, a GitHub Action might run:

  ```yaml
  - name: Sync to Prod
    run: |
      argocd login $ARGOCD_SERVER --username $ARGOCD_USER --password $ARGOCD_PASS
      argocd app sync my-app
  ```

  However, as the docs note, “CI/CD pipelines no longer need direct access to the Argo CD API server… the pipeline makes a commit and push to Git, and Argo CD does the deployment”.

* **Webhooks:** Configure your Git provider to send Webhook (e.g. on Push) to the Argo CD API (e.g. `http://argocd-server:8080/api/webhook`). Argo’s API server will then trigger an immediate diff/sync. This enables rapid responses instead of waiting for the default polling interval.

<details>
<summary><strong>Practice Q&A</strong></summary>

* **Q:** *How does Argo CD integrate with Helm charts?*
  **A:** You specify the chart name, repo URL, and version in the Application. Argo CD runs `helm template` behind the scenes. Values files or `--set` params can be provided in the spec.

* **Q:** *Does Argo CD replace Helm or Kustomize?*
  **A:** No – it works with them. Argo CD uses Helm/Kustomize to generate manifests, but Argo CD (not Helm) performs deployments. Hence “Helm is only used to inflate charts… lifecycle handled by Argo CD”.

* **Q:** *How can a CI job trigger an Argo CD deployment?*
  **A:** The CI job can push changes to the Git repo tracked by Argo. Or it can call Argo CD CLI/API to issue a sync. Webhook triggers are also common.

* **Tip:** In your pipeline, you may use `argocd login` with an automation token (service account token or API token). Argo CD tokens can be generated with `argocd account generate-token`. Ensure to limit token scope via RBAC.

</details>

## 12. Troubleshooting and Best Practices

* **Check Logs:** If an application fails to sync or shows errors, inspect the Argo CD pod logs: `kubectl -n argocd logs deploy/argocd-application-controller` or `argocd-server`. They often show repo auth or K8s API errors. Also check `argocd-repo-server` logs for template errors.
* **`argocd app diff`:** Use this to see *exactly* what differs between Git and live state. It’s invaluable for diagnosing drift.
* **Cluster Permissions:** A common issue is insufficient permissions on a cluster. Ensure the Argo CD service account on each cluster has permissions to create/update all needed resource types. For example, if Argo cannot create CRDs or cluster roles, syncs will error out.
* **Namespace Issues:** If a manifest refers to a Namespace that doesn’t exist (and auto-creation is off), the sync will fail. Solutions: include the Namespace manifest in Git (Argo will create it) or grant Argo permission to create namespaces.
* **CRD Syncing:** If you’re deploying new CRDs, apply the CRD first (maybe via a dedicated `sync-wave` or manual step) before apps that use them. Otherwise, resources of that CRD will error (unknown type).
* **Version Skew:** Ensure `argocd` CLI is reasonably aligned with the server version. Newer Argo CD servers can talk to older CLI but not vice versa.
* **Common Pitfalls:**

  * Forgetting to target the correct cluster in `destination.server`. If you have multiple clusters, ensure you specify the intended one.
  * Expecting changes to apply without syncing. Even with auto-sync, there can be a short delay (see \[13], reconciliation loop \~2–3 minutes by default). You can force a sync manually if needed.
  * **Exam trap:** Remember that Argo CD itself does not automatically create Git branches or tags; it deploys what is *already* in Git. Git workflows (feature branches, PRs) are external to Argo.

<details>
<summary><strong>Practice Q&A</strong></summary>

* **Q:** *What command can you run to see why an app is OutOfSync?*
  **A:** `argocd app diff <app>`. It shows the delta between Git and live manifests.

* **Q:** *Your app is stuck with “Error: unable to sync” in the UI. What might you check?*
  **A:** Check Argo CD logs and events (for permissions, missing namespace, image pull errors). Also try `argocd app sync` in CLI to see detailed output.

* **Tip:** Keep an eye on resource quotas and limits. A common “unknown error” can be a pod OOM or PVC not bound. Also ensure that images used are accessible (use imagePullSecrets if needed, and configure Argo’s ability to use them on the destination cluster).

</details>

## 13. Key Takeaways & Review

* **GitOps:** Argo CD embodies GitOps – your **git repo = config source of truth**, and Argo CD **automates sync** to clusters.
* **Architecture:** Familiarize with Argo’s three core components (API server, Repo server, App controller). Remember: UI/CLI → API → Controller.
* **Declarative Apps:** Know the structure of an `Application` CRD and `AppProject`. Practice writing YAML for common cases (Kustomize, Helm).
* **Sync & Health:** Understand SyncStatus vs HealthStatus. Recognize “Synced & Healthy” as success. Use `argocd app diff/get/sync` CLI for control.
* **Advanced:** Master auto-sync flags (`prune`, `self-heal`) and hooks/waves for complex deployments.
* **RBAC/SSO:** Use RBAC policies (`policy.csv` in argocd-rbac-cm) to secure Argo. Map your company’s users/groups to roles. Configure SSO via Dex or OIDC.
* **Secrets:** Never commit raw secrets. Use sealed secrets or external stores. Understand the trade-offs of manifest-injection plugins.
* **Multi-Cluster:** Argo CD can manage any number of clusters. Use `argocd cluster add` and AppProjects to control deployments across them.
* **Troubleshoot:** Check logs, use Argo CLI tools, verify permissions. Common exam scenarios include permission errors, namespace issues, and sync configuration mistakes.
* **Integration:** Argo CD works with Helm/Kustomize (templating) and plugs into CI pipelines (git pushes or API calls).
