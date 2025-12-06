# Installation

We need to set up vault with a valid TLS certificate as a bootstrapping process. This action needs to be done manually.
The documented process follow [this](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-minikube-tls) tutorial.

1. Connect to one of the nodes, preferably master via ssh 
2. Follow the provided tutorial
3. Set up the [PKI](https://developer.hashicorp.com/vault/docs/secrets/pki/setup#setup).