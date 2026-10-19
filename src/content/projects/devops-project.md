Sources:  
https://docs.docker.com/reference/dockerfile/  
https://helm.sh/docs  
https://argo-cd.readthedocs.io/  
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
Now go to http://127.0.0.1:8080 and check that you can see the kanban app.  

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

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  IMAGE_TAG: ${{ github.event.workflow_run.head_sha }}

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

\# TODO: finish this with the minimum requirements for Helm to work

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

Replace the default values in `values.yaml` using your image repository:
```yaml
replicaCount: 2

image:
  repository: ghcr.io/iagooteroc/kanban-platform
  tag: latest
  pullPolicy: IfNotPresent

resources:
  requests:
    cpu: 10m
    memory: 16Mi
  limits:
    cpu: 100m
    memory: 64Mi
```

Then create `dev.yaml` and `prod.yaml` inside a new folder `helm/kanban/values/`. These will be used to overwrite the default values in `values.yaml`. For example, define more replicas and a higher resource limit for production (`prod.yaml`):

```yaml
replicaCount: 3 # Increased number of replicas

image:
  repository: ghcr.io/iagooteroc/kanban-platform
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

\# TODO: `Chart.yaml`, `_helpers.tpl`, `deployment.yaml`, `service.yaml`, `ingress.yaml`
\# TODO: `helm lint`, `helm template`

```bash
helm install ...
```

\# TODO: temporarely port forward:
```bash
kubectl port-forward \
  -n taskflow \
  service/taskflow-kanban \
  8080:80
```

And test http://localhost:8080

\# TODO: helm commands (status, get, upgrade, history, rollback...)


### 2.3. Prometheus and Grafana

\# TODO: Maybe move this to after CI/CD?

## **Part 3** -- CI/CD
---
### 3.1. CI with GitHub Actions



### 3.2. Argo CD

**Argo CD** is a declarative, GitOps continuous delivery tool for Kubernetes. It considers the state declared in Git to be the desired state of the application and will sync the Kubernetes cluster to match it.  



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

