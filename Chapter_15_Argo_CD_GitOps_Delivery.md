# Chapter 15 — Argo CD GitOps Delivery

## 1. Chapter objectives

This chapter documents the Argo CD GitOps work performed in the Vagrant Kubernetes lab. It follows the progression from installing Argo CD to creating an Application, synchronizing an existing Helm-managed application, enabling automated synchronization, and organizing Application resources declaratively in Git.

The main lab repository was:

\`\`\`text
https://github.com/mearvi/gitops-labs.git
\`\`\`

The Helm chart managed by Argo CD was:

\`\`\`text
helm/multi-tier-chart
\`\`\`

The primary Argo CD Application was named \`multi-tier\` and deployed into namespace \`multi-dev\`.

By the end of this chapter, you should be able to:

- Explain GitOps and the desired-state reconciliation model.
- Install Argo CD into a dedicated Kubernetes namespace.
- Verify Argo CD components and access the Argo CD UI safely.
- Explain the Application custom resource and its important fields.
- Connect a Git repository and deploy a Helm chart through Argo CD.
- Distinguish Healthy from Synced and understand OutOfSync.
- Enable automated sync, self-heal, and pruning through an Application manifest.
- Explain how Git commits trigger reconciliation.
- Use an App of Apps structure to manage child Applications.
- Troubleshoot source paths, Application health, sync policy, and ApplicationSet generation.
- Identify which exercises were verified and which remain future work.

---

## 2. Lab environment and verified baseline

| Component | Recorded lab configuration |
|---|---|
| Kubernetes cluster | Vagrant, kubeadm, three nodes |
| Control plane | \`k8master1\` |
| Control-plane address | \`192.168.56.10\` |
| Worker 1 | \`k8worker1\`, \`192.168.56.20\` |
| Worker 2 | \`k8worker2\`, \`192.168.56.21\` |
| Argo CD namespace | \`argocd\` |
| Argo CD version | \`v3.4.5\` recorded in the practice conversation |
| GitOps repository | \`mearvi/gitops-labs\` |
| Branch | \`main\` |
| Application | \`multi-tier\` |
| Application destination namespace | \`multi-dev\` |
| Chart path | \`helm/multi-tier-chart\` |
| Application project | \`default\` |
| Destination cluster | \`https://kubernetes.default.svc\` |

The three-node cluster baseline is retained from the preceding chapters. Do not confuse this final cluster with earlier abandoned cluster layouts.

### 2.1 Existing application before Argo CD adoption

Before the first Argo CD sync, the multi-tier application had already been installed with Helm:

\`\`\`bash
helm install multi-app . -n multi-dev
\`\`\`

The existing resources were in namespace \`multi-dev\`. A captured Pod listing showed:

- Two backend Pods in Running state.
- Two frontend Pods in Running state.
- PostgreSQL Pod \`postgres-0\` in Running state.

This distinction is important: Argo CD initially discovered an application whose Kubernetes resources already existed. The first Argo CD sync was an adoption/reconciliation event, not the original Helm installation.

---

## 3. What is GitOps?

GitOps is an operational model in which the desired state of an application or platform is stored declaratively in Git. A controller continuously compares that desired state with the live Kubernetes state and works to reconcile differences.

In the earlier manual deployment model, an engineer runs:

\`\`\`bash
kubectl apply -f deployment.yaml
\`\`\`

or:

\`\`\`bash
helm upgrade --install multi-app ./multi-tier-chart -n multi-dev
\`\`\`

The operator must remember which version was applied and repeat the process consistently.

In a GitOps model:

1. The desired configuration is committed to Git.
2. Argo CD observes the repository revision.
3. Argo CD reads and renders the desired resources.
4. Argo CD compares the desired resources with live Kubernetes resources.
5. Argo CD reports whether the application is Synced or OutOfSync.
6. Depending on sync policy, Argo CD applies the required changes automatically or waits for an operator to synchronize.

Git becomes the source of truth for the resources managed by the Application.

### 3.1 Traditional deployment versus GitOps

\`\`\`text
Traditional deployment

Engineer
   |
   v
kubectl apply / helm upgrade
   |
   v
Kubernetes cluster
\`\`\`

\`\`\`text
GitOps deployment

Engineer
   |
   v
Git commit and push
   |
   v
Git repository
   |
   v
Argo CD observes and compares desired state
   |
   v
Kubernetes API
   |
   v
Live workloads converge to Git
\`\`\`

### 3.2 Why GitOps is useful

- Git provides a reviewable history of desired configuration.
- Changes can be reviewed through normal pull-request workflows.
- Reconciliation reduces configuration drift.
- A deployment can be tied to a specific Git revision.
- The desired state can be recreated after a cluster rebuild.
- Multiple applications and environments can be managed consistently.

GitOps does not eliminate the need for access control, backups, testing, or safe change review. It makes the desired state and its change history explicit.

---

## 4. Argo CD architecture

Argo CD is a declarative, Kubernetes-native continuous delivery controller. It watches configured Git repositories and reconciles the resources described by Argo CD Application objects.

### 4.1 Core components

The installation in the lab showed these main components in namespace \`argocd\`:

| Component | Responsibility |
|---|---|
| \`argocd-server\` | API server and web UI |
| \`argocd-repo-server\` | Clones repositories and generates manifests |
| \`argocd-application-controller\` | Compares desired and live state and performs sync operations |
| \`argocd-applicationset-controller\` | Generates Applications from ApplicationSet resources |
| \`argocd-dex-server\` | Identity integration component |
| \`argocd-redis\` | Internal cache |
| \`argocd-notifications-controller\` | Notification processing |

The exact component deployment arrangement may vary by Argo CD version and installation mode.

### 4.2 Mermaid architecture diagram

\`\`\`mermaid
flowchart TD
    DEV[Developer changes Helm values or manifests]
    GIT[GitHub repository: gitops-labs, main]
    API[Argo CD Application custom resource]
    REPO[Repo server: clone repository and render Helm]
    CTRL[Application controller: compare desired and live state]
    DIFF{Desired state equals live state?}
    SYNC[Sync operation applies rendered resources]
    K8S[Kubernetes API]
    LIVE[Deployments, Services, StatefulSets, Ingress, Pods]
    STATUS[Application status: Synced / OutOfSync and Healthy / Degraded]
    DRIFT[Manual cluster change creates drift]
    HEAL[Self-heal reconciliation]
    PRUNE[Prune removes resources no longer declared in Git]

    DEV -->|git commit and push| GIT
    GIT -->|repository revision| REPO
    API -->|repo, path, revision, destination, policy| CTRL
    REPO -->|rendered desired manifests| CTRL
    CTRL --> DIFF
    DIFF -->|No difference| STATUS
    DIFF -->|Difference and sync allowed| SYNC
    SYNC --> K8S
    K8S --> LIVE
    LIVE --> CTRL
    LIVE -->|manual kubectl edit/scale| DRIFT
    DRIFT --> CTRL
    CTRL -->|selfHeal enabled| HEAL
    HEAL --> K8S
    GIT -->|resource removed from desired state| PRUNE
    PRUNE --> K8S
    STATUS -->|status visible in UI and CLI| API
\`\`\`

### 4.3 Pull-based reconciliation

Argo CD normally runs inside the cluster and connects outward to the configured Git repository. It does not require GitHub to push deployment credentials into the cluster.

This is a key difference from a CI workflow that directly runs \`kubectl apply\` from an external runner.

The GitOps repository is the source of desired state. The Argo CD controller performs the comparison and applies changes through the Kubernetes API.

---

## 5. Installing Argo CD

Argo CD was installed into its own namespace.

### 5.1 Create the namespace

\`\`\`bash
kubectl create namespace argocd
\`\`\`

### 5.2 Apply the official installation manifest

The practice used the stable installation manifest:

\`\`\`bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
\`\`\`

The conversation records Argo CD version \`v3.4.5\` during the later configuration work. The installation command above uses the moving \`stable\` manifest URL rather than a version-pinned URL. For repeatable production installations, pin and review a specific Argo CD release manifest.

Wait for the components to become ready before configuring Applications.

### 5.3 Verify Pods

\`\`\`bash
kubectl get pods -n argocd
\`\`\`

The captured output showed the Argo CD components in Running state, including:

\`\`\`text
argocd-application-controller-0                         1/1 Running
argocd-applicationset-controller-7b8ccc5477-xpd5j       1/1 Running
argocd-dex-server-64f9f64b68-t25xq                       1/1 Running
argocd-notifications-controller-8656595f88-cvwws        1/1 Running
argocd-redis-59db66d7d4-wd7zn                           1/1 Running
argocd-repo-server-5864959f88-vxz7c                     1/1 Running
argocd-server-564847b6b7-cllbd                          1/1 Running
\`\`\`

The captured output also included restart counts. A Pod can be Running and Ready while having restarted previously; investigate repeated restarts with \`kubectl describe pod\` and logs.

### 5.4 Verify Services and Deployments

\`\`\`bash
kubectl get svc -n argocd
kubectl get deployments -n argocd
kubectl get statefulsets -n argocd
\`\`\`

The lab conversation instructed checking Services and Deployments, but did not preserve a complete final service listing or a verified external IP/NodePort value. Therefore, do not assume a particular Argo CD UI exposure method or address from this chapter.

### 5.5 Inspect the server service before accessing the UI

\`\`\`bash
kubectl get svc argocd-server -n argocd -o wide
kubectl describe svc argocd-server -n argocd
\`\`\`

For a local-only learning session, a port-forward is a simple access method when the server Service is available:

\`\`\`bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
\`\`\`

Then browse to:

\`\`\`text
https://localhost:8080
\`\`\`

This port-forward is a general access option, not a claim that it was the exact exposure method used in the recorded session. A browser certificate warning can occur with the default self-signed certificate.

---

## 6. Argo CD initial login and credential safety

Argo CD installations commonly provide an initial \`admin\` account. The initial password is sensitive and must never be stored in Git or pasted into public documentation.

A common initial-password retrieval command is:

\`\`\`bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
echo
\`\`\`

This command is provided as a reference for a fresh installation. It does not imply that the original lab password is known or reproduced here.

If the initial secret is no longer present, do not assume the initial password can be retrieved again. Use the documented password reset or account recovery process for the installed Argo CD version.

### 6.1 CLI login example with placeholders

If the Argo CD CLI is installed and the server is reachable:

\`\`\`bash
argocd login <ARGOCD_SERVER_ADDRESS> \
  --username admin \
  --password '<ARGOCD_ADMIN_PASSWORD>' \
  --insecure
\`\`\`

The \`--insecure\` option skips certificate verification and is suitable only for a controlled lab using a self-signed certificate. Prefer a trusted TLS certificate and normal certificate verification in production.

### 6.2 Credential rules

- Never commit the Argo CD admin password.
- Never commit a Git personal access token.
- Never paste private repository credentials into an Application manifest.
- Use Argo CD repository credentials or supported secret integration for private repositories.
- Rotate credentials that may have been exposed.
- Prefer scoped credentials and least-privilege access.
- Do not put real secrets into Helm values files stored in a public repository.

The lab repository URL used in the recorded Application was public. No Git credential was required in the Application manifest shown below.

---

## 7. Connect the Git repository and understand an Application

An Argo CD Application is a Kubernetes custom resource. It describes:

- Which Git repository contains the desired state.
- Which branch, tag, or commit should be tracked.
- Which directory contains the manifests or Helm chart.
- Which Kubernetes cluster is the destination.
- Which namespace receives the resources.
- Whether synchronization is manual or automated.

The lab initially created the Application through the UI to learn the concepts, then moved toward declarative YAML.

### 7.1 Initial UI Application settings

The initial Application created in the UI was configured with:

| Field | Value |
|---|---|
| Application name | \`multi-tier\` |
| Project | \`default\` |
| Sync policy initially | Manual |
| Repository URL | \`https://github.com/mearvi/gitops-labs.git\` |
| Revision | \`main\` |
| Chart path | \`multi-tier-chart\` in the initial chart layout |
| Destination cluster | \`https://kubernetes.default.svc\` |
| Destination namespace | \`multi-dev\` |

At creation, the Application initially appeared as OutOfSync and Missing because Argo CD had not yet synchronized the desired resources. The resources already existed because they had previously been installed with Helm.

### 7.2 Application custom resource

The final recorded Application configuration used the repository's later directory structure:

\`\`\`yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: multi-tier
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/mearvi/gitops-labs.git
    targetRevision: main
    path: helm/multi-tier-chart
    helm:
      valueFiles:
        - ../../environments/dev/values.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: multi-dev

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
\`\`\`

The manifest above reflects the final recorded Application fields. The repository path and values-file path are tied to the directory layout used in the later practice. Verify that the files exist at those exact paths in the Git revision you intend to deploy.

### 7.3 Important Application fields

| Field | Purpose |
|---|---|
| \`apiVersion\` | Argo CD Application API version |
| \`kind\` | Declares an Application resource |
| \`metadata.name\` | Application name |
| \`metadata.namespace\` | Namespace containing the Argo CD Application CR |
| \`spec.project\` | Argo CD AppProject |
| \`source.repoURL\` | Git repository |
| \`source.targetRevision\` | Branch, tag, or commit |
| \`source.path\` | Directory in the repository |
| \`source.helm.valueFiles\` | Helm values files |
| \`destination.server\` | Target Kubernetes API |
| \`destination.namespace\` | Target namespace |
| \`syncPolicy.automated\` | Enables automated synchronization |
| \`prune\` | Removes resources deleted from desired Git state |
| \`selfHeal\` | Reconciles live changes that differ from Git |
| \`CreateNamespace=true\` | Allows Argo CD to create the destination namespace during sync |

### 7.4 Apply an Application manifest

For a local manifest file:

\`\`\`bash
kubectl apply -f applications/apps/multi-tier.yaml
\`\`\`

The file path above is an example based on the later repository layout. Use the actual path in the checked-out repository.

Verify:

\`\`\`bash
kubectl get applications -n argocd
kubectl get application multi-tier -n argocd -o yaml
\`\`\`

The Application CR itself lives in the \`argocd\` namespace. The application workloads are deployed into \`multi-dev\`.

---

## 8. First synchronization: Healthy versus Synced

One of the most important lessons from the lab was that **Health** and **Sync** are different status dimensions.

### 8.1 Healthy but OutOfSync

The existing multi-tier application was already running. Its captured Pods showed frontend, backend, and PostgreSQL resources in Running state.

Argo CD initially showed:

- Health: \`Healthy\`
- Sync: \`OutOfSync\`

Why?

- Kubernetes resources were running and healthy.
- Those resources had been created earlier using Helm.
- Argo CD had discovered the desired chart in Git.
- The Application had not yet completed its first sync operation.

Healthy does not mean the live resources match Git. It means Argo CD considers the application resources healthy according to their health assessment.

### 8.2 Manual first sync

The first synchronization was deliberately performed manually:

1. Open the \`multi-tier\` Application.
2. Inspect Application details and the resource comparison.
3. Select **SYNC**.
4. Review the default sync options.
5. Select **SYNCHRONIZE**.
6. Wait for the operation to complete.

The captured result showed:

\`\`\`text
Health: Healthy
Sync:   Synced
Last Sync: Sync OK
\`\`\`

A later screenshot summary showed the Application as Healthy and Synced to branch \`main\`, with a recorded short revision \`1afbfbe\`.

### 8.3 How Argo CD handles Helm charts

For the Helm-based Application, Argo CD reads the chart and renders Kubernetes manifests internally. It then compares the rendered resources with the live cluster.

The important distinction:

- Argo CD does not need to run the Helm release lifecycle command \`helm install\` or \`helm upgrade\` for this Application.
- Argo CD uses Helm as a manifest-generation tool and manages the resulting Kubernetes resources through its reconciliation process.

This explains why the Application can manage a Helm chart without the CI runner executing a Helm upgrade command.

### 8.4 Inspect the Application from Kubernetes

\`\`\`bash
kubectl get application multi-tier -n argocd
kubectl describe application multi-tier -n argocd
kubectl get application multi-tier -n argocd -o yaml
\`\`\`

A captured later Application status included:

- Health status: \`Healthy\`
- Sync policy: automated
- Operation phase: \`Succeeded\`
- Operation message: \`successfully synced (all tasks run)\`
- Revision: \`5cc2d17cb6ae6e9f02bec5aca8a1dde35ccec39c\`
- Initiator: automated

The status history is evidence of a successful automated sync operation for that revision.

---

## 9. Automated synchronization

Manual sync requires an operator to click Sync after Argo CD detects a difference. Automated sync removes that manual step for eligible changes.

### 9.1 Manual sync flow

\`\`\`text
Git commit
   |
   v
Argo CD detects new revision
   |
   v
Application becomes OutOfSync
   |
   v
Operator reviews the diff
   |
   v
Operator clicks Sync
   |
   v
Kubernetes resources updated
\`\`\`

### 9.2 Automated sync flow

\`\`\`text
Git commit and push
   |
   v
Argo CD detects new revision
   |
   v
Argo CD renders desired manifests
   |
   v
Argo CD compares desired and live state
   |
   v
Automated sync operation
   |
   v
Kubernetes resources converge to Git
\`\`\`

### 9.3 Enable automated sync declaratively

The lab initially used manual sync to understand the first reconciliation. Automated sync, prune, and self-heal were then configured in the Application resource.

The captured patch command was:

\`\`\`bash
kubectl patch application multi-tier \
  -n argocd \
  --type merge \
  -p '
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - PrunePropagationPolicy=orphan
'
\`\`\`

The captured output was:

\`\`\`text
application.argoproj.io/multi-tier patched
\`\`\`

The configuration was verified from the Application YAML:

\`\`\`bash
kubectl get application multi-tier -n argocd -o yaml \
  | grep -A5 syncPolicy
\`\`\`

The recorded output contained:

\`\`\`yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
    - PrunePropagationPolicy=orphan
\`\`\`

A later captured Application manifest also included \`CreateNamespace=true\` for the multi-environment Application configuration.

### 9.4 Why configure the Application CR instead of only the UI?

The lab encountered a situation where the UI did not persist the automated policy as expected. Inspection showed that the Application's \`spec.syncPolicy\` did not contain an \`automated\` section.

The lesson was to inspect the Kubernetes custom resource, not rely only on the UI display. The UI, CLI, and \`kubectl\` are clients that interact with the Kubernetes API. The Application CR's \`spec\` is the declarative configuration to verify.

---

## 10. Git commit triggers an automatic deployment

The Helm chart was parameterized so that the backend replica count came from \`values.yaml\` rather than being hard-coded in the Deployment template.

The recorded values included:

\`\`\`yaml
backend:
  replicaCount: 2
\`\`\`

The backend Deployment template used a Helm value:

\`\`\`yaml
replicas: {{ .Values.backend.replicaCount }}
\`\`\`

This is essential. If the Deployment template hard-codes \`replicas: 2\`, changing a Helm values file cannot change the rendered replica count.

### 10.1 The practice change

The recorded automatic deployment exercise changed the backend replica count from 2 to 3 in the Git repository:

\`\`\`yaml
backend:
  replicaCount: 3
\`\`\`

Then the change was committed and pushed:

\`\`\`bash
git add .
git commit -m "Scale backend to 3 replicas"
git push origin main
\`\`\`

The intended verification was to watch the Argo CD UI and the Deployment without running \`helm upgrade\`, \`kubectl apply\`, or clicking Sync.

### 10.2 What was confirmed

The user explicitly confirmed that this automatic Git-driven deployment had been performed and completed, but screenshots and terminal outputs were not saved.

The practice record later captured the Application as:

- \`Healthy\`
- \`Synced\`
- Auto Sync enabled
- Synced to \`main\` at short revision \`1afbfbe\`

The Application status history also contains a successful automated sync operation for a later revision.

Therefore, the automatic Git-to-Argo-CD synchronization was completed in the lab. The exact backend Pod names and final 3/3 Deployment output were not preserved as a terminal capture, so they are not presented as a quoted verified output here.

### 10.3 Verify the same workflow in a fresh session

\`\`\`bash
kubectl get application multi-tier -n argocd
kubectl get deployment backend -n multi-dev
kubectl get pods -n multi-dev
kubectl get application multi-tier -n argocd -o yaml
\`\`\`

Expected replica count after the Git values change is three. Confirm the live value and the tracked Git revision before concluding that the rollout is complete.

---

## 11. Drift detection

Drift means the live Kubernetes state differs from the desired state declared in Git.

For example:

Git declares:

\`\`\`yaml
backend:
  replicaCount: 3
\`\`\`

An operator then manually scales the Deployment:

\`\`\`bash
kubectl scale deployment backend \
  --replicas=5 \
  -n multi-dev
\`\`\`

Now the cluster has five replicas while Git still declares three. Argo CD detects that the live state no longer matches the desired state and reports the Application as OutOfSync.

### 11.1 Drift detection versus self-heal

These are related but distinct features:

- **Drift detection** identifies the difference between desired and live state.
- **Self-heal** allows automated reconciliation to correct live changes that were not made in Git.

With automated sync enabled but self-heal disabled, a live-only change may be reported as OutOfSync without the controller automatically reverting it in all cases. With self-heal enabled, Argo CD attempts to restore the Git-defined configuration.

### 11.2 Lab drift demonstration: proposed commands

The source conversation planned this demonstration:

\`\`\`bash
kubectl scale deployment backend \
  --replicas=5 \
  -n multi-dev

kubectl get deployment backend -n multi-dev -w
\`\`\`

With Git declaring three replicas and self-heal enabled, the expected sequence is:

\`\`\`text
Manual scale: 3 -> 5
        |
        v
Argo CD detects OutOfSync
        |
        v
Self-heal reconciliation
        |
        v
Deployment returns to 3 replicas
\`\`\`

**Evidence qualification:** The conversation documents this as the next intended self-heal exercise. It does not preserve a terminal output showing the manual scale to five followed by Argo CD restoring three. Do not treat the expected sequence as a captured completed test.

### 11.3 Inspect drift

\`\`\`bash
kubectl get application multi-tier -n argocd
kubectl describe application multi-tier -n argocd
kubectl get deployment backend -n multi-dev
\`\`\`

In the UI, open the Application and inspect the resource tree or diff view. Compare the desired replica count rendered from Git with the live Deployment spec.

### 11.4 Deleting a managed resource

Another proposed demonstration was to delete the backend Deployment manually:

\`\`\`bash
kubectl delete deployment backend -n multi-dev
\`\`\`

If the resource remains in Git and self-heal is enabled, Argo CD should detect the missing resource and recreate it during reconciliation.

This was described as a planned exercise, not a captured successful deletion-and-recreation result.

---

## 12. Prune: remove resources deleted from Git

Prune handles the opposite direction: a resource is removed from the desired manifests in Git, but it still exists in the live cluster.

For example:

1. A Service is declared in a Helm chart.
2. The Service is deployed to Kubernetes.
3. The Service template is removed from Git.
4. The Git change is committed and pushed.
5. Argo CD detects that the resource is no longer part of the desired state.
6. With automated prune enabled, Argo CD can delete the obsolete live resource.

### 12.1 Why prune needs care

Pruning is potentially destructive. Removing a manifest from Git can cause its live Kubernetes resource to be deleted.

Before enabling prune broadly:

- Review pull requests carefully.
- Understand ownership of resources.
- Protect persistent data and critical resources.
- Use separate environments and approvals for production.
- Test deletion behavior with a non-critical resource.
- Understand finalizers and cascading deletion behavior.

### 12.2 Recorded sync option

The Application patch preserved this option:

\`\`\`yaml
syncOptions:
  - PrunePropagationPolicy=orphan
\`\`\`

This controls the Kubernetes deletion propagation behavior for resources pruned by Argo CD. It is not the same as disabling prune.

### 12.3 Evidence qualification

The Application's \`prune: true\` setting was explicitly patched and verified in the Application spec. The conversation describes removing a Service template and observing automatic deletion as a planned exercise. It does not preserve a successful prune output. The configuration is verified; the end-to-end deletion test is not.

---

## 13. Declarative Applications and App of Apps

The first Application was created through the UI. The next step was to move Application definitions into Git so that the delivery configuration itself could be version-controlled.

The repository was reorganized to separate:

- Child Application manifests.
- The root Application manifest.
- Helm charts.

### 13.1 Repository layout

The recorded layout evolved toward:

\`\`\`text
gitops-labs/
├── applications/
│   ├── apps/
│   │   └── multi-tier.yaml
│   └── root/
│       └── root.yaml
├── helm/
│   └── multi-tier-chart/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
└── environments/
    ├── dev/
    │   └── values.yaml
    ├── qa/
    ├── stage/
    └── prod/
\`\`\`

This structure keeps the root Application outside the directory it manages. Otherwise, the root Application may attempt to manage itself recursively.

### 13.2 Child Application manifest

The child Application file was placed under:

\`\`\`text
applications/apps/multi-tier.yaml
\`\`\`

A representative form of the recorded child Application is:

\`\`\`yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: multi-tier
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/mearvi/gitops-labs.git
    targetRevision: main
    path: helm/multi-tier-chart
    helm:
      valueFiles:
        - ../../environments/dev/values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: multi-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
\`\`\`

### 13.3 Root Application manifest

The root Application was stored separately at:

\`\`\`text
applications/root/root.yaml
\`\`\`

Its source path points to the directory containing child Application manifests:

\`\`\`yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/mearvi/gitops-labs.git
    targetRevision: main
    path: applications/apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
\`\`\`

This root Application is bootstrapped with:

\`\`\`bash
kubectl apply -f applications/root/root.yaml
\`\`\`

After that, the root Application reads child Application manifests from \`applications/apps\`. It can create and manage the child Application resources in the \`argocd\` namespace.

### 13.4 Why the directory path matters

A source path in an Argo CD Application points to a repository directory, not an individual YAML file.

A recorded troubleshooting error showed an invalid path resembling:

\`\`\`text
applications/root/root.yml
\`\`\`

The correction was to use a directory path, not a filename. The root Application source should point to:

\`\`\`yaml
path: applications/apps
\`\`\`

The root manifest itself is applied from its local file path; it is not the source directory the root Application should recursively manage.

### 13.5 App of Apps relationship

\`\`\`text
Git repository
    |
    +-- applications/root/root.yaml
    |       |
    |       v
    |    root-app
    |       |
    |       v
    +-- applications/apps/multi-tier.yaml
            |
            v
         multi-tier
            |
            v
       Helm chart resources
            |
            v
      multi-dev namespace
\`\`\`

The captured final status listing showed:

\`\`\`text
NAME        SYNC STATUS   HEALTH STATUS
multi-tier  Synced        Healthy
root-app    Synced        Healthy
\`\`\`

This is evidence that the root and child Application resources were present and reported Synced/Healthy at that point.

---

## 14. ApplicationSet and multi-environment GitOps

The next repository evolution introduced an ApplicationSet to generate Applications for multiple environments.

### 14.1 Intended environments

The List Generator design defined:

| Environment | Namespace | Values file |
|---|---|---|
| dev | \`multi-dev\` | \`../../environments/dev/values.yaml\` |
| qa | \`multi-qa\` | \`../../environments/qa/values.yaml\` |
| stage | \`multi-stage\` | \`../../environments/stage/values.yaml\` |
| prod | \`multi-prod\` | \`../../environments/prod/values.yaml\` |

### 14.2 ApplicationSet manifest design

The following is the ApplicationSet configuration discussed and prepared in the practice. It is included to explain the design, not to claim that the ApplicationSet was successfully created in the final captured state.

\`\`\`yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-tier-appset
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - name: dev
            namespace: multi-dev
            valuesFile: ../../environments/dev/values.yaml
          - name: qa
            namespace: multi-qa
            valuesFile: ../../environments/qa/values.yaml
          - name: stage
            namespace: multi-stage
            valuesFile: ../../environments/stage/values.yaml
          - name: prod
            namespace: multi-prod
            valuesFile: ../../environments/prod/values.yaml

  template:
    metadata:
      name: multi-tier-{{name}}
    spec:
      project: default
      source:
        repoURL: https://github.com/mearvi/gitops-labs.git
        targetRevision: main
        path: helm/multi-tier-chart
        helm:
          valueFiles:
            - "{{valuesFile}}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{namespace}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
\`\`\`

### 14.3 What the List Generator does

The generator contains four elements. Argo CD substitutes the template variables:

- \`{{name}}\`
- \`{{namespace}}\`
- \`{{valuesFile}}\`

The intended generated Applications are:

\`\`\`text
multi-tier-dev
multi-tier-qa
multi-tier-stage
multi-tier-prod
\`\`\`

All four use the same Helm chart but have different destination namespaces and values files.

### 14.4 Important status from the actual lab

The ApplicationSet controller Pod was captured Running:

\`\`\`text
argocd-applicationset-controller-7b8ccc5477-cbvds  1/1 Running
\`\`\`

However, a later verification showed only \`multi-tier\` and \`root-app\` Applications. The expected generated Applications were absent, and the QA and production namespaces had no Deployments.

The conversation correctly diagnosed that the ApplicationSet resource had not been created or discovered by the root Application. Therefore:

- The ApplicationSet design and YAML were prepared.
- The ApplicationSet controller was running.
- The four generated Applications were not verified as created.
- The multi-environment rollout was not completed in the captured state.

The Git Directory Generator was then proposed as the next improvement, but the conversation stopped before it was applied and verified. Do not treat Git Generator discovery as completed.

### 14.5 ApplicationSet troubleshooting commands

\`\`\`bash
kubectl get applicationsets -n argocd
kubectl describe applicationset multi-tier-appset -n argocd
kubectl get applications -n argocd
kubectl get pods -n argocd | grep applicationset
\`\`\`

Also inspect the root Application path:

\`\`\`bash
kubectl get application root-app -n argocd -o yaml \
  | grep -A3 'path:'
\`\`\`

Check that the ApplicationSet YAML is committed inside the exact directory the root Application watches.

---

## 15. Argo CD CLI and kubectl operations

The lab used the UI and \`kubectl\` to create and inspect Applications. These commands are useful for repeatable operations and troubleshooting.

### 15.1 List Applications

\`\`\`bash
kubectl get applications -n argocd
\`\`\`

### 15.2 Inspect one Application

\`\`\`bash
kubectl get application multi-tier -n argocd -o yaml
kubectl describe application multi-tier -n argocd
\`\`\`

### 15.3 Inspect sync policy

\`\`\`bash
kubectl get application multi-tier -n argocd \
  -o jsonpath='{.spec.syncPolicy}'
echo
\`\`\`

### 15.4 Inspect the tracked Git revision

\`\`\`bash
kubectl get application multi-tier -n argocd \
  -o jsonpath='{.status.sync.revision}'
echo
\`\`\`

### 15.5 Inspect the ApplicationSet resources

\`\`\`bash
kubectl get applicationsets -n argocd
kubectl get applications -n argocd
\`\`\`

### 15.6 Argo CD CLI examples

If the CLI is installed and logged in:

\`\`\`bash
argocd app list
argocd app get multi-tier
argocd app diff multi-tier
argocd app sync multi-tier
\`\`\`

These are general CLI examples for inspecting and synchronizing Applications. The core lab evidence is from the UI and Kubernetes Application resources; the CLI commands are included as useful operational alternatives.

---

## 16. Troubleshooting guide

### 16.1 Argo CD Pods are not Ready

\`\`\`bash
kubectl get pods -n argocd -o wide
kubectl describe pod <POD_NAME> -n argocd
kubectl logs <POD_NAME> -n argocd
kubectl get events -n argocd --sort-by=.metadata.creationTimestamp
\`\`\`

Check image pulls, resource availability, PVCs if configured, and component logs.

### 16.2 UI is not reachable

\`\`\`bash
kubectl get svc argocd-server -n argocd -o wide
kubectl describe svc argocd-server -n argocd
kubectl get endpoints argocd-server -n argocd
\`\`\`

Confirm the Service type and current port mapping before using a browser URL. For a local test, use the port-forward command from Section 5.5.

### 16.3 Repository connection fails

Check the repository URL, branch, and access permissions. For a public repository, verify that the URL is correct and reachable. For a private repository, configure credentials using Argo CD's repository credential mechanism; do not put a token into the Application YAML.

### 16.4 Application is OutOfSync

Inspect the Application diff in the UI or run:

\`\`\`bash
argocd app diff multi-tier
kubectl describe application multi-tier -n argocd
\`\`\`

Common reasons include:

- The live cluster was changed manually.
- The chart path or values file differs from the expected repository layout.
- A Helm template renders a different field value.
- Another controller mutates a resource.
- A desired resource is missing or an extra resource exists in the cluster.

Do not assume every OutOfSync state is an application failure. It is a comparison result that needs investigation.

### 16.5 Application is Healthy but OutOfSync

This occurred during the first adoption of the existing Helm-installed resources. The workloads were healthy, but Argo CD had not completed its first sync.

Review the resource diff, then perform the initial manual sync when the desired and live resource changes are understood.

### 16.6 Auto Sync does not remain enabled

Inspect the Application resource:

\`\`\`bash
kubectl get application multi-tier -n argocd -o yaml \
  | grep -A8 syncPolicy
\`\`\`

The actual desired configuration must include:

\`\`\`yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
\`\`\`

If it is absent, update the declarative Application configuration or apply the appropriate patch. Do not rely only on a UI toggle if the Application spec does not reflect the setting.

### 16.7 Root Application shows Unknown or fails to generate children

Check:

- The repository URL and branch.
- The source path points to a directory, not a YAML filename.
- The root Application watches only child Application manifests.
- The root Application is not placed inside the directory it manages.
- Child Application YAML is valid and committed.

Useful commands:

\`\`\`bash
kubectl get application root-app -n argocd -o yaml
kubectl describe application root-app -n argocd
kubectl get applications -n argocd
\`\`\`

### 16.8 ApplicationSet exists in Git but no generated Applications appear

\`\`\`bash
kubectl get applicationsets -n argocd
kubectl describe applicationset multi-tier-appset -n argocd
kubectl get pods -n argocd | grep applicationset
kubectl get application root-app -n argocd -o yaml
\`\`\`

Confirm the root path includes the ApplicationSet manifest. A running ApplicationSet controller alone does not prove that an ApplicationSet resource exists or has generated child Applications.

### 16.9 Unexpected resource deletion

If prune is enabled, inspect the Application's desired resources and the Git diff before allowing deletion. Confirm that the resource is truly intended to be removed. Treat production prune changes as potentially destructive.

---

## 17. Hands-on exercises

Use the dedicated GitOps repository and the lab cluster. Avoid running destructive exercises against a production environment.

### Exercise 1 — Verify the installation

\`\`\`bash
kubectl get namespace argocd
kubectl get pods -n argocd
kubectl get svc -n argocd
kubectl get deployments -n argocd
\`\`\`

Record which components are Running and Ready.

### Exercise 2 — Inspect the Application

\`\`\`bash
kubectl get applications -n argocd
kubectl get application multi-tier -n argocd -o yaml
\`\`\`

Identify the repository, revision, path, values file, destination, and sync policy.

### Exercise 3 — Compare manual and automated sync

In a disposable test Application:

1. Disable automated sync.
2. Commit a harmless manifest or values change.
3. Observe OutOfSync.
4. Review the diff.
5. Trigger a manual sync.
6. Enable automated sync.
7. Commit another harmless change.
8. Observe reconciliation without clicking Sync.

### Exercise 4 — Test self-heal

Only after confirming the Application's Git desired replica count and sync policy:

\`\`\`bash
kubectl scale deployment backend \
  --replicas=5 \
  -n multi-dev

kubectl get deployment backend -n multi-dev -w
\`\`\`

Observe whether the Application reports drift and whether the Deployment returns to the Git-defined replica count.

This exercise was planned in the original lab but does not have a preserved successful output. Run it to obtain direct evidence.

### Exercise 5 — Test prune safely

Use a disposable Service or other non-critical resource:

1. Confirm it is managed by the Application.
2. Remove its template from Git.
3. Review the change.
4. Commit and push.
5. Observe Argo CD's comparison.
6. Confirm the resource is pruned only if that is the intended result.

Do not test prune against PostgreSQL storage or other data-bearing resources.

### Exercise 6 — Verify App of Apps

\`\`\`bash
kubectl apply -f applications/root/root.yaml
kubectl get applications -n argocd
kubectl describe application root-app -n argocd
\`\`\`

Confirm that \`root-app\` manages child Application manifests from \`applications/apps\`.

### Exercise 7 — Diagnose ApplicationSet generation

\`\`\`bash
kubectl get applicationsets -n argocd
kubectl describe applicationset multi-tier-appset -n argocd
kubectl get applications -n argocd
\`\`\`

Check whether the List Generator creates all expected environment Applications. This is the unfinished area from the recorded lab and should be treated as a new exercise, not an already verified outcome.

---

## 18. Interview questions and answers

### Q1. What is GitOps?

GitOps is an operating model in which desired infrastructure and application state is stored declaratively in Git and reconciled into the cluster by an automated controller.

### Q2. What is Argo CD?

Argo CD is a Kubernetes-native continuous delivery tool that observes Git repositories, compares desired and live resources, and synchronizes the cluster with the desired state.

### Q3. What is an Argo CD Application?

An Application is a custom resource that defines a source repository and revision, a path containing manifests or a Helm chart, and a destination cluster and namespace.

### Q4. What is the difference between Healthy and Synced?

Healthy describes the health of the deployed resources. Synced indicates that the live state matches the desired state from the configured source revision. An application can be Healthy but OutOfSync.

### Q5. What does OutOfSync mean?

The desired resources rendered from Git differ from the live resources in Kubernetes. The difference may be caused by a Git change, a manual cluster change, or another controller.

### Q6. What is automated sync?

Automated sync allows Argo CD to apply eligible desired-state changes from Git without an operator manually clicking Sync.

### Q7. What is self-heal?

Self-heal allows Argo CD to correct live changes that differ from the desired state in Git, when automated sync and self-heal are configured.

### Q8. What is prune?

Prune deletes resources from the cluster when they are no longer present in the desired manifests and automated pruning is enabled.

### Q9. What is the difference between self-heal and prune?

Self-heal corrects changes to resources that remain declared in Git. Prune removes live resources that have been removed from the desired source.

### Q10. Does Argo CD run helm upgrade?

For the Helm source in this lab, Argo CD uses Helm to render Kubernetes manifests and manages those resources through its sync mechanism. It does not perform a normal Helm release upgrade lifecycle for the Application.

### Q11. Why was the initial Application Healthy but OutOfSync?

The application resources already existed from a previous Helm installation and were healthy, but Argo CD had not completed its first synchronization against the Git source.

### Q12. What is the App of Apps pattern?

A parent Application manages child Application manifests stored in Git. This allows teams to bootstrap and organize multiple applications declaratively.

### Q13. Why should the root Application be outside the directory it manages?

If the root Application is included in its own source directory, it may attempt to manage itself recursively. Keeping the root manifest separate avoids that structure.

### Q14. What is an ApplicationSet?

An ApplicationSet is a resource that generates multiple Argo CD Applications from generators and a template. The List Generator in this lab was designed to create dev, QA, stage, and prod Applications.

### Q15. Why inspect the Application YAML?

The Kubernetes Application custom resource is the declarative configuration. UI state alone may not prove that the desired sync policy is stored. Inspecting the spec confirms the actual configuration.

---

## 19. Command reference

| Task | Command |
|---|---|
| Create Argo CD namespace | \`kubectl create namespace argocd\` |
| Install Argo CD | \`kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml\` |
| List Argo CD Pods | \`kubectl get pods -n argocd\` |
| List Argo CD Services | \`kubectl get svc -n argocd\` |
| Inspect server Service | \`kubectl get svc argocd-server -n argocd -o wide\` |
| Port-forward UI | \`kubectl port-forward svc/argocd-server -n argocd 8080:443\` |
| List Applications | \`kubectl get applications -n argocd\` |
| Inspect Application | \`kubectl get application multi-tier -n argocd -o yaml\` |
| Describe Application | \`kubectl describe application multi-tier -n argocd\` |
| Inspect sync policy | \`kubectl get application multi-tier -n argocd -o jsonpath='{.spec.syncPolicy}'\` |
| Patch automated sync | \`kubectl patch application multi-tier -n argocd --type merge -p '<PATCH_YAML>'\` |
| List ApplicationSets | \`kubectl get applicationsets -n argocd\` |
| Inspect root Application | \`kubectl get application root-app -n argocd -o yaml\` |
| Check backend Deployment | \`kubectl get deployment backend -n multi-dev\` |
| Simulate drift | \`kubectl scale deployment backend --replicas=5 -n multi-dev\` |
| Inspect Argo CD diff | \`argocd app diff multi-tier\` |
| Sync through CLI | \`argocd app sync multi-tier\` |

---

## 20. Verified work versus unfinished exercises

This distinction is important because the chapter is based on the actual practice conversation, not only on generic Argo CD documentation.

| Area | Evidence from the lab |
|---|---|
| Argo CD installation in \`argocd\` namespace | Verified |
| Argo CD version \`v3.4.5\` | Recorded in the practice conversation |
| Argo CD component Pods Running | Captured |
| Multi-tier Helm chart Application created | Verified |
| Existing Helm-managed app initially Healthy / OutOfSync | Captured |
| First manual sync completed | Captured as Healthy / Synced |
| Automated sync, selfHeal, and prune fields | Patched and verified in Application spec |
| Automatic Git-driven deployment | User confirmed completed; screenshots/terminal output not retained |
| Later Application status Healthy / Synced and auto sync enabled | Captured |
| Successful automated sync operation and revision | Captured in Application status |
| Root Application created | Verified |
| Root and child Application status Synced / Healthy | Captured |
| ApplicationSet List Generator YAML | Prepared/discussed |
| ApplicationSet controller Pod Running | Captured |
| Four generated environment Applications | Not verified; later listing showed they were absent |
| Git Directory Generator | Proposed as next step; not applied or verified |
| Manual drift self-heal scale test | Planned; no captured successful output |
| Prune deletion test | Planned; no captured successful output |
| Manual deletion and automatic recreation test | Planned; no captured successful output |

This is the accurate stopping point for the recorded Argo CD work. The core installation, Application creation, manual sync, automated sync configuration, automatic Git-driven deployment, and App of Apps foundation were completed. The ApplicationSet generation and explicit drift/prune demonstrations remain follow-up work.

---

## 21. Chapter summary and next steps

The lab moved from manual Helm deployments to a GitOps reconciliation model:

1. Argo CD was installed in the \`argocd\` namespace.
2. Its controller and supporting components were verified Running.
3. The \`multi-tier\` Application was connected to \`mearvi/gitops-labs\`.
4. Argo CD rendered the Helm chart and compared desired state with the existing Kubernetes resources.
5. The first manual sync changed the Application to Healthy and Synced.
6. Automated sync, self-heal, and prune were configured and verified in the Application spec.
7. The user confirmed the Git-driven automatic deployment exercise was completed.
8. Application resources were moved toward declarative Git management through an App of Apps structure.
9. The root and child Applications were captured as Synced and Healthy.
10. The List Generator design for dev, QA, stage, and prod was prepared, but the generated Applications were not verified.

### Key takeaways

- Git is the desired-state source of truth.
- Argo CD continuously compares desired state with live state.
- Healthy and Synced represent different status dimensions.
- Automated sync deploys eligible Git changes without a manual Sync click.
- Self-heal corrects live drift when configured.
- Prune removes resources no longer declared in Git.
- Application resources should be managed declaratively.
- App of Apps separates the root Application from its child Applications.
- ApplicationSet can generate Applications from a template, but its generated resources must be verified rather than assumed.

### What comes next?

The next stage of the GitOps learning path is to finish the multi-environment/ApplicationSet work: confirm the List Generator resource is actually created by the root Application, generate the dev/QA/stage/prod Applications, verify each namespace, and then explore the Git Generator. After that, the platform can progress to multiple clusters, image automation, and progressive delivery.

---

## 22. References

### Lab repository

- [GitOps lab repository — mearvi/gitops-labs](https://github.com/mearvi/gitops-labs)

### Official documentation

- [Argo CD documentation](https://argo-cd.readthedocs.io/)
- [Argo CD Getting Started](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [Argo CD Installation](https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/)
- [Argo CD Application specification](https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/)
- [Automated Sync Policy](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/)
- [Sync Options](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/)
- [Helm support](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/)
- [Declarative Setup](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/)
- [App of Apps pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- [ApplicationSet documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

---

**End of Chapter 15 — Argo CD GitOps Delivery**
