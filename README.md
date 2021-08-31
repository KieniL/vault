# command repo for vault <!-- omit in toc -->
- [K8s Integration](#k8s-integration)
  - [Installation (with helm)](#installation-with-helm)
    - [Install release](#install-release)
    - [describe the serviceaccount to see the mountable secret](#describe-the-serviceaccount-to-see-the-mountable-secret)
    - [Now export the vault-secret name to a variable:](#now-export-the-vault-secret-name-to-a-variable)
    - [describe the secret](#describe-the-secret)
    - [enable kubernetes as auth method](#enable-kubernetes-as-auth-method)
    - [get the token from the secret](#get-the-token-from-the-secret)
    - [get the Kubernetes CA Certificate:](#get-the-kubernetes-ca-certificate)
    - [get the Kubernetes Host URL](#get-the-kubernetes-host-url)
    - [configure the k8s auth method to use the service account token, the location of the host and its certificate:](#configure-the-k8s-auth-method-to-use-the-service-account-token-the-location-of-the-host-and-its-certificate)
    - [enable the secret store for k8s](#enable-the-secret-store-for-k8s)
    - [write a policy for read access to a secret](#write-a-policy-for-read-access-to-a-secret)
    - [create a kubernetes authentication role with access to the policy:](#create-a-kubernetes-authentication-role-with-access-to-the-policy)
    - [create a pod with injected annotations](#create-a-pod-with-injected-annotations)


## Installation
The installation was done with: https://github.com/KieniL/software_installer

## Further documentation
The steps were taken from https://github.com/b1tsized/vault-tutorial and the corresponding youtube playlist


## Notes
Since I installed it with a self signed certificate I had to skip-tls-verify on unseal
vault operator unseal -tls-skip-verify

Also this flag is needed for every command e.g.

vault kv list -tls-skip-verify bstv1
## enable a a secret type
vault secrets enable kv

e.g to enable keyvalue: vault secrets enable kv

This will enable the kv on the kv path.
You can also set your own path with -path like so:
vault secrets enable -path=test kv

## disable a a secret type
vault secrets disable TYPE

e.g to disable keyvalue: vault secrets disable kv

---
**WARNING**
If you entered a custom path you can't disable it like using the type you need to disable it on the path.
If you enabled it with -path=test you need to disable it like this:
vault secrets disable test/ 

---

## input a secret 
vault TYPE put TYPE/NAME key=value

e.g to add a keyvalue pair secret named secret:
vault kv put kv/secret username=demo

---
**WARNING**
Doing this on an existing secret will overwrite it

You should first read the value with get and the put both values into it

e.g. 
vault kv put kv/secret username=demo
vault kv get kv/secret

vault kv put kv/secret username=demo pw=pw
---

## delete a secret
vault TYPE delete TYPE/NAME

e.g
vault kv delete kv/secret

This will delete the secret in the kv directory if it exists (it is idempotent) 

## list all secrets in an directory
vault TYPE list TYPE/

e.g.
vault kv list kv/


## get a secret
vault kv get TYPE/NAME

e.g.
vault kv get kv/secret

### format the get to display it in json
vault kv get -format=json TYPE/NAME 

vault kv get -format=json kv/secret

There is a data key which contains the secret data

## Get Secret by API
curl -H 'X-Vault-Token: TOKEN' -X GET VAULT_URL:VAULT_PORT/v1/SECRETPATH/SECRETNAME

curl -H 'X-Vault-Token: TOKEN' -X GET http://vault.local.at:8200/v1/bst/kv1/secret


You can also use Authorization:Bearer TOKEN instead of X-VAULT_TOKEN: TOKEN

---
**WARNING**
This can only be done on version 1 kv's if you try to access a version 2 kv to need to add data to the url like so:

curl -H 'X-Vault-Token: TOKEN' -X GET http://vault.local.at:8200/v1/bst/kv2/data/secret

---


## Create Secret by API

I created the payload.json for this

This is the command to put the secret:

curl -H 'Authorization:Bearer TOKEN' -X POST http://vault.local.at:8200/v1/bst/kv2/data/secret --data @payload.json | jq


## Delete Secret by API

curl -H 'Authorization:Bearer TOKEN' -X DELETE http://vault.local.at:8200/v1/bst/kv2/data/secret | jq


## enable pki
vaut secrets enable pki

## tune a secret
Tune a secret for example setting a max-lease time to live

vault secrets tune --max-lease-ttl=87600h pki

## write rootcertificate
<code>vault write -field=certificate pki/root/generate/internal \
issuing_certificates="http://vault.local.at:8200/v1/pki/crl" \
crl_distribution_points=\http://vault.local.at:8200/v1/pki/crl</code>


#### Then another secret path is enabled:
vault secrets enable -path=pki_int pki

#### Then a csr is created:
<code>vault write -format=json pki_int/intermediate/generate/internal \
common_name="local.at Intermediate Authority" \
|jq -r '.data.csr' > pki_intermediate.csr</code>

#### Then you can sign the csr to create the cert:
<code>vault write -format=json pki/root/sign-intermediate \
 csr=@pki_intermediate.csr \
 format=pem_bundle ttl="43800h" \
 | jq -r '.data.certificate' > intermediate.cert.pem</code>

#### sign the certificate back to vault
vault write pki_int/intermediate/set-signed certificate=@intermediate.cert.pem

#### create a role to generate certificates
<code>vault write pki_int/roles/vault-local \
 allowed_domains="local.at" \
 allow_subdomains=true \
 max_ttl="720h"</code>

#### generate the certificates
vault write pki_int/issue/vault-local common_name="vault.local.at"

write the different certificates to different files

vault_ca_chain.pem --> key is ca_chain
vault_ca.pem --> key is certificate
vault_issuing_ca.pem --> key is issuing_ca
vault_privkey.pem --> key is private_key

#### create a role for cert authentication
vault write pki_int/roles/vault-cert allow_any_name=true max_ttl="720h" generate_lease=true

#### generate the policy
use the vault-cert.hcl for it

then:
vault policy write vault-cert vault-cert.hcl

#### write the certificates
vault write -format=json pki_int/issue/vault-cert common_name="vault-cert" | tee >(jq -r .data.certificate > vault_ca.pem) >(jq -r .data.issuing_ca > vault_issuing_ca.pem) >(jq -r .data.private_key > vault_privkey.pem)


#### enable authentication with cert
vault auth enable cert

#### write the certificate to auth
vault write auth/cert/certs/vault-cert display_name=vault_ca.pem policies=vault-cert certificate=@vault_ca.pem

#### login (does not work without a public dns)
vault login -method=cert -client-cert=vault_ca.pem -client-key=vault_privkey.pem name=vault-cert

#### test secrets (does not work without a public dns)
vault kv get -format=json bstv2/data/secret/ | jq
vault kv get -format=json bstv1/secret/ | jq
vault kv get -format=json outside_cert/secret/ | jq



# K8s Integration

The agent sidecar injector will be used: https://www.vaultproject.io/docs/platform/k8s/injector

## Installation (with helm)

I had to add a microk8s addon (host-access) to access vault on my host:
microk8s enable host-access

This will add the ip 10.0.1.1 as accessible from the cluster

### Install release

helm repo add hashicorp https://helm.releases.hashicorp.com<br/>
helm repo update<br/>
helm install vault hashicorp/vault \
--set "injector.externalVaultAddr=http://10.0.1.1:8200" -n vault --create-namespace

### describe the serviceaccount to see the mountable secret

kubectl describe sa vault -n vault

### Now export the vault-secret name to a variable:

VAULT_HELM_SECRET_NAME=$(kubectl get secrets -n vault --output=json | jq -r '.items[].metadata | select(.name|startswith("vault-token-")).name')


### describe the secret
kubectl describe secret $VAULT_HELM_SECRET_NAME -n vault


### enable kubernetes as auth method
vault auth enable kubernetes

### get the token from the secret
TOKEN_REVIEW_JWT=$(kubectl get secret $VAULT_HELM_SECRET_NAME -n vault --output='go-template={{ .data.token }}' | base64 --decode)

### get the Kubernetes CA Certificate:
KUBE_CA_CERT=$(kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.certificate-authority-data}' | base64 --decode)


### get the Kubernetes Host URL
KUBE_HOST=$(kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.server}')

### configure the k8s auth method to use the service account token, the location of the host and its certificate:
vault write auth/kubernetes/config \
        token_reviewer_jwt="$TOKEN_REVIEW_JWT" \
        kubernetes_host="$KUBE_HOST" \
        kubernetes_ca_cert="$KUBE_CA_CERT"

### enable the secret store for k8s
vault secrets enable -path=k8s_secrets kv

I did this for my own since I wanted to store the secrets on an own path

### write a policy for read access to a secret
vault policy write devwebapp - <<EOF
path "k8s_secrets/devwebapp/config" {
  capabilities = ["read"]
}
EOF

### create a kubernetes authentication role with access to the policy:
vault write auth/kubernetes/role/devweb-app \
        bound_service_account_names=default \
        bound_service_account_namespaces=default \
        policies=devwebapp \
        token_max_ttl=60s \
        token_no_default_policy=true \
      ttl=30s

### create a pod with injected annotations
see the pod.yaml

It has these environment variables:
- name: VAULT_ADDR
  value: 'http://external-vault:8200'
- name: VAULT_TOKEN
  value: root

and also these annotations
vault.hashicorp.com/agent-inject: 'true'
vault.hashicorp.com/role: 'devweb-app'
vault.hashicorp.com/agent-inject-secret-credentials.txt: 'k8s_secrets/devwebapp/config'


so that means that we created a role in vault which stands for a serviceaccount and a namespace in k8s.
The token will be renewed every 30 seconds (ttl) and the secret will also renewed after some time.

For a new usage what will be needed:
* a secret in vault
* a policy to access the secret
* a serviceaccount in k8s
* a role in vault which binds the vault policy and the k8s serviceaccount and a k8s namespace
* the annotations in the deployment/pod