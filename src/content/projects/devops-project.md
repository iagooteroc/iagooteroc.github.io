Sources:  
https://docs.docker.com/reference/dockerfile/  
...  
https://learn.microsoft.com/en-us/azure/virtual-machines/windows/quick-create-terraform  
https://learn.microsoft.com/en-us/azure/developer/terraform/

-----

## Overview

In this project, I have documented all the necessary steps to build a basic DevOps workflow from scratch, including:
- Infrastructure as Code (IaC)
- Container orchestration
- Continuous Integration and Continuous Delivery (CI/CD)
- Monitoring and observability
- Public cloud deployment

## 1. Git repository

We will be using **Git** as the source of truth, so start by creating a repository in the platform of your choice (GitHub, GitLab, Codeberg...).

### 1. 1. Requirements

We need to install the following requirements:

**Requirements:**
- `git`

**Installation on Arch Linux:**
```bash
# Install the package
sudo pacman -S git
```

Either create the repository first through the web interface and `git clone` it, or start with an empty folder and do `git init` inside, adding the remote repository later with `git remote add origin <URL>`.

## 2. Minimal containerized app

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
# Check logs on the container named 'kanban'
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

## 3. Kubernetes

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
# Choose a name. I used 'taskflow' for the kanban app.
sudo kind create cluster --name taskflow
```

>Each container acts as a Kubernetes node running kubelet, containerd, and the necessary control plane or worker components.


```bash
sudo kubectl cluster-info
sudo kubectl get nodes
```


-----

