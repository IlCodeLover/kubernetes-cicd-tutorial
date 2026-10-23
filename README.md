# Kubernetes CICD Tutorial

This repo contains all the code needed to follow along with our **[YouTube Tutorial](https://youtu.be/GlhK7mz5IJo)** or **[Written Article](https://rslim087a.github.io/rayanslim/lesson.html?course=kubernetes-cicd-course&lesson=introduction)**.

## Prerequisites

To follow along with this tutorial, you'll need:

- kubectl installed and configured ([https://youtu.be/IBkU4dghY0Y](https://youtu.be/IBkU4dghY0Y))
- Helm installed: ([https://rslim087a.github.io/rayanslim/lesson.html?course=prometheus-grafana-monitoring-course&lesson=helm-installation](https://rslim087a.github.io/rayanslim/lesson.html?course=prometheus-grafana-monitoring-course&lesson=helm-installation))
- A GitHub account: ([https://github.com/](https://github.com/))

## Install ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create namespace argocd
helm install argocd argo/argo-cd --namespace argocd --version 7.7.0
```

## Access ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:80
```

## Retrieve Credentials

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
## ArgoCD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: grade-submission-api
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/grade-api-gitops.git
    targetRevision: HEAD
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Creating Image Pull Secrets

To allow Kubernetes to pull images from GitHub Container Registry:

```yaml
kubectl create secret docker-registry ghcr-secret \\
  --docker-server=ghcr.io \\
  --docker-username=YOUR_USERNAME \\
  --docker-password=YOUR_PAT \\
  --namespace=default
```

## Troubleshooting

### dial unix backend.sock: connect: connection refused

The message Post "http://ipc/registry/credstore-updated": dial unix backend.sock: connect: connection refused means that the Docker CLI on your self-hosted runner machine is trying to use an external credential store (like Docker Desktop or a specialized credential helper) that expects a background socket (backend.sock) to be running.
Config docker so that self-host runner can access docker
```
(base) ➜  ~ cat ~/.docker/config.json
{
"auths": {
"https://index.docker.io/v1/": {}
},
"_disabled_credsStore": "desktop" // old value: "credsStore": "desktop"
}

```

### Configure self-host runner on repo 
- Open Github, go to rpo kubernetes-cicd-tutorial -> Setting --> Runner
- Create self-host runner -> select OS, architecture, then this code is auto generated. Run them line-by-line on your local machine to config runner on your local.

```bash
**Download**
# Create a folder
$ mkdir actions-runner && cd actions-runner# Download the latest runner package
$ curl -o actions-runner-osx-x64-2.338.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.338.0/actions-runner-osx-x64-2.338.0.tar.gz# Optional: Validate the hash
$ echo "dea7a58796ce215fc424a8b27c0fdd9b813fa5fc03d109a4a1d38131d6bfa1a3  actions-runner-osx-x64-2.338.0.tar.gz" | shasum -a 256 -c# Extract the installer
$ tar xzf ./actions-runner-osx-x64-2.338.0.tar.gz
```

**Configure**
```bash
# Create the runner and start the configuration experience
$ ./config.sh --url https://github.com/IlCodeLover/kubernetes-cicd-tutorial --token A55LN5OW4EB7AHHTMDGJN2TK3N3LY# Last step, run it!
$ ./run.sh
Using your self-hosted runner
# Use this YAML in your workflow file for each job
runs-on: self-hosted
```

**Go to my source code repo kubernetes-cici-tutorial, click on Actions, see if there is any job running when you push code.


### References

Visit[https://rslim087a.github.io/rayanslim](https://rslim087a.github.io/rayanslim)
