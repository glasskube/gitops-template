# gitops-template

Use this repository as a template to get started with ArgoCD & Glasskube in minutes instead of hours.

## Getting Started

### Prerequisites

#### Access to an empty Kubernetes cluster

The easiest would be creating a new [Minikube](https://minikube.sigs.k8s.io/docs/start/) cluster with:

```shell
minikube start -p gitops
````
Glasskube should not yet be bootstrapped in that cluster

#### Install the Glasskube CLI

Make sure to have at least [Glasskube](https://glasskube.dev/docs/getting-started/install/) version 0.16.0 installed locally.
If you don't, you can simply run:

```shell
brew install glasskube/tap/glasskube
````

### Installation

#### 1. Use this repository as your GitOps template

Create a GitHub repository based on this starter template.

#### 2. Replace the placeholder `repoURL` in your GitOps repository

Replace the default value of `repoURL` to your repository url.

- Line 12 in: [`bootstrap/glasskube-application.yaml`](bootstrap/glasskube-application.yaml#L12)
- Lines 11, 16 and 26 in: [`bootstrap/glasskube/applicationset.yaml`](bootstrap/glasskube/applicationset.yaml#L11-L26)
- Commit and push changes to your target git repository.

#### 3. Bootstrap ArgoCD and Glasskube for your Kubernetes cluster

Make sure you are connected to the right cluster and execute:

```shell
glasskube bootstrap git --url <your-repo> --username <org-or-username> --token <your-token>
```

For public repositories you can omit `--username` and `--token`. 

For private repositories, make sure to [obtain a token from GitHub](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens), 
that has read access to the repository.

#### Result

As a result, your cluster will be powered with GitOps capabilities by ArgoCD, as well as package management features by
Glasskube. Argo manages itself, the Glasskube installation, as well as Glasskube packages – all of which you can now manage
GitOps-style with this repo. 

Run `glasskube serve` to open the Glasskube UI and either open the ArgoCD UI there, or with the command `glasskube open argo-cd` –
but of course you can also use the Argo CLI.
Follow the [ArgoCD docs](https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli) to get and reset the password to log in.

Note that it might take a couple of minutes for ArgoCD to start up, and for the initial GitOps sync to happen. 

In this template, for demonstration purposes we also install the `cloudnative-pg` and `kube-prometheus-stack` clusterpackage as well as a bookmarking 
application ([shiori](https://github.com/go-shiori/shiori)), which is making use of `cloudnative-pg`. 

## Managing your cluster

Both CLI and UI offer features to manage your cluster according to GitOps best practices: CLI commands include `--dry-run` and `-o yaml` flags. The UI, when installed with the above `glasskube bootstrap git` command, 
will also output the `yaml` objects which you can copy to use in your git repo, instead of applying your changes directly. 

### Installing packages

To install a `ClusterPackage`, e.g. `cert-manager`, use the `install` command like this:

```shell
glasskube install cert-manager --dry-run -o yaml --yes > cert-manager.yaml
```

Instead of directly installing the `ClusterPackage`, this command writes the `ClusterPackage` custom resource to the `cert-manager.yaml` file, 
which can now be put into a new directory `packages/cert-manager/` in the git repository. 
Once pushed to the repo, ArgoCD will pick up the changes after at most 5 minutes, create the ArgoCD `Application` wrapping 
the Glasskube `ClusterPackage`. After that, the Glasskube operator will pick up the `ClusterPackage` and finally install it in the cluster.

Similarly, when using the Glasskube UI, one can generate the `ClusterPackage` resource by using the "Show YAML" button on the page of the `ClusterPackage`.

### Updating packages

There are two options handling package version updates:
* Using the `glasskube update --dry-run -o yaml` command, or the corresponding button on the Glasskube UI.
* Integrating [renovate](https://github.com/renovatebot/renovate) into the cluster.

The first option follows the same approach as the previously shown package installation, and will be omitted here. The
downside of doing it that way, is that someone has to manually execute the command, even though checking for updates and
preparing the updates to the git repository as an automatable task. Renovate is a tool allowing for exactly that kind of
task, as explained in the following.

#### Integrating with Renovate

Glasskube integrates with Renovate in order to simplify package updates.

Therefore, install the [Renovate GitHub App](https://github.com/apps/renovate) and enable it for your GitOps repo.

The renovate configuration of this template repo (`renovate.json`) contains the `glasskube` manager, looking for all appearances
of Glasskube (`Cluster`-)`Package`s in all the repo's yaml files. 

As soon as one of the installed Glasskube packages uses an outdated version, renovate will open a Pull Request to update to the latest version,
which you only need to approve and merge.

Be aware that the Glasskube Renovate manager is not aware of the package dependencies in your cluster. As a consequence, this could lead to somebody installing a version that is not allowed 
because of dependency restrictions, however, the package operator will not actually install it. 
The package status would be set to "Failed" with an error message indicating the dependency conflict, 
but the previously installed version of the package would not be touched/destroyed. In such a case, you would have to manually
intervene and roll back to the previously used package version. 

### Uninstalling packages

To uninstall a package or a `ClusterPackage`, simply remove the custom resource from the git repository. 

### Updating Glasskube

When a new Glasskube version is available, the manifests have to be updated. Run

```shell
glasskube bootstrap --dry-run -o yaml --force > bootstrap/glasskube/glasskube.yaml
```

to update the Glasskube manifests in your git repo. After reviewing and merging those changes the update will be picked up
by ArgoCD. The `--force` flag is necessary for the command to continue manifest validation, even though failures occur. 

### Working with Apps

This template also contains a demo application: a bookmark manager called [shiori](https://github.com/go-shiori/shiori).
Its manifests are defined in `apps/shiori`, which is a pattern you can follow for your own custom applications.

In a minikube environment, two manual steps are required to access the application (for more information consult the 
[minikube docs](https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/)):
* Run `minikube addons enable ingress -p gitops`.
* Run `minikube ip -p gitops` and add the line `<your-IP> my-shiori.example` to your `/etc/hosts` file. 

After that you will be able to access the application via [http://my-shiori.example](http://my-shiori.example) in your browser. 
The default login credentials are `shiori` / `gopher` – for more information check the [shiori docs](https://github.com/go-shiori/shiori/tree/master/docs).

In general, you can use the `apps` directory to deploy such custom applications into your cluster. Any subdirectory will be
picked up by ArgoCD and grouped as an `Application`. 

### Monitoring with `kube-prometheus-stack`

This template also installs the `kube-prometheus-stack` clusterpackage, which is an easy way to get started with monitoring your cluster.
You can open Grafana with `glasskube open kube-prometheus-stack`. It does not come preconfigured in this example, but you 
can easily add a nice postgres dashboard and observe the metrics of the database while you are working with the bookmarking application.

#### Setting up a postgres dashboard

We are going to make use of the [cloudnativepg dashboard](https://grafana.com/grafana/dashboards/20417-cloudnativepg/).
Import it by opening the [dashboard-import page](http://localhost:8888/dashboard/import), pasting
[https://grafana.com/grafana/dashboards/20417-cloudnativepg/](https://grafana.com/grafana/dashboards/20417-cloudnativepg/)
into the first textfield, and pressing "Load". Use the "Prometheus" data source on the following screen and finish the import process.

![CloudNativePG dashboard](https://github.com/user-attachments/assets/d54dcefe-535c-4812-bd80-486558f6caa4)

Of course monitoring your experimental minikube cluster is a bit of an overkill, but this is simply to demonstrate how
these kind of cluster administration tasks can be integrated into this gitops stack. 

## Repository Structure

Initially, this repository will come with
* a `bootstrap` directory containing the initial/parent Argo `Application`, and the necessary Glasskube manifests
* a `packages` directory containing the `argo-cd` cluster package. 
* an `apps` directory containing your applications. 
* the renovate configuration in `renovate.json`. 

Glasskube custom resources will only be picked up by Argo when being put inside the `packages` directory. Please do not
delete/uninstall the `argo-cd` package, as this will also remove everything else!

Note that the parent application used to bootstrap (`bootstrap/glasskube-application.yaml`) will not be synced after the initial setup.
If you want to change something about it, you will have to change the application via argo directly. 

### Private Repositories

Your gitops repository can be public or private (recommended).

The given username and token are [stored in a secret](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#repositories) by the bootstrap command.
This secret is not commited the git repository. 

If you initially chose to use a public repository, or if you need to change the repo credentials (e.g. because the token expired), you can still do so by adding or updating this secret [manually via ArgoCD](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/).

## Upcoming Features

### Improved Dependency Resolution

Installing packages with dependencies is not 100% GitOps-compatible yet, as the dependencies will be created by the operator.
Consider this: To install a `ClusterPackage` `<P>` that has a dependency on `D`, one would do `glasskube install <P> --dry-run -o yaml`, which
would output the `ClusterPackage` custom resource for `<P>`. However, the dependency `D` will only be resolved at reconciliation time by
the package operator, and will therefore not be represented in the git repository at all. A temporary workaround would be to have a closer look
at the output of the `install` command, which also shows the dependencies which will be installed and in which version. One could then
manually add the required packages custom resources to the git repo as well. However, this will be tackled in a future version to make the
user experience better, see [glasskube/glasskube#430](https://github.com/glasskube/glasskube/issues/430).

## Summary

With this template repository and guide we show how Glasskube can easily be set up in a ArgoCD powered Gitops environment, 
and how efficient package management is possible with this stack. Additionally we install a web application to show how
custom applications can make use of the Gitops setup and Glasskube packages.

This is still in early stages and therefore has some minor shortcomings, but we will continue to improve GitOps support. 

### Feedback

We love feedback! Whether you are just starting out or you are a seasoned professional, we'd like to hear your thoughts, inputs and questions
regarding this starter template and corresponding guide here, in the [glasskube/glasskube repo](https://github.com/glasskube/glasskube) or on
our [Discord](https://discord.gg/SxH6KUCGH7). Thanks!
