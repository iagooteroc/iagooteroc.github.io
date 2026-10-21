Sources:  
https://docs.docker.com/reference/dockerfile/  
https://helm.sh/docs  
https://argo-cd.readthedocs.io/  
https://github.com/argoproj/argo-cd/issues/26265#issuecomment-3850682322  
...  
https://learn.microsoft.com/en-us/azure/virtual-machines/windows/quick-create-terraform  
https://learn.microsoft.com/en-us/azure/developer/terraform/

-----

## Overview

In this project, I have documented all the necessary steps to build a basic DevOps workflow from scratch, including:
- Git as the source of truth and container registry
- Infrastructure as Code (IaC)
- Container orchestration
- Continuous Integration and Continuous Delivery (CI/CD)
- Monitoring and observability
- Public cloud deployment

## **Part 1** -- Git as the source of truth and container registry
---
Following GitOps practices, we will use Git as the single source of truth for our declarative infrastructure in addition to its version control capabilities. On top of that, we will take advantage of GitHub's Container Registry as an easy way of making our containerized application available to the cloud.

### 1.1. Git repository

We will be using **Git** as the source of truth, so start by creating a repository. I chose GitHub and used its Container Registry, but you can host your own with **Harbor** if you prefer, or use any other platform.

**Requirements:**
- `git`

**Installation on Arch Linux:**
```bash
# Install the package
sudo pacman -S git
```

Either create the repository first through the web interface and `git clone` it, or start with an empty folder and do `git init` inside, adding the remote repository later with `git remote add origin <URL>`.

### 1.2. Minimal containerized app

We need a containerized app to deploy in the project, so build a minimal website and pack it in a Dockerfile.

**Requirements:**
- `docker`

**Installation on Arch Linux:**
```bash
# Install the package
sudo pacman -S docker
# Start the Docker daemon service
sudo systemctl start docker
```

Create an `index.html` file with an example site, like a Kanban board. You can copy it from my repository.  
Add a `styles.css` file for some minimal styling and `app.js` with the javascript to make it work.

Then, create the `Dockerfile` to contain the app:
```docker
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html
COPY styles.css /usr/share/nginx/html/styles.css
COPY app.js /usr/share/nginx/html/app.js

EXPOSE 80
```
We'll use the lightweight Linux distribution (Alpine) version of the nginx image as a base to build our own. Then, we copy the website files into the image's filesystem and inform Docker that the container will listen to port 80.  

Now we can build the image and try it:
```bash
sudo docker build -t kanban:1.0 .
# This should show the nginx:alpine image and our kanban:1.0
sudo docker images
# Start a container binding the 8080 port on your machine to the containers 80 port
sudo docker run -d --name kanban -p 8080:80 kanban:1.0
# You should be able to see the new container running with
sudo docker ps
```
Now go to http://localhost:8080 and check that you can see the kanban app.  

Here are some other useful commands:
```bash
# Check logs on the container named "kanban"
sudo docker logs kanban
# Execute an interactive shell on the container
sudo docker exec -it kanban sh
# Stop the container
sudo docker stop kanban
# Remove the container
sudo docker rm kanban
# The image remains. The container is the running instance created from that image
```

Commit your progress:
```bash
git add .
git commit -m "feat: create minimal containerized kanban app"
git push
```

### 1.3 Container registry with GitHub Actions

To make the image accessible we will publish it to Github Container Registry. To automate this step, create `.github/workflows/build-and-publish.yaml` and write:

```yaml
name: Build and publish on main branch

on: 
  push:
    branches:
    - "main"

permissions:
  contents: read
  packages: write

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  IMAGE_TAG: ${{ github.sha }}

jobs:
  build-and-publish-head:
    runs-on: ubuntu-latest

    steps:
    - name: Check out repository
      uses: actions/checkout@v4

    - name: Log in to GHCR
      uses: docker/login-action@v4
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract Docker metadata
      id: meta
      uses: docker/metadata-action@v6
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=raw,value=${{ env.IMAGE_TAG }}
          type=raw,value=latest

    - name: Build and push image
      uses: docker/build-push-action@v7
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
```

This will build and publish our image whenever a push is made to the main branch. The image will be available at `ghcr.io/<user>/<repo>:<tag>`. It tags the image by SHA and also adds the `latest` tag.

## **Part 2** -- Container orchestration and monitoring
---
### 2.1. Kubernetes

