# Kubernetes CICD Tutorial

This repo contains all the code needed to follow along with our **[YouTube Tutorial](https://youtu.be/GlhK7mz5IJo)** or **[Written Article](https://rslim087a.github.io/rayanslim/lesson.html?course=kubernetes-cicd-course&lesson=introduction)**.

## Prerequisites

To follow along with this tutorial, you'll need:

- kubectl installed and configured ([https://youtu.be/IBkU4dghY0Y](https://youtu.be/IBkU4dghY0Y))
- Helm installed: ([https://rslim087a.github.io/rayanslim/lesson.html?course=prometheus-grafana-monitoring-course&lesson=helm-installation](https://rslim087a.github.io/rayanslim/lesson.html?course=prometheus-grafana-monitoring-course&lesson=helm-installation))
- A GitHub account: ([https://github.com/](https://github.com/))

## CI
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
$ ./config.sh --url https://github.com/IlCodeLover/kubernetes-cicd-tutorial --token <TOKEN># Last step, run it!
$ ./run.sh
Using your self-hosted runner
# Use this YAML in your workflow file for each job
runs-on: self-hosted
```

- Go to my source code repo kubernetes-cici-tutorial, click on Actions, see if there is any job running when you push code.

- Check GHCR at this link: https://github.com/IlCodeLover?tab=packages

## Install ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create namespace argocd
helm install argocd argo/argo-cd --namespace argocd --version 7.7.0

NAME: argocd
LAST DEPLOYED: Fri Oct 23 17:42:50 2026
NAMESPACE: argocd
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None
NOTES:
In order to access the server UI you have the following options:

1. kubectl port-forward service/argocd-server -n argocd 8080:443

    and then open the browser on http://localhost:8080 and accept the certificate
    
    - Then Argo login: user = admin, password (see step 3)
2. enable ingress in the values file `server.ingress.enabled` and either
      - Add the annotation for ssl passthrough: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-1-ssl-passthrough
      - Set the `configs.params."server.insecure"` in the values file and terminate SSL at your ingress: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-2-multiple-ingress-objects-and-hosts


3. After reaching the UI the first time you can login with username: admin and the random password generated during the installation. 
You can find the password by running:

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

- Then run command port forward to relogin with password

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
- Copy this content and save into file argocd-application.yaml in any folder (outside our existing folders)
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

- update git repo to yrs
- Run
```
kubectl apply -f argocd-applicaton.yaml
// application.argoproj.io/grade-submission-api created


```
- Open argo on browser, you will see application grade-submission-api.

- See error "Failed to pull image "ghcr.io/ilcodelover/kubernetes-cicd-tutorial:c81542fc3db516de5ca6ceca1b7d6b39e637a52c": rpc error: code = Unknown desc = Error response from daemon: Head "https://ghcr.io/v2/ilcodelover/kubernetes-cicd-tutorial/manifests/c81542fc3db516de5ca6ceca1b7d6b39e637a52c": unauthorized"
  - Root cause: GHCR is private, so K8s cannot pull image from there. 
  - Solution: config pull secrets for K8s
  
## Creating Image Pull Secrets

To allow Kubernetes to pull images from GitHub Container Registry:

```yaml
kubectl create secret docker-registry ghcr-secret \\
  --docker-server=ghcr.io \\
  --docker-username=YOUR_USERNAME \\
  --docker-password=YOUR_PAT \\
  --namespace=default

# running multiline migh has error, you can use
kubectl create secret docker-registry ghcr-secret --docker-server=ghcr.io --docker-username=<YOUR-GITHUB-USERNAME> --docker-password=<YOUR-GITHUB-TOKEN> --namespace=default

# delete it in case yr credential was set wrongly
kubectl delete secret ghcr-secret -n default

# then recreate it with correct credential


```
- Go to Argo URL, click on Sync, choose options: Prune, Force, Replace. Then you should see the system healthy and pod is created OK.

- Now you can run API on the K8s
```bash
#Keep the port-forward running in another terminal
kubectl port-forward svc/grade-submission-api -n default 3000:80
# how to know port 80 where the web app runs
#kubectl get svc grade-submission-api -n default 2>&1)
  NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
grade-submission-api   ClusterIP   10.104.139.142   <none>        80/TCP    18m

#spec:
#    ports:
#    - port: 80          # the Service listens here       ← used in "3000:80"
#      targetPort: 3000  # it forwards to this port on the pod


# the web app is accessed at port 3000
  curl -X POST http://localhost:3000/grades \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","subject":"Math","score":95}'
  
 # open https://localhost:3000/grades
```


## ArgoCD Setup
- Github Action push image to GHCR --> trigger ArgoCD --> detect the state different bwt the desired state that exists in git and the current one in K8s -> it will auto deploy new images to K8s
- Precondition:
  - Kubectl must be installed
  - Helm must be installed (brew install helm)

- Install ArgoCD
- 
## Troubleshooting
###  kubectl create namespace argocd --> got error "The connection to the server localhost:8080 was refused - did you specify the right host or port?"
- Root cause: kubectl has no Kubernetes cluster to talk to. You don't have a ~/.kube/config file and no contexts are set, so kubectl falls back to localhost:8080, where
nothing is running.
- Solution: create a cluster.
```bash
1. Open Docker Desktop → Settings (⚙️) → Kubernetes.
2. Tick Enable Kubernetes, then click Apply & Restart. Setup takes a few minutes and is done when the Kubernetes icon at the bottom turns green.
3. This creates ~/.kube/config with a docker-desktop context. Check it with:
   
   kubectl get nodes        # should show: docker-desktop   Ready
   kubectl create namespace argocd
   kubectl get namespace # should show your namespace
```
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

### References

Visit[https://rslim087a.github.io/rayanslim](https://rslim087a.github.io/rayanslim)
