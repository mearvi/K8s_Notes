# Chapter 14 — CI/CD Automation with GitHub Actions, Docker Hub, and Kubernetes

## 1. Chapter objectives

This chapter documents the CI/CD pipeline built in the dedicated application repository \`mearvi/k8s-github-actions-lab\`. It connects source control, GitHub Actions, Docker image publishing, and automated Kubernetes deployment.

The final workflow is stored at:

\`\`\`text
.github/workflows/cicd.yml
\`\`\`

The chapter explains the earlier separate workflows, why they were consolidated, how the self-hosted runner solved private-network access, how commit-SHA image tags provide traceability, and how to verify the resulting Deployment rollout.

By the end, you should be able to:

- Explain CI and CD and map them to this lab.
- Read a GitHub Actions workflow and understand triggers, jobs, runners, steps, actions, contexts, and secrets.
- Build and publish a Docker image using a Git commit SHA as its tag.
- Explain why deployment uses a self-hosted runner in this Vagrant environment.
- Update a Kubernetes Deployment image and verify the rollout.
- Inspect rollout history and perform a controlled manual rollback.
- Troubleshoot registry authentication, runner availability, cluster connectivity, image pulls, and failed rollouts.
- Keep credentials and kubeconfig material out of source control.

## 2. Lab environment and application

| Component | Lab value |
|---|---|
| Application repository | \`mearvi/k8s-github-actions-lab\` |
| Workflow file | \`.github/workflows/cicd.yml\` |
| Source branch | \`main\` |
| Container registry | Docker Hub |
| Image repository | \`mearvi/flask-lab\` |
| Kubernetes namespace | \`dev\` |
| Kubernetes Deployment | \`flask-app\` |
| Deployment container name | \`flask-app\` |
| Control-plane node | \`k8master1\` |
| Kubernetes API private address | \`192.168.56.10:6443\` |
| Ingress hostname | \`flask.lab.local\` |
| Ingress address captured in the lab | \`192.168.56.241\` |

The Kubernetes cluster has three nodes:

| Node | Role | Host-only address |
|---|---|---|
| \`k8master1\` | Control plane | \`192.168.56.10\` |
| \`k8worker1\` | Worker | \`192.168.56.20\` |
| \`k8worker2\` | Worker | \`192.168.56.21\` |

The Flask application is exposed through a Kubernetes Service and NGINX Ingress. The recorded browser test used \`http://flask.lab.local\` and returned:

\`\`\`text
Hello from GitHub Actions Lab
\`\`\`

The Windows hosts file contained this lab mapping:

\`\`\`text
192.168.56.241 flask.lab.local
\`\`\`

This is a private learning environment. Confirm the current Ingress address with Kubernetes instead of assuming the historical address is permanent.

## 3. CI/CD concepts

### 3.1 Continuous Integration

Continuous Integration (CI) automates validation and build work whenever code is integrated. A typical sequence is:

1. A developer changes application code.
2. The developer commits and pushes the change.
3. A workflow starts from the repository event.
4. The workflow checks out the triggering commit.
5. Automated checks and tests run.
6. A deployable artifact, such as a container image, is built.

In this lab, the verified CI stages include checkout and Docker image build. The final workflow does **not** show unit tests, linting, static analysis, dependency auditing, or image scanning. Those are future improvements, not completed stages.

### 3.2 Continuous Delivery and Continuous Deployment

Continuous Delivery keeps changes deployable and may require a manual approval before release. Continuous Deployment automatically deploys a change after its required pipeline stages succeed.

This lab automatically deploys the newly published image to the \`dev\` namespace. It is an automated deployment pipeline for the homelab.

### 3.3 Mapping CI/CD to the lab

| Activity | Stage | Purpose |
|---|---|---|
| Push to \`main\` | Trigger | Starts the workflow |
| Checkout | CI | Retrieves the source commit |
| Docker Hub login | Build prerequisite | Authenticates the publisher |
| Docker build | CI | Creates the image |
| Docker push | Publish | Uploads the image |
| Kubernetes image update | CD | Changes the Deployment Pod template |
| Rollout status | CD verification | Waits for the new revision |

A successful Docker push does not prove that Kubernetes deployed the image. Registry publishing and deployment are separate stages.

## 4. Final CI/CD architecture

The final workflow contains two jobs:

- \`build-push\` runs on a GitHub-hosted Ubuntu runner.
- \`deploy\` runs on a self-hosted runner.
- \`needs: build-push\` makes the deployment job wait for successful image publishing.

### 4.1 Mermaid architecture diagram

\`\`\`mermaid
flowchart TD
    A[Developer changes Flask source] --> B[Commit and push to main]
    B --> C[GitHub push event]
    C --> D[GitHub Actions: cicd.yml]
    D --> E[build-push job]
    E --> F[GitHub-hosted Ubuntu runner]
    F --> G[Checkout source]
    G --> H[Docker Hub login via GitHub Secrets]
    H --> I[Build Docker image]
    I --> J[Tag with full Git commit SHA]
    J --> K[Push image to Docker Hub]
    K --> L{Build and push succeeded?}
    L -->|Yes| M[deploy job: needs build-push]
    L -->|No| N[Deployment job does not run]
    M --> O[Self-hosted runner on k8master1]
    O --> P[Use local kubectl and cluster access]
    P --> Q[Verify Kubernetes nodes]
    Q --> R[Set flask-app container image in namespace dev]
    R --> S[Annotate Deployment with commit]
    S --> T[Verify rollout status]
    T --> U[New Pods become available]
    U --> V[Ingress: flask.lab.local]
    V --> W[Flask application response]
\`\`\`

### 4.2 Why two runner types?

The build job does not need direct access to the private Kubernetes network. A GitHub-hosted runner can check out the code, build the image, and push it to Docker Hub.

The deployment job must contact the Kubernetes API server at the private lab address:

\`\`\`text
https://192.168.56.10:6443
\`\`\`

The GitHub-hosted runner could not route to this private Vagrant address. The recorded connection attempt failed with an I/O timeout.

The lab solution was to install a GitHub Actions self-hosted runner on \`k8master1\`. It has access to the local cluster network and the node's Kubernetes client configuration.

| Job | Runner | Reason |
|---|---|---|
| \`build-push\` | \`ubuntu-latest\` | Build and publish without cluster-network access |
| \`deploy\` | \`self-hosted\` | Must reach the private Kubernetes API |

A self-hosted runner on a control-plane node is a convenient homelab arrangement, not a general production recommendation. Production environments should use a dedicated, hardened runner or deployment agent with narrowly scoped Kubernetes permissions.

## 5. GitHub Actions workflow fundamentals

Workflow files are YAML documents stored under:

\`\`\`text
.github/workflows/
\`\`\`

The lab's final workflow is \`.github/workflows/cicd.yml\`.

| Keyword | Meaning |
|---|---|
| \`name\` | Human-readable workflow name |
| \`on\` | Trigger event |
| \`push\` | Run when commits are pushed |
| \`branches\` | Restrict the trigger to selected branches |
| \`jobs\` | Defines jobs |
| \`runs-on\` | Selects runner type or label |
| \`needs\` | Establishes a job dependency |
| \`steps\` | Ordered operations in a job |
| \`uses\` | Invokes a reusable action |
| \`run\` | Executes shell commands |
| \`with\` | Supplies action inputs |
| \`secrets.*\` | Reads a configured Actions secret |
| \`github.sha\` | Full SHA of the commit that triggered the workflow |

### 5.1 Trigger

The workflow runs on pushes to \`main\`:

\`\`\`yaml
on:
  push:
    branches:
      - main
\`\`\`

### 5.2 Jobs and execution order

Jobs may run in parallel unless a dependency is declared. The final workflow contains:

\`\`\`yaml
jobs:
  build-push:
    # build and publish the image

  deploy:
    needs: build-push
    # update Kubernetes
\`\`\`

Without \`needs\`, separate jobs triggered by the same push could start at the same time. Deployment could request an image before the build job had published it.

### 5.3 Actions and shell commands

The workflow uses reusable actions for checkout and Docker Hub authentication:

\`\`\`yaml
- name: Checkout
  uses: actions/checkout@v4
\`\`\`

\`\`\`yaml
- name: Login to Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKER_USERNAME }}
    password: ${{ secrets.DOCKER_PASSWORD }}
\`\`\`

Docker build, push, and Kubernetes deployment are executed with shell commands.

## 6. Earlier workflows and consolidation

The practice started with separate workflow files. They helped isolate and learn each pipeline stage.

| Earlier file | Initial purpose |
|---|---|
| \`docker-build.yml\` | Build local image \`flask-lab:v1\`; no registry push |
| \`docker-push.yml\` | Build and push \`flask-lab:latest\` to Docker Hub |
| \`deploy.yml\` | Run Kubernetes commands; initially restarted the Deployment |

### 6.1 Build-only workflow

The initial build-only example used:

\`\`\`bash
docker build -t flask-lab:v1 .
docker images
\`\`\`

The practice record shows a successful build and an image listing for \`flask-lab:v1\`. The displayed image size was approximately 132 MB.

This established the first CI stage: Git push → GitHub Actions runner → Docker build.

### 6.2 Build-and-push workflow

The next stage authenticated to Docker Hub and published an image. The early version used the mutable \`latest\` tag:

\`\`\`bash
docker build -t <DOCKERHUB_USER>/flask-lab:latest .
docker push <DOCKERHUB_USER>/flask-lab:latest
\`\`\`

The first registry login attempt failed because the configured credential was incorrect. The recorded resolution was to generate a replacement Docker Hub access token, update the GitHub secret, and rerun the workflow. A later run recorded successful login, image build, and push.

### 6.3 Initial deployment workflow

The first deployment workflow ran:

\`\`\`bash
kubectl get nodes
kubectl rollout restart deployment/flask-app -n dev
kubectl rollout status deployment/flask-app -n dev
\`\`\`

This restarted Pods but did not change the image reference in the Deployment. If the Deployment still referenced \`latest\`, a restart did not provide a reliable record of which source revision was deployed.

### 6.4 Race condition between separate workflows

When \`docker-push.yml\` and \`deploy.yml\` both had a push-to-\`main\` trigger, GitHub could start them independently:

1. Deployment starts.
2. It requests the image tagged with the new commit.
3. The build or push is still running.
4. Kubernetes cannot pull an image that is not yet in the registry.

The final design places build/push and deployment in one workflow and declares:

\`\`\`yaml
deploy:
  needs: build-push
\`\`\`

This ensures the deployment job waits for successful image publishing. The live application repository contains the consolidated \`.github/workflows/cicd.yml\` workflow.

## 7. Docker image versioning with the Git commit SHA

### 7.1 Why not use only \`latest\`?

A mutable tag such as \`latest\` can point to different image contents over time. If the Deployment uses only \`latest\`, it becomes harder to determine:

- Which source commit produced the running image?
- Which image should be redeployed to reproduce a release?
- Which version was running when a defect appeared?
- Which version should be selected during rollback?

A commit-specific tag creates a traceable relationship between source code and the container image.

### 7.2 The \`github.sha\` context

GitHub Actions provides the full SHA of the commit that triggered the workflow:

\`\`\`text
${{ github.sha }}
\`\`\`

The final workflow uses that value in the image name:

\`\`\`text
<DOCKERHUB_USER>/flask-lab:<FULL_GIT_COMMIT_SHA>
\`\`\`

A recorded image from the lab was:

\`\`\`text
mearvi/flask-lab:25d1c956759c4b13b056dc818103266a9fbabd4a
\`\`\`

A later successful GitHub Actions run used commit:

\`\`\`text
b55df06c700ded1a6f74ac6c656aa38e62a779ee
\`\`\`

These are historical lab identifiers, not credentials.

### 7.3 Build and push commands

\`\`\`bash
docker build \
  -t <DOCKERHUB_USER>/flask-lab:<FULL_GIT_COMMIT_SHA> .

docker push \
  <DOCKERHUB_USER>/flask-lab:<FULL_GIT_COMMIT_SHA>
\`\`\`

The same image name and tag must be used by the deployment job. If build publishes one tag while deployment requests another, Kubernetes will not deploy the artifact just built.

### 7.4 Verify the deployed image

\`\`\`bash
kubectl get deployment flask-app -n dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
echo
\`\`\`

Expected shape:

\`\`\`text
<DOCKERHUB_USER>/flask-lab:<FULL_GIT_COMMIT_SHA>
\`\`\`

The output should identify the intended commit-specific image, not only \`latest\`.

## 8. GitHub Secrets and credential handling

### 8.1 Secrets used by the workflow

| Secret name | Purpose | Value to configure |
|---|---|---|
| \`DOCKER_USERNAME\` | Docker Hub account name | Your registry username |
| \`DOCKER_PASSWORD\` | Docker Hub authentication | Docker Hub access token |

The workflow references the values without embedding them:

\`\`\`yaml
with:
  username: ${{ secrets.DOCKER_USERNAME }}
  password: ${{ secrets.DOCKER_PASSWORD }}
\`\`\`

### 8.2 Configure repository secrets

In the application repository on GitHub:

1. Open **Settings**.
2. Open **Secrets and variables**.
3. Select **Actions**.
4. Select **New repository secret**.
5. Create \`DOCKER_USERNAME\`.
6. Create \`DOCKER_PASSWORD\` and store the Docker Hub access token as its value.

Never paste a token into a workflow file, a shell command committed to Git, an issue, a pull request, or public documentation.

### 8.3 Why use an access token?

An access token can be rotated or revoked without changing the account password. Grant it only the permissions needed to publish the image. Prefer repository-scoped tokens where the registry supports them.

### 8.4 Secret safety checklist

- Never commit tokens, passwords, private keys, or kubeconfig files.
- Never print secret values with \`echo\`, \`env\`, \`printenv\`, or debug output.
- Do not place credentials directly in Docker build arguments.
- Rotate credentials that may have been exposed.
- Review workflow logs before sharing them.
- Restrict who can change workflow files and who can modify repository secrets.
- Do not assume masking makes it safe to deliberately print a secret.

### 8.5 Kubeconfig: earlier proposal versus final design

An earlier approach discussed creating a kubeconfig file on the control-plane node, Base64-encoding it, and storing it in a GitHub secret named \`KUBECONFIG_DATA\`.

That secret was not required by the final self-hosted runner workflow.

The final deployment job runs on the self-hosted runner on \`k8master1\`, where the cluster network and local \`kubectl\` configuration are available. The live \`cicd.yml\` does not decode a kubeconfig or read \`KUBECONFIG_DATA\`.

Do not add a kubeconfig secret merely because an earlier draft described it. The runner architecture determines how cluster access is supplied.

For production, avoid using a broad administrator kubeconfig on a CI runner. Prefer a dedicated Kubernetes identity with the minimum RBAC permissions required for the target namespace and Deployment.

## 9. Self-hosted runner setup and lifecycle

### 9.1 Why the runner was needed

The Kubernetes API endpoint is on the private Vagrant network. The GitHub-hosted runner could not route to \`192.168.56.10:6443\`; the recorded failure was an I/O timeout.

The lab solution was to install a GitHub Actions runner on \`k8master1\`.

### 9.2 Registration

Register the runner from the repository's GitHub Actions runner setup page. Download the runner package and use the registration command generated by GitHub.

The registration token is short-lived. Do not copy it into documentation or commit it.

From the runner directory, the configuration command is:

\`\`\`bash
./config.sh
\`\`\`

Follow the generated instructions and register the runner to the intended repository. Do not reuse an old registration token.

### 9.3 Run interactively for a first test

\`\`\`bash
./run.sh
\`\`\`

The terminal should show that the runner is listening for jobs. This is useful for a first test because job arrival and execution are visible.

The first runner session exited after the job completed. That is expected for an interactive process that is not managed as a service.

### 9.4 Install the runner as a service

The lab notes record converting the runner to a service so that it remains available after logout or reboot.

From the runner installation directory:

\`\`\`bash
sudo ./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh status
\`\`\`

The recorded verification indicated that the service was active and listening for jobs. Service names and output vary by runner version and Linux distribution; inspect the installed service rather than assuming a fixed unit name.

### 9.5 Runner security

A self-hosted runner executes code from workflow jobs. Anyone able to modify a workflow that runs on the runner may be able to execute commands with the runner account's permissions.

For a production-like setup:

- Use a dedicated runner host instead of the Kubernetes control-plane node.
- Run the runner as an unprivileged account.
- Restrict which repositories and workflows can use it.
- Do not run untrusted pull-request code on a privileged self-hosted runner.
- Keep the runner and operating system patched.
- Restrict network access and Kubernetes RBAC.
- Rotate credentials after suspected exposure.
- Consider ephemeral runners for stronger isolation.

The demonstrated lab runner was installed on \`k8master1\`.

## 10. Final workflow: \`cicd.yml\`

The following is the final workflow structure present in the application repository. Credential values are represented only by GitHub secret references.

**File:** \`.github/workflows/cicd.yml\`

\`\`\`yaml
name: CI-CD Pipeline

on:
  push:
    branches:
      - main

jobs:

  build-push:

    runs-on: ubuntu-latest

    outputs:
      image_tag: ${{ github.sha }}

    steps:

      - name: Checkout
        uses: actions/checkout@v4

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build Docker Image
        run: |
          docker build \
            -t ${{ secrets.DOCKER_USERNAME }}/flask-lab:${{ github.sha }} .

      - name: Push Docker Image
        run: |
          docker push \
            ${{ secrets.DOCKER_USERNAME }}/flask-lab:${{ github.sha }}

  deploy:

    needs: build-push

    runs-on: self-hosted

    steps:

      - name: Checkout
        uses: actions/checkout@v4

      - name: Verify Cluster
        run: |
          kubectl get nodes

      - name: Update Deployment Image
        run: |
          kubectl set image deployment/flask-app \
            flask-app=${{ secrets.DOCKER_USERNAME }}/flask-lab:${{ github.sha }} \
            -n dev

          kubectl annotate deployment/flask-app \
            kubernetes.io/change-cause="Deploy commit ${{ github.sha }}" \
            -n dev --overwrite

      - name: Verify Rollout
        run: |
          kubectl rollout status deployment/flask-app -n dev
\`\`\`

### 10.1 Understand the build-push job

\`\`\`yaml
build-push:
  runs-on: ubuntu-latest
\`\`\`

GitHub provisions a hosted Ubuntu runner for this job.

The checkout action retrieves the source associated with the triggering commit:

\`\`\`yaml
- name: Checkout
  uses: actions/checkout@v4
\`\`\`

The Docker login action authenticates using the two repository secrets:

\`\`\`yaml
- name: Login to Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKER_USERNAME }}
    password: ${{ secrets.DOCKER_PASSWORD }}
\`\`\`

The build step creates the image and tags it with the full commit SHA:

\`\`\`yaml
- name: Build Docker Image
  run: |
    docker build \
      -t ${{ secrets.DOCKER_USERNAME }}/flask-lab:${{ github.sha }} .
\`\`\`

The push step uploads that exact tag:

\`\`\`yaml
- name: Push Docker Image
  run: |
    docker push \
      ${{ secrets.DOCKER_USERNAME }}/flask-lab:${{ github.sha }}
\`\`\`

### 10.2 Understand the deploy job

\`\`\`yaml
deploy:
  needs: build-push
  runs-on: self-hosted
\`\`\`

The deploy job waits for the image publishing job to succeed and then runs on the self-hosted runner.

First, it checks cluster access:

\`\`\`bash
kubectl get nodes
\`\`\`

Then it changes the image in the Deployment:

\`\`\`bash
kubectl set image deployment/flask-app \
  flask-app=<DOCKERHUB_USER>/flask-lab:<FULL_GIT_COMMIT_SHA> \
  -n dev
\`\`\`

The first \`flask-app\` identifies the Deployment. The second \`flask-app\` is the container name inside the Pod template. They happen to match in this lab, but are separate Kubernetes fields.

The container name was checked during the practice:

\`\`\`bash
kubectl get deployment flask-app -n dev \
  -o jsonpath='{.spec.template.spec.containers[0].name}'
echo
\`\`\`

The observed container name was:

\`\`\`text
flask-app
\`\`\`

The workflow annotates the Deployment with the commit being deployed:

\`\`\`bash
kubectl annotate deployment/flask-app \
  kubernetes.io/change-cause="Deploy commit <FULL_GIT_COMMIT_SHA>" \
  -n dev --overwrite
\`\`\`

Finally, it waits for the rollout:

\`\`\`bash
kubectl rollout status deployment/flask-app -n dev
\`\`\`

A successful \`kubectl set image\` means the Pod template was updated. The rollout-status step separately checks whether the new Pods become available.

### 10.3 Note about the declared job output

The live workflow declares an \`image_tag\` job output equal to the triggering commit SHA. The deploy job currently uses the commit context directly instead of reading \`needs.build-push.outputs.image_tag\`. Both identify the same triggering commit in this workflow.

The output declaration is therefore not required by the current implementation. It is retained here because it exists in the live workflow. A later cleanup could remove the unused output or explicitly consume it from the dependent job. Test any workflow change before replacing the verified version.

## 11. End-to-end execution sequence

1. **Change the application.** A later practice example changed the Flask homepage response from \`Hello from GitHub Actions Lab\` to a versioned message such as \`Hello from GitHub Actions Lab - Version 2\`.
2. **Commit and push.** From the application repository:

   \`\`\`bash
   git status
   git add .
   git commit -m "Update homepage"
   git push origin main
   \`\`\`

3. **Checkout.** The build job retrieves the triggering source commit.
4. **Authenticate.** The Docker login action uses \`DOCKER_USERNAME\` and \`DOCKER_PASSWORD\` from GitHub Secrets.
5. **Build.** Docker creates the image tagged with the full Git commit SHA.
6. **Publish.** Docker pushes the same image name and tag to Docker Hub.
7. **Wait.** \`needs: build-push\` prevents deployment from starting before image publishing succeeds.
8. **Verify cluster.** The self-hosted runner runs \`kubectl get nodes\`.
9. **Update image.** The runner changes the \`flask-app\` container image in Deployment \`flask-app\`, namespace \`dev\`.
10. **Verify rollout.** The workflow waits for \`kubectl rollout status deployment/flask-app -n dev\`.
11. **Verify application.** From a host that can resolve the lab hostname and reach Ingress:

   \`\`\`bash
   curl -i http://flask.lab.local
   \`\`\`

The original lab captured the greeting \`Hello from GitHub Actions Lab\`. A subsequent application change should be checked against the new response expected from that commit.

## 12. Verification commands

Run these commands from a machine with a working Kubernetes context.

### 12.1 Cluster nodes

\`\`\`bash
kubectl get nodes -o wide
\`\`\`

### 12.2 Deployment details

\`\`\`bash
kubectl get deployment flask-app -n dev
kubectl describe deployment flask-app -n dev
\`\`\`

### 12.3 Deployed image

\`\`\`bash
kubectl get deployment flask-app -n dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
echo
\`\`\`

### 12.4 Pod status and placement

\`\`\`bash
kubectl get pods -n dev -o wide
\`\`\`

### 12.5 Watch a rollout

\`\`\`bash
kubectl get pods -n dev -w
\`\`\`

During a successful rolling update, old Pods are replaced by new Pods according to the Deployment rollout strategy and readiness conditions.

### 12.6 Rollout status and history

\`\`\`bash
kubectl rollout status deployment/flask-app -n dev
kubectl rollout history deployment/flask-app -n dev
\`\`\`

The lab recorded multiple Deployment revisions, including revisions 1–3. The change-cause annotation improves the information associated with later revisions.

### 12.7 Inspect the change-cause annotation

\`\`\`bash
kubectl get deployment flask-app -n dev \
  -o jsonpath='{.metadata.annotations.kubernetes\.io/change-cause}'
echo
\`\`\`

### 12.8 Service and Ingress

\`\`\`bash
kubectl get service -n dev
kubectl get ingress -n dev
kubectl describe ingress flask-ingress -n dev
\`\`\`

### 12.9 HTTP test

\`\`\`bash
curl -i http://flask.lab.local
\`\`\`

If the host cannot resolve the name, check the Windows hosts file and confirm the current Ingress address with \`kubectl get ingress -n dev\`.

## 13. Verified results from the practice session

This section distinguishes captured execution evidence from recommendations and future work.

### 13.1 Initial build-only workflow

The practice record shows:

- The GitHub Actions Docker Build workflow completed successfully.
- The Docker build step completed successfully.
- The image listing showed \`flask-lab:v1\`.
- The displayed image size was approximately 132 MB.

This verified the initial checkout and image-build stage.

### 13.2 Docker Hub publishing

The practice record shows successful Docker Hub login, image build, and image push after correcting the registry credential secret.

The initial mutable \`latest\` tag was later replaced by commit-SHA image tags.

### 13.3 Self-hosted runner and Kubernetes

The captured runner output included:

\`\`\`text
Running job: deploy
Job deploy completed with result: Succeeded
\`\`\`

The captured Kubernetes Pod output showed two Flask Pods in the Running state:

\`\`\`text
flask-app-6c7464679b-bqq5r    Running
flask-app-6c7464679b-nkwrd    Running
\`\`\`

The browser test for \`http://flask.lab.local\` returned:

\`\`\`text
Hello from GitHub Actions Lab
\`\`\`

This confirms a working application delivery path through the Service and Ingress.

### 13.4 Versioned image and successful workflow run

The practice record captured this commit-specific image:

\`\`\`text
mearvi/flask-lab:25d1c956759c4b13b056dc818103266a9fbabd4a
\`\`\`

The live GitHub repository also records a successful run:

| Field | Recorded value |
|---|---|
| Workflow | \`CI-CD Pipeline\` |
| Workflow file | \`.github/workflows/cicd.yml\` |
| Run number | \`2\` |
| Event | \`push\` |
| Branch | \`main\` |
| Commit | \`b55df06c700ded1a6f74ac6c656aa38e62a779ee\` |
| Commit message | \`Update homepage\` |
| Conclusion | \`success\` |
| Build job | \`build-push\` — success |
| Deployment job | \`deploy\` — success |

The run's job summary records successful completion of the following steps.

**Build job**
- Checkout
- Login to Docker Hub
- Build Docker Image
- Push Docker Image

**Deployment job**
- Checkout
- Verify Cluster
- Update Deployment Image
- Verify Rollout

This is direct workflow-run evidence that both jobs and their listed steps completed successfully for the recorded commit.

### 13.5 Evidence boundaries

| Item | Evidence status |
|---|---|
| Workflow exists in the application repository | Verified from live repository |
| Trigger is push to \`main\` | Verified in workflow YAML |
| Docker Hub login action configured | Verified in workflow YAML |
| Image built and pushed using commit SHA | Verified in YAML and successful run steps |
| Deployment waits for build-push | Verified in workflow YAML |
| Deployment uses self-hosted runner | Verified in workflow YAML |
| Kubernetes image update command | Verified in workflow YAML |
| Rollout-status step | Verified in YAML and successful run |
| Successful GitHub Actions run exists | Verified; run #2 completed successfully |
| Two Flask Pods Running in a captured snapshot | Verified in practice record |
| Ingress returned the recorded greeting | Verified in practice record |
| Unit tests or quality scans | Not shown in verified workflow |
| Container vulnerability scanning | Not shown in verified workflow |
| Production approval gates | Not shown in verified workflow |
| Highly available deployment infrastructure | Not part of this local lab |

The pipeline is a real end-to-end learning workflow, but it is not a complete production software-supply-chain pipeline. Preserve these distinctions when presenting the project in a portfolio or interview.

## 14. Rolling updates and rollback

### 14.1 Why changing the image creates a rollout

A Kubernetes Deployment manages ReplicaSets and Pods. When the Pod template changes—for example, when the container image changes—the Deployment creates a new ReplicaSet and gradually replaces old Pods according to its rollout strategy.

The workflow changes the Pod template with:

\`\`\`bash
kubectl set image deployment/flask-app \
  flask-app=<DOCKERHUB_USER>/flask-lab:<FULL_GIT_COMMIT_SHA> \
  -n dev
\`\`\`

### 14.2 Observe the new Pods and ReplicaSets

\`\`\`bash
kubectl get pods -n dev -w
kubectl get rs -n dev
\`\`\`

The old ReplicaSet may remain with zero desired replicas so that Deployment history and rollback remain possible.

### 14.3 Inspect rollout history

\`\`\`bash
kubectl rollout history deployment/flask-app -n dev
\`\`\`

A Deployment revision is created when its Pod template changes. Updating the image reference therefore creates a new revision.

### 14.4 Manual rollback

If a release is unhealthy, a manual rollback can be initiated:

\`\`\`bash
kubectl rollout undo deployment/flask-app -n dev
\`\`\`

Then verify:

\`\`\`bash
kubectl rollout status deployment/flask-app -n dev
kubectl rollout history deployment/flask-app -n dev
kubectl get deployment flask-app -n dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
echo
\`\`\`

Rollback changes the Deployment Pod template to a previous revision. It does not delete the image from Docker Hub.

The workflow documented here performs deployment and rollout verification. Rollback is a manual recovery operation; the evidence does not show an automated rollback stage inside \`cicd.yml\`.

## 15. Troubleshooting guide

### 15.1 Docker Hub login fails

**Symptom:** The login step reports an authentication error or Docker reports that a password is required.

Check:

1. Secret names match the workflow exactly.
2. Docker Hub username is correct.
3. Token is valid and has not been revoked.
4. Token has permission to push to the image repository.
5. Secret was updated after token replacement.

Do not paste the token into the workflow or terminal.

### 15.2 Docker image build fails

Check the Dockerfile, build context, dependency installation, and application files.

\`\`\`bash
docker version
docker build -t local-flask-test:debug .
\`\`\`

These are local diagnostic examples, not additional steps claimed to have run in the verified workflow.

### 15.3 Image push fails

Check that Docker login succeeded, the repository exists, the tag is valid, the account can push, and build/push use the same image name and tag. A successful local build does not prove that the registry push succeeded.

### 15.4 Deployment job remains queued

Likely cause: no eligible self-hosted runner is online or available.

From the runner installation directory:

\`\`\`bash
sudo ./svc.sh status
\`\`\`

Check runner registration and labels. The workflow's \`runs-on: self-hosted\` must match an available runner.

### 15.5 Kubernetes API connection times out

The GitHub-hosted runner is outside the Vagrant host-only network and cannot automatically route to the private cluster address.

The lab resolution was to run deployment on the self-hosted runner on \`k8master1\`.

On the runner host:

\`\`\`bash
ip -br address
kubectl config current-context
kubectl cluster-info
kubectl get nodes
\`\`\`

These checks help distinguish network reachability from kubeconfig or API-server issues.

### 15.6 ImagePullBackOff or ErrImagePull

Inspect the Pod and events:

\`\`\`bash
kubectl describe pod <POD_NAME> -n dev
kubectl get events -n dev --sort-by=.metadata.creationTimestamp
\`\`\`

Check that the image repository exists, the exact commit tag was pushed, the Deployment uses the same tag, the registry is public or a suitable imagePullSecret exists, and nodes can reach Docker Hub.

### 15.7 Deployment update succeeds but rollout fails

\`\`\`bash
kubectl rollout status deployment/flask-app -n dev
kubectl describe deployment flask-app -n dev
kubectl get pods -n dev -o wide
kubectl get events -n dev --sort-by=.metadata.creationTimestamp
\`\`\`

Inspect the failing Pod:

\`\`\`bash
kubectl describe pod <POD_NAME> -n dev
kubectl logs <POD_NAME> -n dev
\`\`\`

Possible causes include image pull failures, application startup errors, readiness-probe failures, resource pressure, or incorrect container configuration.

### 15.8 Application is not reachable through Ingress

Check each layer:

\`\`\`bash
kubectl get pods -n dev
kubectl get service -n dev
kubectl get endpoints -n dev
kubectl get ingress -n dev
kubectl describe ingress flask-ingress -n dev
\`\`\`

Then confirm the hostname resolves to the current Ingress address and test:

\`\`\`bash
curl -i http://flask.lab.local
\`\`\`

### 15.9 Workflow deploys an old image

Inspect the image reference:

\`\`\`bash
kubectl get deployment flask-app -n dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
echo
\`\`\`

Compare it with the triggering commit SHA and Docker Hub tag. Confirm that build and deploy use the same commit context.

### 15.10 Separate workflows run out of order

If independent push-triggered workflows are reintroduced, they can run concurrently. Use one workflow and an explicit dependency:

\`\`\`yaml
deploy:
  needs: build-push
\`\`\`

This dependency is present in the final lab workflow.

## 16. Security and production-readiness

The homelab demonstrates the core delivery flow. A production pipeline needs additional controls.

### 16.1 Minimize GitHub token permissions

The lab workflow does not declare a top-level \`permissions\` block. A hardening improvement is to grant only the permissions required, for example:

\`\`\`yaml
permissions:
  contents: read
\`\`\`

Review the permissions required by every action before adding them. Do not grant write access by default.

### 16.2 Protect deployment environments

For production:

- Use GitHub Environments and approval rules where appropriate.
- Restrict deployment to protected branches or approved tags.
- Separate development and production credentials.
- Isolate production runners from untrusted pull-request jobs.
- Restrict Kubernetes RBAC to the resources and namespace the pipeline manages.
- Record deployment identity and commit metadata.

### 16.3 Pin third-party actions

The lab uses version tags such as \`actions/checkout@v4\` and \`docker/login-action@v3\`. For a hardened supply-chain workflow, consider pinning actions to reviewed full commit SHAs and updating them through a controlled process.

### 16.4 Add testing and scanning

A future pipeline can add:

- Unit tests.
- Static analysis and linting.
- Dependency vulnerability checks.
- Container image vulnerability scanning.
- Software bill of materials (SBOM) generation.
- Image signing and provenance verification.

These controls were not present in the verified lab workflow. Add and test them as improvements rather than claiming they were completed.

### 16.5 Prefer immutable deployment references

The full Git commit SHA tag is more traceable than \`latest\`. For stronger supply-chain guarantees, a production deployment can reference an image digest:

\`\`\`text
registry.example.com/team/application@sha256:<IMAGE_DIGEST>
\`\`\`

A digest identifies image content independently of mutable tags. Digest-based promotion was not demonstrated in this lab.

### 16.6 Protect the Kubernetes API

Do not expose the private Kubernetes API publicly just to make a hosted runner reach it. The self-hosted runner solved the homelab network limitation without publishing the API endpoint.

In production, use a controlled network path, a dedicated deployment runner, or a managed identity-based deployment integration. Scope credentials to the minimum required operations.

## 17. Hands-on exercises

Complete these exercises in the dedicated application repository. Do not experiment directly against a production cluster.

### Exercise 1 — Inspect the workflow

1. Open \`.github/workflows/cicd.yml\`.
2. Identify the push trigger.
3. Identify both jobs and their runners.
4. Explain why \`deploy\` has \`needs: build-push\`.
5. Identify every GitHub secret reference.

\`\`\`bash
cat .github/workflows/cicd.yml
git status
\`\`\`

### Exercise 2 — Trace a commit to an image

1. Make a harmless application change.
2. Commit and push to \`main\`.
3. Open the GitHub Actions run.
4. Identify the commit SHA.
5. Confirm the Docker image tag uses that SHA.
6. Confirm the push step succeeds.

Expected image naming pattern:

\`\`\`text
<DOCKERHUB_USER>/flask-lab:<FULL_GIT_COMMIT_SHA>
\`\`\`

### Exercise 3 — Observe the Kubernetes rollout

Before pushing a new change, open a terminal with cluster access:

\`\`\`bash
kubectl get pods -n dev -w
\`\`\`

Push a new application commit. After the workflow completes:

\`\`\`bash
kubectl rollout status deployment/flask-app -n dev
kubectl get deployment flask-app -n dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
echo
\`\`\`

Compare the image tag with the commit that triggered the run.

### Exercise 4 — Inspect revision history

\`\`\`bash
kubectl rollout history deployment/flask-app -n dev
\`\`\`

Inspect the change-cause annotation. Explain how the commit SHA helps correlate a Kubernetes rollout with a source revision.

### Exercise 5 — Controlled manual rollback

Only after recording the current image and revision:

\`\`\`bash
kubectl rollout history deployment/flask-app -n dev
kubectl rollout undo deployment/flask-app -n dev
kubectl rollout status deployment/flask-app -n dev
\`\`\`

Verify the restored image reference and application response.

### Exercise 6 — Explain the runner architecture

1. Why did the GitHub-hosted runner time out when accessing the private Kubernetes API?
2. Why did the self-hosted runner work?
3. What security risks arise when a runner executes on the control-plane node?
4. What would you change for a production deployment runner?

### Exercise 7 — Propose a quality gate

Design a new job that runs unit tests before image publishing. Do not merge it into the active workflow until it passes independently.

The intended dependency is:

\`\`\`text
test → build-push → deploy
\`\`\`

Document the test command and expected failure behavior.

## 18. Interview questions and answers

### Q1. What is CI/CD?

CI/CD automates integrating code, validating changes, producing deployable artifacts, and delivering them to an environment.

### Q2. What is the role of GitHub Actions?

GitHub Actions runs event-triggered workflows. A workflow contains jobs, and jobs contain ordered steps executed on selected runners.

### Q3. Why did the lab use two jobs?

The build job runs on a GitHub-hosted runner, while deployment requires access to a private Kubernetes cluster. The deployment job therefore runs on a self-hosted runner.

### Q4. Why use \`needs: build-push\`?

It ensures deployment waits until the image has been built and pushed successfully. It prevents a race between publishing and deployment.

### Q5. Why tag an image with the Git commit SHA?

It creates a traceable mapping between the source commit and container image and avoids relying exclusively on a mutable \`latest\` tag.

### Q6. What is the purpose of GitHub Secrets?

Secrets store sensitive values outside the workflow source. The workflow references a secret by name without embedding its value.

### Q7. Why did the GitHub-hosted runner fail to contact Kubernetes?

The lab API endpoint is on a private Vagrant host-only network. A hosted runner cannot route to that private address without an explicitly configured network path.

### Q8. What is a self-hosted runner?

It is a runner installed and managed on infrastructure controlled by the repository owner or organization. It can access permitted internal resources, but must be secured because it executes workflow code.

### Q9. What does \`kubectl set image\` do?

It changes the container image in a workload's Pod template. For a Deployment, this triggers a new rollout.

### Q10. Why verify the rollout?

A successful API update proves Kubernetes accepted the desired Pod-template change. Rollout verification checks whether the new Pods become available.

### Q11. Difference between restarting a Deployment and updating its image?

A restart recreates Pods using the image already specified in the Pod template. Updating the image changes the Pod template to a specific image reference and creates a new Deployment revision.

### Q12. How can you roll back a failed Deployment?

Use \`kubectl rollout undo deployment/<name> -n <namespace>\`, then verify rollout status, image reference, and application behavior.

### Q13. Does a successful Docker push prove that the application is deployed?

No. The deployment job must update Kubernetes, the new Pods must start and become ready, and application reachability should be checked separately.

### Q14. What CI stages are missing from the demonstrated pipeline?

The verified workflow does not show unit tests, static analysis, dependency scanning, image scanning, SBOM generation, or image signing. These are possible improvements.

### Q15. What is the risk of a privileged self-hosted runner?

Workflow code can execute on the runner host. Untrusted code may gain access to local files, credentials, or internal network resources available to the runner account.

## 19. Command reference

| Task | Command |
|---|---|
| Inspect workflow | \`cat .github/workflows/cicd.yml\` |
| Check Git state | \`git status\` |
| Commit application change | \`git add . && git commit -m "Update application"\` |
| Push to main | \`git push origin main\` |
| Check runner service | \`sudo ./svc.sh status\` |
| Check cluster nodes | \`kubectl get nodes\` |
| Check Deployment | \`kubectl get deployment flask-app -n dev\` |
| Check Pods | \`kubectl get pods -n dev -o wide\` |
| Watch Pods | \`kubectl get pods -n dev -w\` |
| Check rollout | \`kubectl rollout status deployment/flask-app -n dev\` |
| View rollout history | \`kubectl rollout history deployment/flask-app -n dev\` |
| Roll back | \`kubectl rollout undo deployment/flask-app -n dev\` |
| Check Service | \`kubectl get svc -n dev\` |
| Check Ingress | \`kubectl get ingress -n dev\` |
| Test application | \`curl -i http://flask.lab.local\` |

## 20. Chapter summary

The lab progressed from a manual Docker build to an automated application delivery pipeline:

1. A push to \`main\` triggers GitHub Actions.
2. A GitHub-hosted Ubuntu runner checks out the source.
3. The workflow authenticates to Docker Hub using GitHub Secrets.
4. The workflow builds the Flask image and tags it with the full commit SHA.
5. The workflow pushes the image to Docker Hub.
6. After \`build-push\` succeeds, the dependent deployment job starts on the self-hosted runner.
7. The runner verifies cluster access and updates Deployment \`flask-app\` in namespace \`dev\`.
8. The workflow annotates the Deployment with the commit and waits for rollout completion.
9. The application is verified through its Ingress hostname.

The live workflow run for commit \`b55df06c700ded1a6f74ac6c656aa38e62a779ee\` completed successfully, with both jobs and their recorded steps successful.

The key lessons are:

- Build and deploy should have an explicit dependency.
- Use commit-specific image identifiers rather than relying only on \`latest\`.
- A hosted runner cannot automatically access a private homelab network.
- A self-hosted runner needs isolation and least-privilege access.
- Registry credentials belong in GitHub Secrets, not source control.
- A successful image push and successful Kubernetes rollout are different checks.
- Production pipelines need additional testing, scanning, approvals, and supply-chain controls.

### What comes next?

Chapter 15 introduces **Argo CD GitOps Delivery**. It builds on the Git, container, Kubernetes, and deployment concepts in this chapter, but changes the deployment control model: rather than a CI runner directly applying the desired change, Argo CD continuously reconciles Kubernetes with the desired state stored in Git.

## 21. References

### Lab repositories

- Application and workflow repository: [mearvi/k8s-github-actions-lab](https://github.com/mearvi/k8s-github-actions-lab)
- Final workflow: [\`.github/workflows/cicd.yml\`](https://github.com/mearvi/k8s-github-actions-lab/blob/main/.github/workflows/cicd.yml)
- Recorded successful workflow run: [CI-CD Pipeline — run #2](https://github.com/mearvi/k8s-github-actions-lab/actions/runs/27340531791)

### Official documentation

- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [Understanding GitHub Actions](https://docs.github.com/en/actions/about-github-actions/understanding-github-actions)
- [GitHub Actions workflow syntax](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions)
- [Using secrets in GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)
- [Self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners)
- [Docker login action](https://github.com/docker/login-action)
- [Docker build documentation](https://docs.docker.com/build/)
- [Docker Hub documentation](https://docs.docker.com/docker-hub/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [kubectl set image](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_set/kubectl_set_image/)
- [Deployment rolling updates](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment)
- [Kubernetes ServiceAccounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

---

**End of Chapter 14 — CI/CD Automation**