**Kubernetes** (K8s) is an open-source container orchestration system to automate the deployment, scaling and management of containerized applications. There are a number of options to run it locally, but I used `kind` (Kubernetes in Docker). It is lightweight and runs Kubernetes cluster nodes as Docker containers instead of virtual machines. We'll also need the Kubernetes command-line tool, `kubectl`.

**Requirements:**
- `docker` installed and running
- `kind`
- `kubectl`

**Installation on Arch Linux:**
```bash
# Install the packages
sudo pacman -S kind kubectl
```

We'll start by creating a cluster using `kind`:
```bash
# Choose a name. I used "taskflow" for the kanban app.
sudo kind create cluster --name taskflow
```

Now, instead of writing multiple Kubernetes resources by hand, we will be using **Helm** charts.

### 2.2. Helm

**Helm** is a tool for managing and installing Kubernetes applications using "charts". These are templates that can define Kubernetes resources in a YAML-based format. We'll use Helm charts to separate *dev* and *prod* environment-specific values.

**Requirements:**
- `docker` installed and running
- `kind` cluster created and running
- `kubectl`
- `helm`

**Installation on Arch Linux:**
```bash
# Install the package
sudo pacman -S helm
```

Create a new chart:
```bash
mkdir helm
helm create helm/kanban
```

This creates the following inside the 'helm' directory:
```
kanban/
├── charts/                  # Dependency charts in a packaged .tgz format go here
├── Chart.yaml               # Chart metadata like name and version
├── .helmignore              # Patterns to ignore when packaging
├── templates/               # Kubernetes manifest templates go here
│   ├── deployment.yaml      # Basic manifest for deployment
│   ├── _helpers.tpl         # Helper functions go here
│   ├── hpa.yaml             # Horizontal Pod Autoscaling config (can be deleted)
│   ├── httproute.yaml       # HTTP routing via gateway-api (can be deleted)
│   ├── ingress.yaml         # Maps outside traffic to cluster services
│   ├── NOTES.txt            # Help text displayed when installing this chart
│   ├── serviceaccount.yaml  # Account for services if required (can be deleted)
│   ├── service.yaml         # Basic manifest for a service endpoint
│   └── tests/               # Tests go here (can be deleted)
└── values.yaml              # Default values for the templates
```

For simplification you can remove `hpa.yaml`, `httproute.yaml`, `serviceaccount.yaml`, and `tests`.  

Replace the default values in `values.yaml` with your image repository:
```yaml
replicaCount: 2

image:
  repository: ghcr.io/iagooteroc/devops-project
  tag: latest
  pullPolicy: IfNotPresent

resources:
  requests:
    cpu: 10m
    memory: 16Mi
  limits:
    cpu: 100m
    memory: 64Mi

serviceAccount:
  create: false
  automount: false
```

Then create `dev.yaml` and `prod.yaml` inside a new folder `helm/kanban/values/`. These will be used to overwrite the default values in `values.yaml`. For example, define more replicas and a higher resource limit for production (`prod.yaml`):

```yaml
replicaCount: 3 # Increased number of replicas

image:
  repository: ghcr.io/iagooteroc/devops-project
  tag: latest # This should be a fixed version in prod
  pullPolicy: IfNotPresent

resources: # Higher resource limit
  requests:
    cpu: 25m
    memory: 32Mi
  limits:
    cpu: 250m
    memory: 128Mi
```

You can check how the final chart will look like with the `helm template <release-name> <chart-name>` command, but first validate it:
```bash
helm lint helm/kanban
```
It should say `1 chart(s) linted, 0 chart(s) failed`. Now you can render the templates with the default values:
```bash
helm template default-release helm/kanban
```
And compare it with the production release:
```bash
helm template prod-release helm/kanban \
  -f helm/kanban/values/prod.yaml
```
You should see the number of replicas has been replaced with the value from `prod.yaml`. This would be the Kubernetes YAML to be installed in production.

Let's install the chart and create a namespace for it:
```bash
sudo helm install taskflow helm/kanban \
  --namespace taskflow \
  --create-namespace \
  -f helm/kanban/values/prod.yaml
```

You can check when the pods are ready with:
```bash
sudo kubectl get all -n taskflow
```
If any get stuck on `ImagePullBackOff` status, you can get more information with:
```bash
sudo kubectl describe <pod-name> -n taskflow
```

To access the app for now we need to use port forwarding since the Service is running in ClusterIP mode (cluster-scoped virtual IP address) on port 80:
```bash
sudo kubectl port-forward \
  service/taskflow-kanban \
  -n taskflow \
  8080:80
```

Keep that running and go to http://localhost:8080. You should see the kanban app.

