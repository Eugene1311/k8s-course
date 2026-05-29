# Tutorial taken from here https://developer.hashicorp.com/vault/tutorials/kubernetes-introduction/vault-secrets-operator

# 1. Install Vault cluster
## Add the HashiCorp repository.
```shell
helm repo add hashicorp https://helm.releases.hashicorp.com
```
## Update to the latest version of the HashiCorp Helm charts, update the repository.
```shell
helm repo update
```
## Determine the latest version of Vault.
```shell
helm search repo hashicorp/vault
```
## Using the YAML file in the appropriate sub-folder, install Vault on your cluster
```shell
helm install vault hashicorp/vault -n vault --create-namespace --values vault/vault-values.yaml
```
## Wait until the Vault pods are Ready 1/1 and status is Running.
```shell
kubectl get pods -n vault
```
## Check Vault UI: localhost:8200, token - "root"

# 2. Configure Vault
## Connect to the Vault instance
```shell
kubectl exec --stdin=true --tty=true vault-0 -n vault -- /bin/sh
```
## Move into the tmp directory
```shell
cd /tmp
```
## Enable the Kubernetes auth method
```shell
vault auth enable -path demo-auth-mount kubernetes
```
## Configure the auth method.
```shell
vault write auth/demo-auth-mount/config \
   kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
```
## Enable the kv v2 Secrets Engine.
```shell
vault secrets enable -path=kvv2 kv-v2
```
## Create a JSON file with a Vault policy.
```shell
tee webapp.json <<EOF
path "kvv2/data/webapp/config" {
   capabilities = ["read", "list"]
}
EOF
```
## Add policy
```shell
vault policy write webapp webapp.json
```
## Create a role in Vault to enable access to secrets within the kv v2 secrets engine.
```shell
vault write auth/demo-auth-mount/role/role1 \
   bound_service_account_names=demo-static-app \
   bound_service_account_namespaces=app \
   policies=webapp \
   audience=vault \
   ttl=24h
```
## Create a secret.
```shell
vault kv put kvv2/webapp/config username="static-user" password="static-password"
```
## Exit the Vault instance.

# 3. Install the Vault Secrets Operator
## Use Helm to deploy the Vault Secrets Operator.
```shell
helm install vault-secrets-operator hashicorp/vault-secrets-operator \
    -n vault-secrets-operator-system --create-namespace \
    --values vault/vault-operator-values.yaml
```
## Check what was added.
```shell
kubectl get all -n vault-secrets-operator-system
```

# 4. Deploy and sync a secret
## Create a namespace called app on your Kubernetes cluster.
```shell
kubectl create ns app
```
## Set up Kubernetes authentication for the secret.
```shell
kubectl apply -f vault/vault-auth-static.yaml
```
## Create the secret names secretkv in the app namespace.
```shell
kubectl apply -f vault/static-secret.yaml
```
## Check secret was created
```shell
kubectl get secrets -n app
```

# 5. Rotate the static secret
## Connect to the Vault instance.
```shell
kubectl exec --stdin=true --tty=true vault-0 -n vault -- /bin/sh
```
## Rotate the secret.
```shell
vault kv put kvv2/webapp/config username="static-user2" password="static-password2"
```
## Exit Vault.
```shell
exit
```
## Check secret was changed in Vault: http://localhost:8200/ui/vault/secrets/kvv2/kv/webapp%2Fconfig/details?version=2
## Check secret was changed in k8s cluster
```shell
kubectl get secret secretkv -n app -o yaml
```
