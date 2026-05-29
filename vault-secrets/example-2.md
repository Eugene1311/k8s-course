# Dynamic secret with the database secrets engine

# 1. Install PostgreSQL pod
## Create a namespace for the PostgreSQL pod.
```shell
kubectl create ns postgres
```
## Add the Bitnami repository to your local Helm.
```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
```
## Install PostgreSQL.
```shell
helm upgrade --install postgres bitnami/postgresql --namespace postgres \
--set auth.audit.logConnections=true  --set auth.postgresPassword=secret-pass
```

# 2. Setup PostgreSQL
## Connect to the Vault instance.
```shell
kubectl exec --stdin=true --tty=true vault-0 -n vault -- /bin/sh
```
## Enable an instance of the Database Secrets Engine.
```shell
vault secrets enable -path=demo-db database
```
## Configure the Database Secrets Engine.
```shell
vault write demo-db/config/demo-db \
   plugin_name=postgresql-database-plugin \
   allowed_roles="dev-postgres" \
   connection_url="postgresql://{{username}}:{{password}}@postgres-postgresql.postgres.svc.cluster.local:5432/postgres?sslmode=disable" \
   username="postgres" \
   password="secret-pass"
```
## Create a role for the PostgreSQL pod.
```shell
vault write demo-db/roles/dev-postgres \
   db_name=demo-db \
   creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
      GRANT ALL PRIVILEGES ON DATABASE postgres TO \"{{name}}\";" \
   revocation_statements="REVOKE ALL ON DATABASE postgres FROM  \"{{name}}\";" \
   backend=demo-db \
   name=dev-postgres \
   default_ttl="1m" \
   max_ttl="1m"
```
## Create the demo-auth-policy-db policy.
```shell
vault policy write demo-auth-policy-db - <<EOF
path "demo-db/creds/dev-postgres" {
   capabilities = ["read"]
}
EOF
```
## Disconnect to the Vault instance.
```shell
exit
```
# Check created secret in k8s
```shell
kubectl get secret postgres-postgresql -o yaml -n postgres
```

# 3. Transit encryption
## Connect back to the Vault instance.
```shell
kubectl exec --stdin=true --tty=true vault-0 -n vault -- /bin/sh
```
## Enable an instance of the Transit Secrets Engine at the path demo-transit.
```shell
vault secrets enable -path=demo-transit transit
```
## Create a encryption key.
```shell
vault write -force demo-transit/keys/vso-client-cache
```
## Create a policy for the operator role to access the encryption key.
```shell
vault policy write demo-auth-policy-operator - <<EOF
path "demo-transit/encrypt/vso-client-cache" {
   capabilities = ["create", "update"]
}
path "demo-transit/decrypt/vso-client-cache" {
   capabilities = ["create", "update"]
}
EOF
```
## Create Kubernetes auth role for the operator.
```shell
vault write auth/demo-auth-mount/role/auth-role-operator \
   bound_service_account_names=vault-secrets-operator-controller-manager \
   bound_service_account_namespaces=vault-secrets-operator-system \
   token_ttl=0 \
   token_period=120 \
   token_policies=demo-auth-policy-operator \
   audience=vault
```

# 4. Setup dynamic secrets
## Create a new role for the dynamic secret
```shell
vault write auth/demo-auth-mount/role/auth-role \
   bound_service_account_names=demo-dynamic-app \
   bound_service_account_namespaces=demo-ns \
   token_ttl=0 \
   token_period=120 \
   token_policies=demo-auth-policy-db \
   audience=vault
```

# 5. Create the application
## Create a new namespace.
```shell
kubectl create ns demo-ns
```
## Create the app, Vault connection, authentication, service account and corresponding secrets.
```shell
kubectl apply -f dynamic-secrets/.
```
## Check created secrets
```shell
kubectl get secrets -n demo-ns
```
## Check "vso-db-demo" secret
```shell
kubectl get secret vso-db-demo -n demo-ns -o yaml
```
## Check "vso-db-demo" secret has been changed in 1 minute
```shell
kubectl get secret vso-db-demo -n demo-ns -o yaml
```
