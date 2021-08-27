# command repo for vault <!-- omit in toc -->
- [Installation](#installation)
- [Further documentation](#further-documentation)
- [Notes](#notes)
- [enable a a secret type](#enable-a-a-secret-type)
- [disable a a secret type](#disable-a-a-secret-type)
- [input a secret](#input-a-secret)
- [vault kv put kv/secret username=demo pw=pw](#vault-kv-put-kvsecret-usernamedemo-pwpw)
- [delete a secret](#delete-a-secret)
- [list all secrets in an directory](#list-all-secrets-in-an-directory)
- [get a secret](#get-a-secret)
  - [format the get to display it in json](#format-the-get-to-display-it-in-json)
- [Get Secret by API](#get-secret-by-api)
- [Create Secret by API](#create-secret-by-api)
- [Delete Secret by API](#delete-secret-by-api)
- [enable pki](#enable-pki)
- [tune a secret](#tune-a-secret)
- [write rootcertificate](#write-rootcertificate)
    - [Then another secret path is enabled:](#then-another-secret-path-is-enabled)
    - [Then a csr is created:](#then-a-csr-is-created)
    - [Then you can sign the csr to create the cert:](#then-you-can-sign-the-csr-to-create-the-cert)
    - [sign the certificate back to vault](#sign-the-certificate-back-to-vault)
    - [create a role to generate certificates](#create-a-role-to-generate-certificates)
    - [generate the certificates](#generate-the-certificates)
    - [create a role for cert authentication](#create-a-role-for-cert-authentication)
    - [generate the policy](#generate-the-policy)
    - [write the certificates](#write-the-certificates)
    - [enable authentication with cert](#enable-authentication-with-cert)
    - [write the certificate to auth](#write-the-certificate-to-auth)
    - [login (not tested since I don't use tls due to local machine)](#login-not-tested-since-i-dont-use-tls-due-to-local-machine)
    - [test secrets (not tested since I don't use tls due to local machine)](#test-secrets-not-tested-since-i-dont-use-tls-due-to-local-machine)


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

#### login (not tested since I don't use tls due to local machine)
vault login -method=cert -client-cert=vault_ca.pem -client-key=vault_privkey.pem name=vault-cert


#### test secrets (not tested since I don't use tls due to local machine)
vault kv get -format=json bstv2/data/secret/ | jq
vault kv get -format=json bstv1/secret/ | jq
vault kv get -format=json outside_cert/secret/ | jq