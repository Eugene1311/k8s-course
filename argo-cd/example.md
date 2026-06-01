# Argo-CD demo taken from https://argo-cd.readthedocs.io/en/stable/getting_started/
# Argo-CD architecture: https://argo-cd.readthedocs.io/en/stable/operator-manual/architecture/

## Create 'argocd' objects
```shell
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
## Install argocd CLI
## Port-forward 'argocd-server'
```shell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
## The API server can then be accessed using https://localhost:8080
## Open another terminal
## Get password for argocd admin user
```shell
argocd admin initial-password -n argocd
```
## Using the username 'admin' and the password from above, log in to Argo CD's IP or hostname:
```shell
argocd login localhost:8080
```
## Switch to argocd namespace
```shell
kubectl config set-context --current --namespace=argocd
```
## Create app from Github repo
```shell
argocd app create guestbook --repo https://github.com/argoproj/argocd-example-apps.git \
    --path guestbook --dest-server https://kubernetes.default.svc --dest-namespace default
```
## Once the guestbook application is created, you can now view its status:
```shell
argocd app get guestbook
```
## Sync app
```shell
argocd app sync guestbook
```