\# TODO: helm commands (status, get, upgrade, history, rollback...)  
You can view the values used by the release with:
```bash
# Get values
sudo helm get values taskflow \
  --namespace taskflow
# Get all values, including defaults
sudo helm get values taskflow --all \
  --namespace taskflow
```

If you want to change a value, you would have to edit the chart and re-install it. Instead of doing that manually, we will use **Argo CD**.

### 2.3. Prometheus and Grafana

\# TODO: Maybe move this to after CI/CD?

## **Part 3** -- CI/CD
---
### 3.1. CI with GitHub Actions

>TODO:  
>Every push validates:  
>    yamllint - YAML syntax.  
>    manifest-validate - kubeconform against the target Kubernetes schema.  
>    helm-validate - helm template on every umbrella chart.  
>    argocd-validate - ArgoCD Application YAML structure.

### 3.2. Argo CD

**Argo CD** is a declarative, GitOps continuous delivery tool for Kubernetes. It considers the state declared in Git to be the desired state of the application and will sync the Kubernetes cluster to match it.  

**Requirements:**
- `docker` installed and running
- `kind` cluster created and running
- `kubectl`
- `helm`
- `argocd`

**Installation on Arch Linux:**
```bash
# Install the package
sudo pacman -S argocd
```

Install Argo CD using its official manifest:
```bash
# Recreate the cluster if you have anything running
sudo kind delete cluster --name taskflow
sudo kind create cluster --name taskflow

# Create a namespace for Argo CD
sudo kubectl create namespace argocd

# Install
sudo kubectl apply \
  -n argocd \
  --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

The `--server-side --force-conflicts` parameters are now part of the official installation command due to CRD size limitations.

Check the status of the pods until they are `Running`:
```bash
sudo kubectl get pods -n argocd
# wait until the main components are running
sudo kubectl get pods -n argocd -w
```

You should now be able to access Argo CD web interface with port forwarding:
```bash
sudo kubectl port-forward \
  -n argocd \
  svc/argocd-server \
  8081:443
```

Open https://localhost:8081. Your browser should show a warning due to the self-signed certificate. Proceed anyway.

To retrieve the initial admin password:
```bash
sudo argocd admin initial-password -n argocd
```
Use it to login and change the password through the UI or CLI:
```bash
sudo argocd login localhost:8081
sudo argocd account update-password
```

Now you need to have your **Helm** charts accesible in your repository, so commit them if you haven't already:
```bash
git add .
git commit -m "feat: helm chart for kanban app"
git push
```

Let's create the Argo CD Application
```bash
mkdir -p argocd
touch argocd/taskflow-application.yaml
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: taskflow
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/<username>/<repo>.git
    targetRevision: main
    path: helm/kanban
    helm:
      valueFiles:
        - values/prod.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: taskflow

  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
    - CreateNamespace=true
```
Replace with your repository URL and application name.

Apply it:
```bash
sudo kubectl apply -f argocd/taskflow-application.yaml
```

Now you should see the app in the UI and via the CLI:
```bash
sudo argocd app list
sudo argocd app get taskflow
```


```bash
sudo kubectl logs -n argocd \
  argocd-applicationset-controller-86bcdc46cf-vwmn7 \
  -c argocd-applicationset-controller \
  --previous
```

app-of-apps pattern:
https://www.arthurkoziel.com/setting-up-argocd-with-helm/


### 3.3 Prometheus and Grafana

```bash
touch argocd/prometheus-stack-application.yaml
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prometheus-stack
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 92.11.0
    helm:
      values: |
        pushgateway:
          enabled: false
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```bash
sudo kubectl apply -f argocd/prometheus-stack-application.yaml
```

Find the Grafana Service: (replace monitoring with taskflow)
```bash
sudo kubectl get services -n monitoring
```

Port forward it:
```bash
sudo kubectl port-forward \
  -n monitoring \
  service/monitoring-grafana \
  3000:80
```

To reach this point:
```bash
sudo kind delete cluster --name taskflow
sudo kind create cluster --name taskflow
sudo kubectl create namespace argocd
sudo kubectl apply \
  -n argocd \
  --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
sudo kubectl apply -f argocd/prometheus-stack-application.yaml

```

---


>Each container acts as a Kubernetes node running kubelet, containerd, and the necessary control plane or worker components.


```bash
sudo kubectl cluster-info
sudo kubectl get nodes
```

## **Part 4** -- Infrastructure as Code
---
### 4.1. Terraform/OpenTofu

### 4.2. Ansible

## **Part 5** -- Public cloud deployment
---

-----

