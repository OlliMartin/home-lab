```sh
sudo kubectl port-forward vault-0 -n vault 8202:8202
export VAULT_ADDR='http://127.0.0.1:8202'
export VAULT_TOKEN='root'
vault status

# Init the vault
vault operator init
# Unseal the vault using any 3 of the 5 keys generated during init
vault operator unseal $key1
vault operator unseal $key2
vault operator unseal $key3
vault status

# Login using the root token generated during init
vault login

# Enable PKI
vault secrets enable pki
vault secrets tune -max-lease-ttl=8760h pki

vault write pki/root/generate/internal \
  common_name=internal \
  ttl=8760h
  
vault write pki/config/urls \
  issuing_certificates="http://127.0.0.1:8200/v1/pki/ca" \
  crl_distribution_points="http://127.0.0.1:8200/v1/pki/crl"

vault write pki/issue/internal \
  common_name=www.internal
  
vault auth enable approle
vault write -f auth/approle/role/cert-role secret_id_ttl=768h \
  token_ttl=768h token_max_ttl=768h token_policies=pki
  
vault read auth/approle/role/cert-role/role-id # Note down role-id for cert-manager
vault write -f auth/approle/role/cert-role/secret-id

sudo kubectl create secret generic cert-manager-vault-approle \
  -n cert-manager --from-literal=secretId=$secretId
```