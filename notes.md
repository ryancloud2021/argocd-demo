# ArgoCD + Kubernetes: Learn GitOps | GitHub, Helm, Kustomize

This hands-on tutorial will guide you through implementing GitOps workflows using ArgoCD with Kubernetes. You'll learn three different approaches to manage your applications:

1. Basic ArgoCD setup - Direct deployment from a GitHub repository
2. Environment management with Kustomize - Using overlays for different environments
3. Helm-based deployments - Leveraging Helm charts for more complex applications

By the end of this tutorial, you'll have practical experience with GitOps principles and understand how to implement continuous deployment pipelines that automatically sync your Kubernetes resources with your Git repositories.

## Add a GitHub repository

To add a GitHub repository to your workspace in Visual Studio Code, follow these steps:

1. **Open Visual Studio Code**.

2. **Open the Command Palette** by pressing `Ctrl+Shift+P` (or `Cmd+Shift+P` on macOS).

3. **Clone the Repository**:
   - Type `Git: Clone` and select the `Git: Clone` option.
   - Enter the URL(https://github.com/ryancloud2021/argocd-demo.git) of the GitHub repository you want to clone.
   - Choose a local directory(argocd-demo) where you want to clone the repository.

4. **Open the Cloned Repository**:
   - Once the repository is cloned, you will be prompted to open the cloned repository. Click `Open`.

Alternatively, you can add an existing local repository to your workspace:

1. **Open Visual Studio Code**.

2. **Add Folder to Workspace**:
   - Go to `File` > `Add Folder to Workspace...`.
   - Select the folder that contains your local GitHub repository.

3. **Save the Workspace** (optional):
   - Go to `File` > `Save Workspace As...` to save your workspace configuration.

This will add the GitHub repository to your current workspace in Visual Studio Code.

## Setting Up ArgoCD

Let's start by installing ArgoCD in our Kubernetes cluster using Helm.

- Installing ArgoCD

Run the following commands to add the Argo Helm repository and install ArgoCD:

```shell
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create namespace argocd
helm install argocd argo/argo-cd --namespace argocd
```

- Wait for all the ArgoCD components to start:

```shell
kubectl get pods -n argocd
```
- Accessing the ArgoCD UI

To access the ArgoCD UI, we'll use port forwarding:

```shell
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
```

Now you can access the ArgoCD UI by visiting https://localhost:8080 in your web browser.

- Getting the Initial Admin Password

To retrieve the initial admin password:

```shell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Use `admin` as the username and the retrieved password to log in to the ArgoCD UI.


## Approach 1: Basic GitOps with ArgoCD

In our first approach, we'll set up a direct connection between our Git repository and our Kubernetes cluster.

### Understanding the Application Resource

ArgoCD uses an Application custom resource to define which Git repositories to monitor and how to sync them to your cluster.

Here's our first Application resource (argocd/applications/basic-application.yaml):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: grade-submission
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/ryancloud2021/argocd-demo.git
    targetRevision: HEAD
    path: gitops/basic
  destination:
    server: https://kubernetes.default.svc
    namespace: grade-demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

Let's break down this configuration:

**name:** The name of our application  
**source:** Defines where ArgoCD should look for our application manifests  
**repoURL:** The GitHub repository URL  
**targetRevision:** The Git revision to track (HEAD for the latest commit on the default branch)  
**path:** The path within the repository where our Kubernetes manifests are located  
**destination:** Defines where the application should be deployed  
**server:** The Kubernetes API server (in this case, the same cluster where ArgoCD is installed)  
**namespace:** The target namespace for our application  
**syncPolicy:** Defines how ArgoCD should sync the application  
**automated.prune:** Automatically delete resources that were removed from Git  
**automated.selfHeal:** Automatically fix drift between the desired state in Git and the actual state in the cluster  
**syncOptions.CreateNamespace:** Create the target namespace if it doesn't exist  

## Add a Repository in Argo CD

- Generate RSA keys

```shell
> pwd
/home/antony/argocd/sshkeys/dev
> ssh-keygen -t rsa -b 4096 -f /home/antony/argocd/sshkeys/dev/id_rsa
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/antony/argocd/sshkeys/dev/id_rsa
Your public key has been saved in /home/antony/argocd/sshkeys/dev/id_rsa.pub
The key fingerprint is:
SHA256:Y2tfjYGQiVQGiNmOGmEWoJDIcnRJmR4jhKKn+FM3npU antony@antony-Latitude-7480
The key's randomart image is:
+---[RSA 4096]----+
|**+=o=ooo        |
|X+=.O. o o       |
|Bo = o. +        |
|o o o    . .     |
|.=      S.. .    |
|+   . o.Eo   +   |
| . . o +o   o .  |
|  o   o. . .     |
|   .      .      |
+----[SHA256]-----+
```

- Adding a new SSH key to your GitHub account

1. In the upper-right corner of any page on GitHub, click your profile photo, then click  **Settings**.  
2. In the "Access" section of the sidebar, click **SSH and GPG keys**.  
3. Click **New SSH key** or **Add SSH key**.  
4. In the "Title" field, add a descriptive label for the new key.  
5. In the "Key" field, paste your `public key`.

- Add repository in argo CD

Go to Settings > Repositories > Connect Repo

| Field                | Description                                      |
|----------------------|--------------------------------------------------|
| Name                 | The name of the repository connection            |
| Project              | The ArgoCD project to associate with the repo    |
| Repository URL       | The URL of the Git repository                    |
| SSH Private key data | The private key for SSH authentication           |


- Verify the connection status. 

You should see `successful` in the connection status.

## Deploying Our First Application

- Apply `basic-application.yaml` manifest:

```shell
kubectl apply -f argocd/applications/basic-application.yaml
```

Now check the ArgoCD UI - you should see your application being synchronized with the Git repository. ArgoCD will create the necessary resources in your cluster based on the Kubernetes manifests in the specified Git repository path.

![deployment](<Screenshot from 2025-03-11 16-45-05.png>)

- Scale up `grade-submission-api-deployment.yaml` 

Scale up replicaset of `grade-submission-api-deployment.yaml` deployment from 3 to 5.  

