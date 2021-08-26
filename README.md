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

---
**WARNING**
This can only be done on version 1 kv's if you try to access a version 2 kv to need to add data to the url like so:

curl -H 'X-Vault-Token: TOKEN' -X GET http://vault.local.at:8200/v1/bst/kv2/data/secret

---

