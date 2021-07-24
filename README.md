### https://learn.hashicorp.com/tutorials/vault/kubernetes-sidecar
### https://learn.hashicorp.com/tutorials/vault/database-secrets

Para criar o Vault Precisará de um S3 bucket e access_key, secret_key e deverá ser configurado no arquivo vault.yaml: linhas 6 e 7 e para o S3 na linha 80
As chaves da AWS deveram esta em base64

### Após configurar será necessario seguir o passo a passo

```bash
kubectl create namespace vault

kubectl apply -f vault.yaml

kubectl -n vault get pods --watch

### Com o output do comando execute efetue o unseal do vault

kubectl -n vault exec --stdin=true --tty=true vault-0 -- vault operator init

#### Preenchendo a key1,key2 e key3 com a chaves de unseal do output acima

for i in key1 key2 key2
do
kubectl -n vault  exec --stdin=true --tty=true vault-0 -- vault operator unseal $i
done

kubectl get pods --selector='app.kubernetes.io/name=vault' -n vault

kubectl apply -f postgres.yaml

kubectl -n  vault exec -it vault-0 -- /bin/sh

export VAULT_TOKEN=" "

vault auth enable kubernetes

vault write auth/kubernetes/config \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

#### Verificar ip do banco

```bash

kubectl describe pods \
    $(kubectl get pod -l app=postgres -o jsonpath="{.items[0].metadata.name}") | grep IP

````
### Configure o mecanismo de segredos do banco de dados com as credenciais de conexão para o banco de dados Postgres.

```bash

vault secrets enable database

vault write database/config/postgresql \
     plugin_name=postgresql-database-plugin \
     connection_url="postgresql://{{username}}:{{password}}@{IP_DO_BANCO}/postgres?sslmode=disable" \
     allowed_roles=readonly \
     username="root" \
     password="rootpassword"

tee readonly.sql <<EOF
CREATE ROLE "{{name}}" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}' INHERIT;
GRANT ro TO "{{name}}";
EOF

vault write database/roles/readonly \
      db_name=postgresql \
      creation_statements=@readonly.sql \
      default_ttl=1h \
      max_ttl=24h

vault write auth/kubernetes/role/web \
    bound_service_account_names=web \
    bound_service_account_namespaces=default \
    policies=web \
    ttl=1h

```

#### Acessando banco para criar role

```bash

kubectl exec -ti \
    $(kubectl get pod -l app=postgres -o jsonpath="{.items[0].metadata.name}") \
    --container postgres bash

psql
CREATE ROLE ro NOINHERIT;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO "ro";

SELECT usename, valuntil FROM pg_user;

```
### Criando Policy para acesso

```bash
vault policy write web web-policy.hcl - <<EOF
path "database/creds/readonly" {
  capabilities = ["read"]
}
EOF

vault read database/creds/readonly

```
## Aplicando o yamlfile do web.yaml e verificando a criação do user e da senha

```bash
kubectl apply -f web.yaml

kubectl get serviceaccounts

kubectl exec \
    $(kubectl get pod -l app=web -o jsonpath="{.items[0].metadata.name}") \
    --container web -- cat /vault/secrets/db-creds
```
## Acessa o banco e verifique o user criado
```
kubectl exec -ti \
    $(kubectl get pod -l app=postgres -o jsonpath="{.items[0].metadata.name}") \
    --container postgres bash

SELECT usename, valuntil FROM pg_user;

```
