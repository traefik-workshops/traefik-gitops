# Traefik GitOps with Flux

GitOps makes configuration management seamless by creating a single source of truth for configuration changes, so changes can be transparent, validated, and low-risk. Check out this article for step-by-step instructions on how to deploy multiple Traefik instances on two clusters with Flux using a GitOps approach.

## GitOps Principles
 - **Declarative** - the desired stated of the infrastrucutre is expressed declaratively.
 - **Versioned** and immutable - the desired state is stored in a source of truth that enforces immutability.
 - **Pulled Automatically** from a source of truth *a git repo* - the changes are pulled by Controllers and applied on a cluster.
 - **Continuously Reconciled** - the agents are continuously observing the desired state and attempt to apply the desired state.

For more details visit the tutorial entitled:  [How to Deploy Traefik using Flux and GitOps Principles](https://traefik.io/blog/deploy-traefik-proxy-using-flux-and-gitops/)