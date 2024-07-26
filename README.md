# glasskube-argocd-starter

Use this repository to get started with the glasskube-argo-gitops-stack easily.

## Getting Started

> At this very early stage, for simplicity we only support git repos that are accessible without any token.
> Additionally, some manual changes from you will be necessary, as described below.
> These shortcomings will be resolved in a future version.

**Prerequisites**

* You should have access to an empty kubernetes cluster in the form of a kubeconfig file on your machine.
* Glasskube CLI should be installed on your machine, but Glasskube should **not be bootstrapped** yet in the cluster.

**Installation**

1. Create a repo based on this starter template.
2. In `bootstrap/glasskube-application.yaml` and `bootstrap/glasskube/applicationset.yaml`, change the `repoURL` to your repository.
3. Run `glasskube bootstrap --template argocd --repository <your-repo>`.

As a result, your cluster will be powered with GitOps capabilities by ArgoCD, as well as package management features by
Glasskube. Argo manages itself, the Glasskube installation as well as Glasskube packages, which you can now manage
GitOps-style with this repo. 

**Temporary: Making your repo private**

TBA

## Managing your cluster

### Installing packages

### Updating packages

### Updating Glasskube

## Known Issues

## Summary
