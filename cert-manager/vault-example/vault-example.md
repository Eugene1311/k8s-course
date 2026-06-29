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
helm install vault hashicorp/vault -n vault --create-namespace --values vault-values.yaml
```
## Wait until the Vault pods are Ready 1/1 and status is Running.
```shell
kubectl get pods -n vault
```
## Check Vault UI: localhost:8200, token - "root"
## Install cert-manager
```shell
helm install cert-manager oci://quay.io/jetstack/charts/cert-manager \
--namespace cert-manager \
--create-namespace \
--set crds.enabled=true
```
## Install ingress-nginx
```shell
helm install quickstart ingress-nginx/ingress-nginx
```
## Put 'demo.localdev.me' into /etc/hosts against '127.0.0.1'

# 2. Configure Vault
## Connect to the Vault instance
```shell
kubectl exec --stdin=true --tty=true vault-0 -n vault -- /bin/sh
```
## Enable the PKI (Certificates) Secrets engine
```shell
vault secrets enable pki
```
## Tune the secret engine so that your certificates satisfy the Certificate manager 90-day duration request
```shell
vault secrets tune -max-lease-ttl=8760h pki
```
## Create a self-signed CA certificate and key pair with a customized expiration
```shell
vault write pki/root/generate/internal common_name=demo.localdev.me ttl=8760h
```
## Configure URLs for creating the certificates
```shell
vault write pki/config/urls issuing_certificates="http://vault.vault.svc.cluster.local:8200/v1/pki/ca" crl_distribution_points="http://vault.vault.svc.cluster.local:8200/v1/pki/crl"
```
## Create a role that can process the certificate signing requests
```shell
vault write pki/roles/my-role allowed_domains=demo.localdev.me allow_subdomains=true allow_any_name=true allow_localhost=true enforce_hostnames=false max_ttl=8760h
```
## Create Issuers by using the Token Authentication:
## Create a policy that enables usage of the PKI Vault APIs
```shell
cd /tmp
```
```shell
echo 'path "pki*" {  capabilities = ["create", "read", "update", "delete", "list", "sudo"]}' > pki_policy.hcl
```
```shell
vault policy write pki_policy pki_policy.hcl
```
## Create a token that uses the policy that you just created.
## Be sure that the ttl setting is longer than the period that you want to issue renewals against the token that you created
```shell
vault write /auth/token/create policies="pki_policy" no_parent=true no_default_policy=true renewable=true ttl=767h num_uses=0
```
## Encode token to base64
```shell
echo -n '<token>' | base64
```
## Put encoded token into secret and create it
```shell
kubectl apply -f manifests/vault-token-secret.yml
```
## Create Issuer
```shell
kubectl apply -f manifests/vault-issuer.yml
```
## Create Certificate
```shell
kubectl apply -f manifests/vault-certificate.yml
```
## Check READY status of created certificate is 'True'
```shell
kubectl get certificates
```

## Create kuard deployment, service and ingress
```shell
kubectl apply -f manifests/kuard-deployment.yml
kubectl apply -f manifests/kuard-service.yml
kubectl apply -f manifests/kuard-ingress.yml
```

## Check that on this requests server returns signed certificate
```shell
curl -kivL -H 'Host: demo.localdev.me' 'http://localhost'
```

## Check that on this requests server returns nginx 404 page
```shell
curl -kivL 'http://localhost'
```
