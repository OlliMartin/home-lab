Note: Make sure to increase the liveness probe; Otherwise the vault pod will keep restarting while doing this setup.

### Port Foward & Initial CLI setup

``` sh
sudo kubectl port-forward vault-0 -n vault 8202:8202
export VAULT_ADDR='http://127.0.0.1:8202'
export VAULT_TOKEN='root'

# Check the status -> Vault should be sealed
vault status
```

### Init the vault.

``` sh
# Init the vault
vault operator init

# Unseal the vault using any 3 of the 5 keys generated during init
vault operator unseal $key1
vault operator unseal $key2
vault operator unseal $key3

# Verify that Initialized is now true
vault status
```

Note down the initial root token. You cannot access it later anymore.
It is returned as part of the `vault oeprator init`

`Initial Root Token: $ROOT_TOKEN`

### Login to Vault UI

With the vault unsealed (standard operation mode), you can now log in to the Vault UI.
Login with the root token identified earlier.

### Enable PKI

``` sh
vault secrets enable pki
vault secrets tune -max-lease-ttl=87600h pki
```

### Create the Root CA

```sh
vault write pki/root/generate/internal \
  common_name="OMA-Root-CA" \
  issuer_name="oma-root-ca" \
  ttl=8760h
```

### Configure the URLs

Important: The URLs must be resolvable later!
Choose appropriate URLs that your end-devices can reach.

My vault instance will run under `vault.internal`.

``` sh
vault write pki/config/urls \
  issuing_certificates="https://vault.internal/v1/pki/ca" \
  crl_distribution_points="https://vault.internal/v1/pki/crl"
```

### Create the Intermediate CA

``` sh
# We need to mount it at a different path compared to the root CA.
vault secrets enable -path=pki_int pki
vault secrets tune -max-lease-ttl=43800h pki_int
```

### Generate Intermediate Certificate Signing Request

```sh
# Generate the CSR in the file system

vault write -format=json pki_int/intermediate/generate/internal \
  common_name="Internal Intermediate Authority" \
  issuer_name="internal-intermediate" \
  ttl=43800h | jq -r '.data.csr' > pki_intermediate.csr
  
vault write -format=json pki/root/sign-intermediate \
     issuer_ref="oma-root-ca" \
     csr=@pki_intermediate.csr \
     format=pem_bundle ttl="43800h" \
     | jq -r '.data.certificate' > intermediate.cert.pem
     
vault write pki_int/intermediate/set-signed certificate=@intermediate.cert.pem

vault write pki_int/config/urls \
  issuing_certificates="https://vault.internal/v1/pki_int/ca" \
  crl_distribution_points="https://vault.internal/v1/pki_int/crl"
```

### Confguring a role

Since I want to use the PKI exclusively for cert-manager, I create a role for it.

```sh
vault write pki_int/roles/cert-manager \
    allowed_domains=internal \
    allow_subdomains=true max_ttl=72h
```

### Enabling App Role

```sh
vault auth enable approle
vault write -f auth/approle/role/cert-role secret_id_ttl=24h \
  token_ttl=24h token_max_ttl=24h token_policies="pki pki_int"
  
vault read auth/approle/role/cert-role/role-id # Note down role-id for cert-manager
vault write -f auth/approle/role/cert-role/secret-id # We'll create a secret in kubernetes for that.

# Verify:
# Note how we issue against the /cert-manager endpoint, i.e. the role we created above.
vault write pki_int/issue/cert-manager common_name="test.internal" ttl="1h"
```

### Creating the Vault Policy

The policy can be created via the vault UI.
It must be attached to the role `cert-manager`, which can be found under:
- Vault UI
- Access
- Entities
- entity_[HEXCODE]
- Policies Tab

```sh
# Enable secrets engine
path "sys/mounts/*" {
  capabilities = [ "create", "read", "update", "delete", "list" ]
}

# List enabled secrets engine
path "sys/mounts" {
  capabilities = [ "read", "list" ]
}

# Work with pki secrets engine
path "pki*" {
  capabilities = [ "create", "read", "update", "delete", "list", "sudo", "patch" ]
}
```