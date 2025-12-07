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
vault write -f auth/approle/role/cert-role secret_id_ttl=24h \
  token_ttl=24h token_max_ttl=24h token_policies="pki pki_int"
  
vault read auth/approle/role/cert-role/role-id # Note down role-id for cert-manager
vault write -f auth/approle/role/cert-role/secret-id

sudo kubectl create secret generic cert-manager-vault-approle \
  -n cert-manager --from-literal=secretId=$secretId
  
vault write -field=certificate pki/root/generate/internal \
     common_name="internal" \
     issuer_name="root-2025" \
     ttl=87600h > root_2025_ca.crt
     
vault write pki/roles/2025-servers allow_any_name=true

vault write pki/config/urls \
     issuing_certificates="127.0.0.1:8200/v1/pki/ca" \
     crl_distribution_points="127.0.0.1:8200/v1/pki/crl"
     
vault secrets enable -path=pki_int pki 
vault secrets tune -max-lease-ttl=43800h pki_int

vault write -format=json pki_int/intermediate/generate/internal \
     common_name="Internal Intermediate Authority" \
     issuer_name="internal-intermediate" \
     | jq -r '.data.csr' > pki_intermediate.csr
     
vault write -format=json pki/root/sign-intermediate \
     issuer_ref="root-2025" \
     csr=@pki_intermediate.csr \
     format=pem_bundle ttl="43800h" \
     | jq -r '.data.certificate' > intermediate.cert.pem

vault write pki_int/intermediate/set-signed certificate=@intermediate.cert.pem

vault write pki_int/roles/internal-intermediate \
     issuer_ref="$(vault read -field=default pki_int/config/issuers)" \
     allowed_domains="internal" \
     allow_subdomains=true \
     max_ttl="720h"
     
# Verify:
vault write pki_int/issue/internal-intermediate common_name="test.internal" ttl="24h"
```



6fb8252a-b713-a2a4-10d8-b557173aec9f