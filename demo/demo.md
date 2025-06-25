# Demo Script

The demo is planned and organized in a way that makes it easy to follow in a local environment. Of course, it can also be executed against a cloud-based environment.

*Note 1:  It is expected that the workstation already has **kubectl** and **helm** installed. If not, they could be installed by following the steps outlined here for **kubectl** (<https://kubernetes.io/docs/tasks/tools/#kubectl>) and **helm** (<https://github.com/helm/helm/releases>).*

*Note 2: It is expected that on the workstation there is a context defined for the target cluster and it is set as default.*

## Requirements

It is expected that there is a Load Balancer (assumed **MetalLB**), a local git-based solution (assumed **Gitea**), and a CI/CD solution (assumed **Jenkins**). In addition, a read-write access to a container registry (assumed **Docker Hub**) is expected.

Detailed explanation on how to prepare an evironment like the one assumed and used, check the **Preparation** document, available [here](../preparation/preparation.md).

## Demo Steps

Let's assume that we are working in a folder named **demo**. All paths that will follow are relative to this folder.

In addition, make sure you cloned the two imported repositories respectively to **demo/gitops-app** and **demo/gitops-app-infra**.

### Without GitOps (manual deployment)

Let's see how the things happen when we do not have GitOps implemented and we are using manual but declarative approach to deploy our application.

We will put all the specific implementations like pipelines aside and will use just a single YAML manifest to deploy an application.

```bash
kubectl apply -f gitops-app/manifests/application.yaml
```

Now, check the resulting objects in the default namespace

```bash
kubectl get all
```

We can even check the application by opening the respective URL in a browser.

Now, what will happen if we change something? For example, the number of replicas

```bash
kubectl scale --replicas=5 deployment/gitops-app
```

If we check, will see that the change is there

```bash
kubectl get all
```

We can wait for an hour or two but nothing will happen. The cluster won't change anything here alone, just by itself. And this is by design.

Should we want to return to the **desired state** (declared by the YAML manifest), we can either execute another scale command, or re-apply the manifests.

In a similar fashion, what will happen if we delete an object, for example, the deployment?

Nothing, the cluster will "accept" our will and that it is.

Let's test this as well

```bash
kubectl delete deployment/gitops-app
```

And bam, the deployment, the replicaset, and the pods are gone, just like that

```bash
kubectl get all
```

Again, the cluster won't do anything on its own. Should we want to return to the **desired state** (declared by the YAML manifest), we must "refresh the cluster's memory" by applying the manifests again.

Let's do it

```bash
kubectl apply -f gitops-app/manifests/application.yaml
```

If we check, we will see that everything is back the way it used to be

```bash
kubectl get all
```

In fact, with our actions of **observing** the **actual state**, **comparing** it with the **desired state**, and refreshing the cluster's memory, we implemented manually a reconciliation loop which is part of what GitOps (plus other nice stuff) brings on the table.

Don't forget to remove the application

```bash
kubectl delete -f gitops-app/manifests/application.yaml
```

### Without GitOps (CI/CD pipeline)

Let's see how the things happen when we do not have GitOps implemented and we are using a CI/CD pipeline to deploy our application. 

Now, we should refer to a tool like Jenkins.

Go to Jenkins UI and start the **pipeline-cicd** pipeline.

After a while our application should be deployed

```bash
kubectl get pods,svc
```

Nice. Now, let's do a change. For example, add some text to the **fff** file.

Then stage, commit and push the changes to the repository

```bash
git add .
```

```bash
git commit -m 'application change 1'
```

```bash
git push
```

Now, return to Jenkins UI and start the **pipeline-cicd** pipeline again.

*Note that we are starting it manually for the sake of simplicity. In the real life it would be configured to trigger automatically via a webhook or by polling the repository periodically for changes.*

After a while, the new version of our application will be deployed to the cluster.

Now, if we do a direct change in the running configuration of the application, there won't be anything that will act and rever it.

Let's go ahead and prove this by changing, for example, the number of replicas to 5

```bash
kubectl scale deployment gitops-app --replicas=5
```

Check that the changes are there

```bash
kubectl get pods,svc
```

Again, the cluster won't do anything on its own to revert our changes because it accepts that if we changed something, it is because we are knowing what we are doing. ;)

Let's clean up by removing our application

```bash
kubectl delete deployment gitops-app
```

```bash
kubectl delete service gitops-app-svc
```

Now, we are ready to move forward and dive (at least the tip of our toes) into the world of GitOps.

### FluxCD

#### Installing the CLI

On Linux

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

Alternatively, or on other OSes, download the appropriate artefact from <https://github.com/fluxcd/flux2/releases>

Set the command completion for bash

```bash
. <(flux completion bash)
```

For other OSes and shells, check the documentation.

Once we have it, we can check if all the requirements are met

```bash
flux check --pre
```

#### Bootstrap with Custom Namespace

Go to the Gitea instance and create an empty repository **gitops-flux**.

```bash
flux bootstrap git --allow-insecure-http --token-auth --url=http://GITEA-IP:3000/GITEA-USER/gitops-flux --branch=main --path=clusters/dingo --namespace=gitops-flux --force
```

*You must substitute **GITEA-IP** and **GITEA-USER** with your values.*

Check the newly added objects

```bash
kubectl get all -n gitops-flux
```

Of course, we can try a few other commands, both with **flux** and **kubectl**, to explore our new **FluxCD** deployment.

```bash
flux get kustomizations --namespace=gitops-flux
```

```bash
kubectl get Kustomizations -n gitops-flux
```

```bash
flux get sources git --namespace=gitops-flux
```

```bash
kubectl get GitRepository -n gitops-flux
```

```bash
flux tree kustomization gitops-flux --namespace=gitops-flux
```

```bash
flux trace kustomization gitops-flux --namespace=gitops-flux
```

```bash
flux logs --flux-namespace=gitops-flux --namespace=gitops-flux
```

```bash
flux stats --namespace=gitops-flux
```

Note that a custom namespace is used - ***gitops-flux*** instead of the usual one ***flux-system***.

#### App Infra Repository

Don't forget to set the **user.email** and **user.name** settings on the **git** client to match the ones used in the local **Gitea**, if not set already.

Then clone the **gitops-flux** repository in our working (**demo**) folder

```bash
git clone http://GITEA-IP:3000/GITEA-USER/gitops-flux
```

*You must substitute **GITEA-IP** and **GITEA-USER** with your values.*

Ener the folder

```bash
cd gitops-flux
```

And generate a manifest for the application's infrastructure repository

```bash
flux create source git gitops-app --namespace=gitops-flux --url=http://GITEA-IP:3000/GITEA-USER/gitops-app-infra --branch=main --interval=1m --export > ./clusters/dingo/gitops-app-source.yaml
```

*You must substitute **GITEA-IP** and **GITEA-USER** with your values.*

Check the resulting manifest

```bash
cat ./clusters/dingo/gitops-app-source.yaml
```

Now check, add, commit and push the manifest

```bash
git status
```

```bash
git add .
```

```bash
git commit -m "Add gitops-app GitRepository"
```

```bash
git push
```

If we check immediately, we won't see any change

```bash
flux get sources git --namespace=gitops-flux
```

```bash
kubectl get GitRepository -n gitops-flux
```

After a while, the new git repository will appear.

#### App with Kustomization

Now, we are ready to register the actual infrastructure that will be used to spin up the application.

Let's create a **kustomization** resource manifest for **FluxCD** to manage

```bash
flux create kustomization gitops-app-prd --namespace=gitops-flux --target-namespace=prd --source=gitops-app --path="./kustomize/overlays/prd" --prune=true --wait=true --interval=2m --retry-interval=1m --health-check-timeout=1m --export > ./clusters/dingo/gitops-app-kustomization-prd.yaml
```

Note that here we are reusing (***--source=gitops-app***) the Git Repository we registered earlier.

Check the resulting manifest

```bash
cat ./clusters/dingo/gitops-app-kustomization-prd.yaml
```

Now check, add, commit and push the manifest

```bash
tree .
```

```bash
git status
```

```bash
git add .
```

```bash
git commit -m "Add gitops-app Kustomization"
```

```bash
git push
```

```bash
flux get kustomizations --namespace=gitops-flux --watch
```

```bash
kubectl -n prd get all
```

After all is settled, we can do a few experiments to see the GitOps reconciliation in action.

Remove/delete the deployment

```bash
kubectl delete -n prd deployment gitops-app
```

Or change (the number of replicas) a monitored resource 

```bash
kubectl scale -n prd --replicas=5 deployment/gitops-app
```

And observe

```bash
kubectl -n prd get all
```

```bash
kubectl -n prd get pods --watch
```

After the ***--interval=2m*** passes, the changes will be reverted.

Between the time of change and the revert may take less time, it depends on when the last check was executed.

We can explore what is happening

```bash
flux get kustomizations --namespace=gitops-flux
```

```bash
kubectl describe Kustomizations -n gitops-flux gitops-app-prd
```

```bash
flux logs --flux-namespace=gitops-flux --namespace=gitops-flux
```

#### App with just YAML manifests

Prepare the target namespace

```bash
kubectl create ns plain
```

Let's create a **kustomization** (the plain YAML manifests are managed via Kustomization) resource manifest for **FluxCD** to manage

```bash
flux create kustomization gitops-app-yaml --namespace=gitops-flux --target-namespace=plain --source=gitops-app --path="./manifests" --prune=true --wait=true --interval=2m --retry-interval=1m --health-check-timeout=3m --export > ./clusters/dingo/gitops-app-plain.yaml
```

Note that here we are reusing (***--source=gitops-app***) the Git Repository we registered earlier.

Check the resulting manifest

```bash
cat ./clusters/dingo/gitops-app-plain.yaml
```

Now check, add, commit and push the manifest

```bash
tree .
```

```bash
git status
```

```bash
git add .
```

```bash
git commit -m "Add gitops-app plain yaml (kustomization)"
```

```bash
git push
```

After the ***--interval=2m*** passes, the changes will be reverted.

Between the time of change and the revert may take less time, it depends on when the last check was executed.

We can explore what is happening

```bash
flux get kustomizations --namespace=gitops-flux --watch
```

```bash
flux get all --namespace=gitops-flux
```

```bash
kubectl get all -n plain
```

#### App with Helm Chart from Git Source

We will reuse again (***--source=GitRepository/gitops-app***) the Git Repository we registered earlier.

Let's create a **helmrelease** resource manifest for **FluxCD** to manage

```bash
flux create helmrelease gitops-app-helm --namespace=gitops-flux --target-namespace=gitops-app-helm --create-target-namespace=true --source=GitRepository/gitops-app --chart="./helm/gitops-app" --interval=2m --reconcile-strategy=Revision --export > ./clusters/dingo/gitops-app-helm.yaml
```

Check the resulting manifest

```bash
cat ./clusters/dingo/gitops-app-helm.yaml
```

Now check, add, commit and push the manifest

```bash
tree .
```

```bash
git status
```

```bash
git add .
```

```bash
git commit -m "Add gitops-app Helm Chart"
```

```bash
git push
```

We can explore what is happening

```bash
flux get helmreleases --namespace=gitops-flux --watch
```

```bash
kubectl -n gitops-app-helm get all
```

```bash
flux get all --namespace=gitops-flux
```

#### Capacitor UI

Should you want to have an UI, you can install one of the few available. Here, we will go over the steps for installing **Capacitor**.

Make sure you are back in our working directory (**demo**).

Download the two installation manifests

```bash
curl https://raw.githubusercontent.com/gimlet-io/capacitor/main/deploy/k8s/rbac.yaml -o capacitor-rbac.yaml
```

```bash
curl https://raw.githubusercontent.com/gimlet-io/capacitor/main/deploy/k8s/manifest.yaml -o capacitor-manifest.yaml
```

Adjust the namespace in the two files to match yours (for example, ***gitops-flux**).

Send the files to the cluster:

```bash
kubectl apply -f .\capacitor-rbac.yaml
```

```bash
kubectl apply -f .\capacitor-manifest.yaml
```

By default, the service is operating in ClusterIP mode. Sure, we can use port forwarding to access it, but in our case, NodePort will be more suitable/easier.

Patch the service to NodePort

```bash
kubectl patch svc capacitor -n gitops-flux -p '{"spec": {"type": "NodePort"}}'
```

Get the list of services

```bash
kubectl get svc -n gitops-flux
```

Visit the UI and explore it.

Suspend a Source or Kustomization (in the UI) and check on the command line

```bash
flux get kustomizations --namespace=gitops-flux
```

```bash
flux get sources git --namespace=gitops-flux
```

If you like, do a change (corresponding to the suspension) and observe if anything will happen or not.

#### Complete Removal

Should you want, you could remove completely the **FluxCD** installation, together with all resources managed by it.

First, initiate the actual removal

```bash
flux uninstall --namespace=gitops-flux
```

Then remove any leftovers (if any and depending on what you tested)

```bash
kubectl delete ns prd
```

```bash
kubectl delete namespace plain
```

```bash
kubectl delete namespace gitops-app-helm
```

### ArgoCD

#### Installation

Create the namespace

```bash
kubectl create namespace argocd
```

And install it (together with the UI)

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Monitor the progress with

```bash
kubectl get pods -n argocd --watch
```

Once done, you can patch the service to NodePort or LoadBalancer for easier access

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
```

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Then check the services

```bash
kubectl get svc -n argocd
```

#### CLI Installation

On Linux you can download the latest version of the CLI

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
```

Or the latest stable version with

```bash
VERSION=$(curl -L -s https://raw.githubusercontent.com/argoproj/argo-cd/stable/VERSION)
```

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/download/v$VERSION/argocd-linux-amd64
```

Whichever version you downloaded, you can install it with

```bash
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
```

And then remove the leftovers

```bash
rm argocd-linux-amd64
```

For other OSes, download the appropriate binary from here: <https://github.com/argoproj/argo-cd/releases/latest>

Once done, you can check the CLI is working

```bash
argocd version --client
```

Should you want command completion, you can enable it with

```bash
. <(argocd completion bash)
```

#### Admin Credentials

Get the initial **admin** password

```bash
argocd admin initial-password -n argocd
```

And use it to login

```bash
argocd login ARGOCD-IP
```

*You must substitute **ARGOCD-IP** with your value.*

Now we can check both the server and CLI versions

```bash
argocd version 
```

Next we can change the **admin** password

```bash
argocd account update-password
```

You can use the new credentials to log into the UI and explore.

#### Clusters

To check the list of registered clusters, execute

```bash
argocd cluster list
```

*The next few lines is for informative purposes only. Feel free to skip them.*

Should we want to add a cluster (besides the **in-cluster**), we can start from the available contexts

```bash
kubectl config get-contexts -o name
```

And then add one of them, for example

```bash
argocd cluster add kubernetes-admin@kubernetes
```

#### Project and Applications

We can ask for the list of defined projects

```bash
argocd proj list
```

On a new installation, only one should appear - the **default** project.

In the same manner, we can ask for the list of applications

```bash
argocd app list
```

None should be returned (for now).

#### Create Application (manifests)

Let's register an application for ArgoCD to manage.

Of course, we can do it in the UI, but on the command line is more interesting.

Execute the following command

```bash
argocd app create gitops-app --repo http://GITEA-IP:3000/GITEA-USER/gitops-app-infra --path manifests --dest-server https://kubernetes.default.svc --dest-namespace default --label purpose=gitops-demo --label apptype=yaml
```

*You must substitute **GITEA-IP** and **GITEA-USER** with your values.*

(Skip this one for now) Alternatively, we can enable auto sync and self-heal during creation (including namespace autocreation)

```bash
argocd app create gitops-app-yaml --repo http://GITEA-IP:3000/GITEA-USER/gitops-app-infra --path manifests --dest-server https://kubernetes.default.svc --dest-namespace argo-app-yaml --sync-policy auto --self-heal --sync-option CreateNamespace=true --label purpose=gitops-demo --label apptype=yaml
```

Note that if we want to use annotations, the above (the last part of the command) will change to ***--annotations purpose=gitops-demo,apptype=yaml***

However, as with Kubernetes, annotations and labels serve different purposes. Annotations are for metadata, while labels allows us to use them for filtering/narrowing down a list of resources.

Ask for the list of registered applications

```bash
argocd app list
```

And then for details about the **gitops-app**

```bash
argocd app get gitops-app
```

Explore the information.

Pay attention to this line URL: <https://SOME-IP/applications/gitops-app>

Open a browser and visit the above.

Use the admin credentials.

In the UI, we have the same - the application is out of sync.

We can sync it either from the UI or by executing the following command

```bash
argocd app sync gitops-app
```

We can explore the resources of the application (it is in the default namespace)

```bash
kubectl get all
```

And delete one of them - the deployment

```bash
kubectl delete deployment gitops-app
```

After a while the deployment and everything managed by it will be gone

```bash
kubectl get all
```

Now, because our application is not configured with automatic sync, it won't heal itself.

We can enable the automatic sync again from the UI (**Application** > ***particular application*** > **Details** > **Sync Policy** and enable **Self Heal**) or by executing this

```bash
argocd app set gitops-app --sync-policy auto --self-heal
```

Even with this, when there is a change in the repository, it may take up to ***3 minutes***. More information:

* <https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/#automated-sync-semantics>

* <https://argo-cd.readthedocs.io/en/stable/faq/#how-often-does-argo-cd-check-for-changes-to-my-git-or-helm-repository>

Self-heal, when configuration drift is detected, will take considerably less time - around ***5 seconds***. Details here: <https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/#automated-sync-semantics>

Now, besides the information available in the UI, we can use a set of commands to explore our application and its state

```bash
argocd app logs gitops-app
```

```bash
argocd app history gitops-app
```

```bash
argocd app get gitops-app --refresh
```

```bash
argocd app get gitops-app --show-operation
```

```bash
argocd app get gitops-app --output tree
```

```bash
argocd app get gitops-app --show-operation --output tree
```

You can run the **GitOps** pipeline (in **Jenkins**) and observe **ArgoCD**'s reaction.

After up to ***3 minutes*** **ArgoCD** will sync the application with the current state in Git.

Of course, if we do not want to wait, we can execute the synchronization manually either from the UI or on the command line

```bash
argocd app sync gitops-app
```

Check the application's history (if you triggered the **GitOps** pipeline)

```bash
argocd app history gitops-app
```

#### Create Application (helm)

To create a **Helm**-based application on the command line, we can execute the following

```bash
argocd app create gitops-app-helm --repo http://GITEA-IP:3000/GITEA-USER/gitops-app-infra --path helm/gitops-app --dest-namespace argo-app-helm --dest-server https://kubernetes.default.svc --sync-policy auto --self-heal --sync-option CreateNamespace=true --helm-set replicaCount=2 --label purpose=gitops-demo --label apptype=helm
```

*You must substitute **GITEA-IP** and **GITEA-USER** with your values.*

Now, either use the UI or the ususal commands that you are already aware of, to explore our application and its state.

Should you want, you can trigger the **GitOps** pipeline again and observe.

#### Create Application (kustomize)

To create a **kustomize**-based application on the command line, we can execute the following

```bash
argocd app create gitops-app-kust --repo http://GITEA-IP:3000/GITEA-USER/gitops-app-infra --path kustomize/overlays/stg --dest-namespace argo-app-kust --dest-server https://kubernetes.default.svc --sync-policy auto --self-heal --sync-option CreateNamespace=true --kustomize-namespace argo-app-kust --label purpose=gitops-demo --label apptype=kust
```

*You must substitute **GITEA-IP** and **GITEA-USER** with your values.*

Now, either use the UI or the ususal commands that you are already aware of, to explore our application and its state.

Should you want, you can trigger the **GitOps** pipeline again and observe.

#### Complete Removal

First, we can explore and delete the applications either one-by-one or in batches.

Get a list of all applications

```bash
argocd app list
```

Or filter the list by label

```bash
argocd app list -l apptype=yaml
```

(Skip this) Delete single application

```bash
argocd app delete gitops-app
```

(Skip this) Or delete multiple at once

```bash
argocd app delete gitops-app-helm gitops-app-kust gitops-app-yaml
```

(Use this) Or delete all that have particular label

```bash
argocd app delete -l purpose=gitops-demo
```

Finally, we can uninstall the ArgoCD suite

```bash
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

And remove the namespace as well

```bash
kubectl delete namespace argocd
```
