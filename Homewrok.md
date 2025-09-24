# Home Assignment: ArgoCD – GitOps in Action
## Objective
This assignment is designed to give you hands-on experience with ArgoCD by applying GitOps
principles to deploy and manage Kubernetes applications. By completing this assignment, you
will:

    ● Understand how to set up ArgoCD in a Kubernetes cluster.
    ● Practice deploying applications using Git as the source of truth.
    ● Explore advanced features such as sync policies, hooks, and Helm chart deployments.

# Tasks
## Task 1: Setting Up ArgoCD
### 1. Setup a Kubernetes Cluster  
Use a local Kubernetes cluster (Minikube, Kind, or K3s) or a managed cloud  
solution (EKS, GKE, or AKS).  
Install ArgoCD using the official installation manifest.  
Copy code:
```
kubectl create namespace argocd
kubectl apply -n argocd -f
https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/in
stall.yaml
```
### 2. Access the ArgoCD UI
Port-forward the ArgoCD server and access the UI in your browser.  
Copy code:
```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

## Task 2: Deploy a Simple Application using ArgoCD
### 1. Create a Git Repository
Create a public or private Git repository containing Kubernetes manifests for a simple web application (e.g., Nginx or a basic Node.js app).  
Ensure the manifests are structured correctly (Deployment, Service, ConfigMap,etc.).  
### 2. Create an ArgoCD Application
Connect ArgoCD to the Git repository and deploy the application using either:
- UI: Create the application using the ArgoCD UI.
- CLI: Use the following example CLI command:  
Copy code:
```
argocd app create simple-app \
--repo <your-git-repo-url> \
--path <path-to-manifests> \
--dest-server https://kubernetes.default.svc \
--dest-namespace default
```

### 3. Sync the Application
- Manually sync the application and verify that the resources are created in your cluster.
- Make a small change in the Git repository (e.g., update an image version) and observe how ArgoCD detects the drift.

## Task 3: Deploy a Helm Chart using ArgoCD
### 1. Use a Helm Chart
- Choose a Helm chart from an official repository (e.g., the Nginx chart from the stable Helm repo).
- Deploy the chart using ArgoCD.
- Customize the deployment by specifying a values.yaml file.
### 2. Auto-Sync and Self-Heal
Enable auto-sync and self-heal policies for the Helm application .yaml  
Copy code:
```
syncPolicy:
automated:
prune: true
selfHeal: true
```

### 3. Test Auto-Sync
Delete a resource manually from the cluster and observe how ArgoCD restores it automatically.

## Task 4: Use Sync Waves and Hooks
### 1. Create a Multi-Resource Deployment
- Add dependencies in your application (e.g., a ConfigMap used by a Deployment).
- Use sync waves to ensure the ConfigMap is applied before the Deployment.
### 2. Implement a PreSync Hook
Add a hook to perform a custom action (e.g., print a log message) before syncing the application .yaml  
Copy code:
```
metadata:
annotations:
argocd.argoproj.io/hook: PreSync
```

## Task 5: Multi-Cluster Management
### 1. Add a Second Cluster
- Create a second Kubernetes cluster (e.g., using Kind or Minikube).
- Register the second cluster in ArgoCD and deploy a separate application to it.

## Deliverables
### 1. Git Repository Link
Provide the link to the Git repository containing your Kubernetes manifests and Helm
charts.
### 2. Screenshots
ArgoCD UI showing the deployed applications.  
Sync status before and after making changes in the Git repository.
### 3. Commands and Configurations
Include the ArgoCD CLI commands used and any YAML configurations (e.g.,syncPolicy, hooks, and sync waves).
### 4. Summary Report
Write a brief report (1-2 pages) summarizing your experience:
- Challenges faced and how you resolved them.
- Key learnings from the assignment.
### Submission Instructions
Submit your assignment by uploading the required files (report, screenshots, and Git repository link) to the course portal.