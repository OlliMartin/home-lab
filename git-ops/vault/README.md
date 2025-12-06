# Installation

We need to set up vault with a valid TLS certificate as a bootstrapping process. This action needs to be done manually.
The documented process follow [this](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-minikube-tls) tutorial.

1. Connect to one of the nodes, preferably master via ssh 
2. Follow the provided tutorial
3. Initialize, Unseal the vault. Make sure to copy the initial root token somewhere safe. It is required to [log in](https://developer.hashicorp.com/vault/docs/commands/login). 
4. Set up the [PKI](https://developer.hashicorp.com/vault/docs/secrets/pki/setup#setup).

Note: You can also refer to [this](https://developer.hashicorp.com/vault/tutorials/pki/pki-engine#step-1-generate-root-ca) full flow description.