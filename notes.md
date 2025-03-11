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

