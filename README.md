# glasskube-argocd-starter

Use this repository as a template to get started with the glasskube-argo-gitops-stack easily.

## Getting Started

> At this very early stage, for simplicity we only support GitHub repos that are accessible without any token (this applies only
> for the bootstrapping step – afterwards you can move the repo somewhere else and/or make it private).
> Additionally, one simple manual change from you will be necessary, as described below.
> These shortcomings will be resolved in a future version.

**Prerequisites**

* Access to an empty Kubernetes cluster
* Glasskube CLI should be installed on your machine, but Glasskube should **not be bootstrapped** yet in the cluster.

**Installation**

1. Create a public GitHub repository based on this starter template. You can move it and/or make it private afterwards.
2. In `bootstrap/glasskube-application.yaml` and `bootstrap/glasskube/applicationset.yaml`, change the `repoURL` to your repository.
3. Run `glasskube bootstrap gitops --repository <your-repo>`.

As a result, your cluster will be powered with GitOps capabilities by ArgoCD, as well as package management features by
Glasskube. Argo manages itself, the Glasskube installation, as well as Glasskube packages – all of which you can now manage
GitOps-style with this repo. 

Run `glasskube serve` to open the Glasskube UI and either open the ArgoCD UI there, or with the command `glasskube open argo-cd` –
but of course you can also use the Argo CLI.
Follow the [ArgoCD docs](https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli) to get and reset the password to log in.

**Temporary: Making your repo private**

When the installation has succeeded you are free to make your source repository private. However, make sure to [configure
the repository correctly](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/) in ArgoCD via UI or CLI, such that Argo can still access this repo. 

## Managing your cluster

Both CLI and UI offer features to manage your cluster following GitOps best practices: The relevant CLI commands offer
the flags `--dry-run` and `-o yaml`. The UI, when installed with the above `glasskube bootstrap gitops` command, 
will also output the `yaml` objects which you can copy to use in your git repo, instead of applying your changes directly. 

### Installing packages

To install a clusterpackage, e.g. `cert-manager`, use the `install` command like this:

```shell
glasskube install cert-manager --dry-run -o yaml --yes > cert-manager.yaml
```

Instead of directly installing the clusterpackage, this will write the `ClusterPackage` custom resource to the `cert-manager.yaml` file, 
which can now be put into a new directory `packages/cert-manager/` in the git repository. 
Once pushed to the repo, ArgoCD will pick up the changes after at most 5 minutes, create the ArgoCD `Application` wrapping 
the Glasskube `ClusterPackage`. After that, the Glasskube operator will pick up the `ClusterPackage` and finally install it in the cluster.

Similarly, when using the Glasskube UI, one can generate the `ClusterPackage` resource by using the "Show YAML" button 
on the page of the clusterpackage.

### Updating packages

