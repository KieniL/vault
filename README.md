# command repo for vault <!-- omit in toc -->
- [Installation](#installation)
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


## Installation
The installation was done with: https://github.com/KieniL/software_installer




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
vault write -format=json pki/root/sign-intermediate \
 csr=@pki_intermediate.csr \
 format=pem_bundle ttl="43800h" \
 | jq -r '.data.certificate' > intermediate.cert.pem

### sign the certificate back to vault
vault write pki_int/intermediate/set-signed certificate=@intermediate.cert.pem