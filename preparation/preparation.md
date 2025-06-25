# Preparation Script

The steps that follow will help you setup an environment identical to the one used during the demonstration. Of course, you can use something else, but then some adjustments should be made accordingly.

## Kubernetes Cluster

Create and start a Minikube-based Kubernetes cluster

```bash
minikube start --driver=hyperv --hyperv-virtual-switch='NAT vSwitch' --cpus=4 --memory=8196 -p gitops
```

*Change at least the **--driver** and the networking connectivity to match your setup.*

Should we want to check what other profiles exist, we can do it by executing

```bash
minikube profile list
```

Get the IP address of the instance

```bash
minikube -p gitops ip
```

Enable the MetalLB addon

```bash
minikube -p gitops addons enable metallb
```

Configure the MetalLB addon

```bash
minikube -p gitops addons configure metallb
```

Use the following values (or others that match your setup)

```
 -- Enter Load Balancer Start IP: 192.168.99.51
 -- Enter Load Balancer End IP: 192.168.99.59
```

Check that the current context of **kubectl** is set correctly

```bash
kubectl config get-contexts
```

## Gitea - Installation

Check what Helm repositories you currently have

```bash
helm repo list
```

And if the **Gitea** one is not listed, add it

```bash
helm repo add gitea-charts https://dl.gitea.com/charts/
```

Adjust the provided **gitea/values.yaml** file to match your setup and execute

```bash
helm install gitea -n gitea gitea-charts/gitea --create-namespace -f gitea/values.yaml
```

Watch the progress of **Gitea** objects transitioning to running state

```bash
kubectl get pods -n gitea --watch
```

Once all settled, press **Ctrl+C** to exit and list most of the related objects

```bash
kubectl get all -n gitea 
```

Here, we will assume that the address assigned (in fact, specified in the values file) by the Load Balancer is ***192.168.99.51***.

## Gitea - Repositories Creation

Navigate to ***192.168.99.51:3000*** and use the credentials specified in the **gitea/values.yaml** file.

Import/migrate the following sample repostiories:

* <https://github.com/shekeriev/gitops-and-k8s-demo-app> as **gitops-app**

* <https://github.com/shekeriev/gitops-and-k8s-demo-app-infra> as **gitops-app-infra**

## Gitea - Personal Access Token

While still in Gitea UI, create a new access token by going to **Settings** > **Applications** > **Manage Access Tokens**.

Name it **gitops** (or something else) and git it at least the following permissions:

* **read:misc**

* **write:repository**

* **write:user**

Copy the token somewhere as we are going to needed it for a few of the coming steps.

## Jenkins - Installation

Check what Helm repositories you currently have

```bash
helm repo list
```

And if the **Jenkins** one is not listed, add it

```bash
helm repo add jenkins https://charts.jenkins.io
```

You may want to update the repositories

```bash
helm repo update
```
**
Install **Jenkins** by executing the following

```bash
helm install jenkins jenkins/jenkins --namespace=jenkins --create-namespace=true --set controller.admin.password=Parolka-12345 --set controller.serviceType=LoadBalancer 
```

*Of course, adjust the password (**Parolka-12345**) as you like. In addition, you can change or pass other custom values as per the chart specification.*

Watch the progress of **Jenkins** objects transitioning to running state

```bash
kubectl get pods --namespace jenkins --watch
```

Once all settled, press **Ctrl+C** to exit and list most of the related objects

```bash
kubectl get all -n jenkins
```

Then we must assign specific permissions to **jenkins** service account. In this particular example, we are going to assign the **cluster-admin** role, as we want to have a capable Jenkins instance up and running as quick and easy as possible, and focus on other matters

```bash
kubectl create clusterrolebinding jenkins --clusterrole=cluster-admin --serviceaccount=jenkins:jenkins
```

Depending on our setup, we may need to get where the **Jenkins** service is accessible. One way to accomplish this is to list the published services

```bash
minikube -p gitops service list
```

Or check with **kubectl**

```bash
kubectl get svc -n jenkins
```

If you are following closely, your should be available on

* <http://192.168.99.X:NODEPORT> where **X** is the last portion of the IP address of the minikube instance and **NODEPORT** is the NodePort assigned to the service

* <http://192.168.99.52:8080> the Load Balancer assigned address

We could check the admin password (if we did not set it via a value)

```bash
kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password
```

If set via a value, the result from the above should match the one used during the installation.

Prepare a **docker/config.json** file containing credentials (usually created via **docker login**).

Create a config map out of it (adjust the name and/or path to match yours). This CM will be used later by Jenkins

```bash
kubectl create configmap docker-config --from-file=./docker/config.json -n jenkins
```

Create Git credentials by visitng **Manage Jenkins** > **Credentials** > **Global** > **Add Credentials**.

Use the following:

* Kind: **Username with password**

* Username: ***username-from-gitea***

* Password: ***access-token-from-gitea***

* ID: **git-credentials**

## Gitea - Clone repositories (to the demo folder)

Navigate to the folder you plan to use for the demo and clone the two repositories you migrated from GitHub.

The one for the application's code

```bash
git clone http://GITEA-IP:3000/GITEA-USER/gitops-app
```

*You must substitute **GITEA-IP** and **GITEA-USER** with your values.*

And the one for the infrastructure files

```bash
git clone http://GITEA-IP:3000/GITEA-USER/gitops-app-infra
```

*You must substitute **GITEA-IP** and **GITEA-USER** with your values.*

## Jenkins - Pipelines

Return to Jenkins UI and create two pipelines by using the respective **Jenkinsfile** from the **gitops-app** repository:

* **pipeline-cicd** - use the file **Jenkinsfile-CICD**. This is an illustration of a "classic" simple CI/CD pipeline

* **pipeline-gitops** - use the file **Jenkinsfile-GitOps**. This is an illustration of how the "classic" pipeline should change to address the transition to GitOps

Ideally, they should be configured to poll the **gitops-app** repository or to be triggered by a web hook. However, please note that only one of them should be active. The **pipeline-cicd** is used just as a starting point and the **pipeline-gitops** is the pipeline we actually need. It represents "an evolution" of the classic CI/CD pipeline.