There are two options handling package version updates:
* Using the `glasskube update --dry-run -o yaml` command, or the corresponding button on the Glasskube UI.
* Integrating [renovate](https://github.com/renovatebot/renovate) into the cluster.

The first option follows the same approach as the previously shown package installation, and will be omitted here. The
downside of doing it that way, is that someone has to manually execute the command, even though checking for updates and
preparing the updates to the git repository as an automatable task. Renovate is a tool allowing for exactly that kind of
task, as explained in the following.

#### Proof of Concept: Integrating with Renovate

Renovate Glasskube Support is still work in progress (see [renovatebot/renovate#29322](https://github.com/renovatebot/renovate/issues/29322)),
but we will show a proof of concept with the already available datasource/versioning part.

Therefore, install the [Renovate Github App](https://github.com/apps/renovate) and enable it for your GitOps repo.

The renovate configuration of this starter repo (`renovate.json`) contains a `regexManager`, looking for all appearances of

```yaml
packageInfo:
  name: <depName>
  version: <currentValue>
```

in all the repo's yaml files. `depName` and `currentValue` will be used by renovate to extract the current version of this (cluster-)package.

The regex-based approach has some limitations (see below) which will be resolved with the custom Glasskube Renovate manager.
However, we can show that on a general level, Glasskube packages can be updated successfully with this: As soon as one of
the installed Glasskube packages uses an outdated version, renovate will open a Pull Request to update to the latest version,
which you only need to approve and merge.

One issue with this regex-based approach is, that `name` and `version` have to appear in that order, even though schematically it would also be correct the other way around.

Another limitation of the current renovate integration is, that it works only with packages of the [public Glasskube package repo](https://github.com/glasskube/packages).

Last but not least, without a dedicated glasskube manager inside renovate, renovate will not be aware of dependencies. 
That means, it will simply always try to update to the latest version, instead of checking whether an update to that version is 
actually allowed in the used cluster. As a consequence, this could lead to somebody installing a version that is not allowed 
because of dependency restrictions, however, the package operator will not actually install it. 
The package status would be set to "Failed" with an error message indicating the dependency conflict, 
but the previously installed version of the package would not be touched/destroyed. In such a case, you would have to manually
intervene and roll back to the previously used package version. 

### Uninstalling packages

To uninstall a package or a clusterpackage, simply remove the custom resource from the git repository. 

### Updating Glasskube

When a new Glasskube version is available, the manifests have to be updated. Run

```shell
glasskube bootstrap --dry-run -o yaml > bootstrap/glasskube/manifests.yaml
```

to update the Glasskube manifests in your git repo. After reviewing and merging those changes the update will be picked up
by ArgoCD.

## Repository Structure

Initially, this repository will come with
* a `bootstrap` directory containing the initial/parent Argo `Application`, and the necessary Glasskube manifests
* a `packages` directory containing the `argo-cd` cluster package. 
* the renovate configuration in `renovate.json`. 

Glasskube custom resources will only be picked up by Argo when being put inside the `packages` directory. Please do not
delete/uninstall the `argo-cd` package, as this will also remove everything else!

## Upcoming Features

### Private Repo Support

We are aware that GitOps repositories should not be public, but for simplicity we omitted this feature in the first version
of the new gitops-bootstrap command. Supporting private repos with authentication of course has high priority for the upcoming releases. 
We will also replace the `repoURL` automatically, such that you don't need to this step manually when setting up the repo.

### Improved Renovate Integration

As described above, the renovate integration currently is regex-based, and it does not consider dependencies yet.
However, we don't see these shortcomings as a blocker and recommend to try out the renovate integration in the Glasskube/Argo Gitops setup.

### Improved Dependency Resolution

Installing packages with dependencies is not 100% GitOps-compatible yet, as the dependencies will be created by the operator.
Consider this: To install a clusterpackage `P` that has a dependency on `D`, one would do `glasskube install P --dry-run -o yaml`, which
would output the clusterpackage custom resource for `P`. However, the dependency `D` will only be resolved at reconciliation time by
the package operator, and will therefore not be represented in the git repository at all. A temporary workaround would be to have a closer look
at the output of the `install` command, which also shows the dependencies which will be installed and in which version. One could then
manually add the required packages custom resources to the git repo as well. However, this will be tackled in a future version to make the
user experience better, see [glasskube/glasskube#430](https://github.com/glasskube/glasskube/issues/430).

## Summary

With this template repository and guide we show how Glasskube can easily be set up in a ArgoCD powered Gitops environment, 
and how efficient package management is possible with this stack.

This is a first concept with some minor shortcomings but we will continue to improve gitops support. 

### Feedback

We love feedback! Whether you are just starting out or you are a seasoned professional, we'd like to hear your thoughts, inputs and questions
regarding this starter template and corresponding guide here, in the [glasskube/glasskube repo](https://github.com/glasskube/glasskube) or on
our [Discord](https://discord.gg/SxH6KUCGH7). Thanks!
