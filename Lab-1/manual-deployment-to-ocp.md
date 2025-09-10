# Lab 1: Manual Deployment to OpenShift

## Objective

Manually containerize and deploy a sample application (UI + API) to an OpenShift cluster using your own Quay.io registry.

**Repos used:**

* UI: [https://github.com/apj-aih/pipelines-vote-ui](https://github.com/apj-aih/pipelines-vote-ui)
* API: [https://github.com/apj-aih/pipelines-vote-api](https://github.com/apj-aih/pipelines-vote-api)

At the end of the lab, you will have:

* Two contianer images (UI & API) in **Quay.io** under your organization
* A working application deployed to your **OpenShift** project/namespace

<br/>

## Prerequisites (should be done before the lab)

1. **Red Hat Sandbox**: Create an account at `sandbox.redhat.com` and launch an OpenShift cluster.
2. **Quay.io**: Log in using the same Red Hat credentials.
3. **GitHub**: Make sure you can log in to your account.

> You’ll receive a **VM** to run commands from. Keep your openshift cluster console open in a browser tab.

<br/>

# Part 1 – Prepare the Environment

## Folder & Image Conventions

We’ll use these conventions throughout the lab. Open the terminal & edit the file **bashrc** using below command:

```bash
vim ~/.bashrc
```
Edit the below stated values:
```bash
# === Edit these values ===
export QUAY_USERNAME="<your-quay-username>"   # e.g., john (copied from quay.io)
export IMAGE_TAG="v1"                         # e.g., v1, v1.0.0, or today’s date
export OCP_PROJECT="<your-ocp-project-name>"  # e.g., john-dev (copied from sandbox environment)

# Derived convenience vars (do not edit)
export UI_IMAGE="quay.io/${QUAY_USERNAME}/vote-ui:${IMAGE_TAG}"
export API_IMAGE="quay.io/${QUAY_USERNAME}/vote-api:${IMAGE_TAG}"
```
Save the file
```bash
Esc :wq

Enter
```

Reload the terminal
```bash
source ~/.bashrc
```
<br/>


## Environment Check (run on the provided VM)

Verify the needed CLIs are available.

```bash
# Container engine
podman --version

# OpenShift CLI
oc version --client

# Git
git --version
```

If any command is missing, ask a facilitator.

![Screenshot: Environment Check](./images/environment-check.png)

<br/>

## Set Quay.io CLI password

### 1) Go to Quay.io (container registry) & open account settings

![Screenshot: Quay IO](./images/quay-io.png)

### 2) In CLI Password - click **Set password**

![Screenshot: Quay IO Set Password](./images/quay-set-password.png)

Keep this password & use it when CLI password is asked.

Download the Kubernetes secret yaml and keep it as well.

![Screenshot: Quay IO K8s Secret](./images/quay-k8s-secret.png)

<br/>

## Log In to Required Services

### 1) Quay.io (container registry)

```bash
podman login quay.io
# Enter your Quay.io username & CLI password when prompted
```
Make sure "Login Succeeded!"

![Screenshot: Quay CLI](./images/quay-cli-login.png)

### 2) OpenShift cluster (CLI)

Use the token from the OpenShift Web Console (top-right > Copy Login Command).

![Screenshot: Quay CLI Credentials](./images/ocp-cli-credentials.png)

Click **Display Token** and copy command under **Log in with this token**

![Screenshot: Quay CLI Credentials Copy](./images/ocp-cli-credentials-1.png)

```bash
# paste the command in the terminal:
oc login --token=<paste-your-token> --server=https://api.<cluster-domain>:6443
```

Confirm your user and cluster:

```bash
oc whoami
oc project
```

![Screenshot: OCP CLI Verify](./images/ocp-cli-verify.png)

<br/>

## What You’ll Build (High-Level)

* **vote-ui**: Frontend service (build image and deploy)
* **vote-api**: Backend service (build image and deploy)
* Expose the **UI** via a **Route** so you can access it from a browser

<br/>

# Part 2 – Clone the Repositories

Now that you are logged in and your environment is ready, clone the two repositories to your VM.

```bash
# Navigate to a working directory
mkdir -p ~/labs && cd ~/labs

# Clone the UI repo
git clone https://github.com/apj-aih/pipelines-vote-ui.git

# Clone the API repo
git clone https://github.com/apj-aih/pipelines-vote-api.git

# Verify
tree -L 1
```

Expected structure:

```
./labs
├── pipelines-vote-ui
└── pipelines-vote-api
```

![Screenshot: git clone](./images/git-clone.png)

<br/>

# Part 3 – Build Docker Images

Next, we will build container images for both the UI and API components and tag them for Quay.

### 1) Build the UI image

```bash
cd ~/labs/pipelines-vote-ui

git checkout manual

echo "$UI_IMAGE"

podman build -t "$UI_IMAGE" .
```

After the build completes, confirm the image exists:

```bash
podman images | grep vote-ui
```

### 2) Build the API image

```bash
cd ~/labs/pipelines-vote-api

git checkout manual

echo "$API_IMAGE"

podman build -t "$API_IMAGE" .
```

Check the image:

```bash
podman images | grep vote-api
```

### 3) Verify both images

```bash
podman images | grep "$QUAY_USERNAME"
```

Expected result: two images tagged with your Quay.io namespace.

![Screenshot: Image Verify](./images/image_verify.png)


<br/>

# Part 4 – Push Images to Quay.io

Now that the images are built locally, push them to your Quay.io registry.

### 1) Create new repositories for both images. Go to quay.io & click "Create New Repository"

![Screenshot: Quay New Registry](./images/quay-new-repo.png)

### 2) Create public repo for vote-ui & vote-api

![Screenshot: Quay New Vote UI Registry](./images/quay-new-repo-vote-ui.png) 

![Screenshot: Quay New Vite API Registry](./images/quay-new-repo-vote-api.png)

### 3) Push the UI image

```bash
podman push "$UI_IMAGE"
```

### 4) Push the API image

```bash
podman push "$API_IMAGE"
```

### 5) Verify in Quay.io Web Console

1. Log in to [https://quay.io](https://quay.io) with your account.
2. Navigate to your namespace.
3. Ensure you see both repositories: `vote-ui` and `vote-api` with quota consumed.

![Screenshot: Push Verify](./images/push_verify.png)

4. You can see the tag uploaded as shown below:

![Screenshot: Quay Vote API Tags](./images/quay-vote-api-v1.png)

<br/>

# Part 5 – Create Deployment in OpenShift Cluster

Now that your images are in Quay.io, let’s deploy them to OpenShift.

### 1) Deploy the API

```bash
oc new-app "$API_IMAGE" --name=vote-api
```

Expose the service so the UI can reach it:

```bash
oc expose deploy/vote-api --port 9000
```

Check status:

```bash
oc get pods

oc get svc vote-api
```

### 2) Deploy the UI

```bash
oc new-app "$UI_IMAGE" --name=vote-ui \
  -e VOTING_API_SERVICE_HOST=vote-api \
  -e VOTING_API_SERVICE_PORT=9000
```

Expose the UI as a public route:

```bash
oc expose svc/vote-ui
```

Check route:

```bash
oc get route vote-ui
```

Copy the `HOST/PORT` and open it in your browser.

![Screenshot: Voting APP UI](./images/voting-ui.png)

### 3) Verify End-to-End

1. Open the route URL in your browser.
2. Ensure the UI loads and communicates with the API.

![Screenshot: Vote for Dog UI](./images/voting-ui-dog.png)

<br/>

# Clean Up

Run the below commands to remove all the resources created in the openshift cluster.

```bash
# Delete the UI app
oc delete all -l app=vote-ui

# Delete the API app
oc delete all -l app=vote-api
```

**Ignore the Warning & Permission issues while deleting**

<br/>

# ✅ Lab 1 Completed

You have successfully:

* Built and pushed UI & API images to Quay.io
* Deployed both components manually in OpenShift
* Exposed the UI route and accessed the running application

Next, you will automate this process using OpenShift Pipelines in **Lab 2**.
