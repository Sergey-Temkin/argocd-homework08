## Task 1: Setting Up ArgoCD
### Create the recommended structure:
```bash
mkdir -p argocd-lab/{manifests,helm,docs,screenshots}
```
### Start a fresh, isolated minikube profile named "argo-lab":
```bash
minikube start -p argo-lab --driver=docker --cpus=2 --memory=2200 --disk-size=10g
```
### Make kubectl talk to this profile’s cluster (sets context):
```bash
kubectl config use-context argo-lab
```
### Create the ArgoCD namespace:
```bash
kubectl create namespace argocd
```
### Install ArgoCD using the official stable manifest:
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
### Verify:
```bash
kubectl get pods -n argocd
kubectl get svc  -n argocd
```
### Store the password in an environment variable and print it:
```bash
ARGO_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo)
echo "$ARGO_PWD"
```
### Password:
- Copy password from the command above
### Recommended for you (avoid 8080 conflicts):
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
## Task 2: Deploy a Simple Application using ArgoCD
### Login to ArgoCD:
- Login user: admin
- Local URL: https://localhost:8080
- Password: password from the command above

### Create the app folder:
```bash
mkdir -p argocd-lab/manifests/simple-nginx
```
### Create configmap.yaml:
```bash
cat > argocd-lab/manifests/simple-nginx/configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: simple-nginx-index             # ConfigMap name used by the Deployment volume
  labels:
    app: simple-nginx                  # App label to group resources
data:
  index.html: |                        # The actual HTML page served by Nginx
    <!DOCTYPE html>
    <html>
      <head>
        <meta charset="utf-8"/>
        <title>ArgoCD Demo</title>
        <style>
          body { font-family: Arial, sans-serif; margin: 40px; }
          .ok { padding: 10px 14px; border: 1px solid #2f855a; display: inline-block; }
        </style>
      </head>
      <body>
        <h1>ArgoCD GitOps Demo — v1</h1>
        <p class="ok">If you see this page, your Deployment, Service and ConfigMap are working.</p>
      </body>
    </html>
EOF
```
### Create deployment.yaml:
```bash
cat > argocd-lab/manifests/simple-nginx/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-nginx
  labels:
    app: simple-nginx                  # Same label ties resources together
spec:
  replicas: 1                          # Keep it small for minikube
  selector:
    matchLabels:
      app: simple-nginx
  template:
    metadata:
      labels:
        app: simple-nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine     # Small Nginx image
          ports:
            - containerPort: 80        # Default Nginx port
          volumeMounts:
            - name: web
              mountPath: /usr/share/nginx/html   # Serve our ConfigMap content
      volumes:
        - name: web
          configMap:
            name: simple-nginx-index   # Mount the ConfigMap created above
            items:
              - key: index.html
                path: index.html       # File will appear as /usr/share/nginx/html/index.html
EOF
```
### Create service.yaml:
```bash
cat > argocd-lab/manifests/simple-nginx/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: simple-nginx
  labels:
    app: simple-nginx
spec:
  type: NodePort                       # NodePort so we can curl/browse via minikube service
  selector:
    app: simple-nginx
  ports:
    - name: http
      port: 80                         # Service port
      targetPort: 80                   # Pod's containerPort
      nodePort: 30080                  # Fixed NodePort (optional); allows stable external access
EOF
```
### .gitignore: 
```bash
cat > .gitignore << 'EOF'
# Python / Node / Editor junk we might not need for this repo
__pycache__/
*.pyc
node_modules/
.vscode/
.idea/
.DS_Store
EOF
```
### Stay in homework08 root for clarity:
```bash
cd /mnt/c/Users/USER/Desktop/DevOpsCourse/class0107/homework/homework08
```
### Initialize git:
```bash
git init
```
### Set your identity (global once; you already asked for these earlier):
```bash
git config --global user.name "Sergey-Temkin"
git config --global user.email "sergheytemkin@gmail.com"
```
### Stage and commit:
```bash
git add .
git commit -m "Task 2.1: add simple Nginx app manifests (ConfigMap, Deployment, Service)"
```
### Create the repo on GitHub (web):
- Go to GitHub → New repository
- Name: argocd-homework08
### Link your local repo to the remote and push:
```bash
git remote add origin https://github.com/Sergey-Temkin/argocd-homework08.git
git branch -M main
git push -u origin main
```
### Create Application using ArgoCD UI: 
In your browser, go to your ArgoCD UI:
```bash
https://localhost:8080
```  
### Login as admin with the password you decoded:
- Click + NEW APP (top-left).
- Fill the form:
- Application Name: simple-nginx
- Project: default
- SYNC POLICY: Manual
- Repository URL: https://github.com/Sergey-Temkin/argocd-homework08.git
- Revision: HEAD (or main)
- Path: argocd-lab/manifests/simple-nginx
- Cluster URL: https://kubernetes.default.svc
- Namespace: default
Click CREATE.
### Verify(CLI):
```bash
kubectl get applications.argoproj.io -n argocd
```
### Sync the app:
- Open ArgoCD UI → click your app simple-nginx
- Click Sync → Synchronize
- Wait until it shows Synced and Healthy
### Verification(CLI):
```bash
kubectl get deploy,po,svc -n default -l app=simple-nginx
```
### Test the app via NodePort:
```bash
minikube -p argo-lab ip
```
### Make a small Git change to observe drift(Update the HTML inside the ConfigMap data):
```bash
cat > argocd-lab/manifests/simple-nginx/configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: simple-nginx-index
  labels:
    app: simple-nginx
data:
  index.html: |
    <!DOCTYPE html>
    <html>
      <head>
        <meta charset="utf-8"/>
        <title>ArgoCD Demo</title>
        <style>
          body { font-family: Arial, sans-serif; margin: 40px; }
          .ok { padding: 10px 14px; border: 1px solid #2f855a; display: inline-block; }
        </style>
      </head>
      <body>
        <h1>ArgoCD GitOps Demo — v2</h1>
        <p class="ok">This is version v2. ArgoCD should show the app OutOfSync until we sync.</p>
      </body>
    </html>
EOF
```
### Dry run locally (optional sanity):
```bash
kubectl apply --dry-run=client -f argocd-lab/manifests/simple-nginx/
```
### Commit & push:
```bash
git add argocd-lab/manifests/simple-nginx/configmap.yaml
git commit -m "Task 2.3: Bump index page to v2 to demonstrate drift"
git push
```
### Observe drift in ArgoCD:
- In the UI, your app should change to OutOfSync (ArgoCD noticed the Git change).
- Click Refresh if needed.
- Sync to apply the change:
- UI: Sync → Synchronize
## Task 3: Deploy a Helm Chart using ArgoCD
### 1. Use a Helm Chart
### Create a dedicated server-block ConfigMap
```bash
cat > argocd-lab/helm/helm-nginx-serverblock-cm.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-nginx-serverblock          # <- we reference this name from Helm values
  namespace: default                  # Bitnami NGINX is deploying in 'default'
  labels:
    app.kubernetes.io/name: nginx
data:
  server-block.conf: |-
    server {
      listen 0.0.0.0:8080;            # HTTP only
      server_name _;
      root /app;
      index index.html;
      location / {
        try_files $uri $uri/ =404;
      }
    }
EOF
```
### Apply:
```bash
kubectl apply -f argocd-lab/helm/helm-nginx-serverblock-cm.yaml
```
### Create the ArgoCD Application(Helm NGINX):
```bash
cat > argocd-lab/manifests/argocd-app-helm-nginx.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: helm-nginx
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://charts.bitnami.com/bitnami
    chart: nginx
    targetRevision: "*"
    helm:
      values: |
        replicaCount: 1

        httpsEnabled: false
        containerPorts:
          http: 8080
        service:
          type: NodePort
          ports:
            http: 80
          nodePorts:
            http: 30081

        existingServerBlockConfigmap: my-nginx-serverblock

        staticSite:
          enabled: true
          content: |
            <!DOCTYPE html>
            <html>
            <body>
              <h1>ArgoCD + Helm Demo</h1>
              <p>Status: <b>v1</b></p>
            </body>
            </html>

        networkPolicy:
          enabled: false

  destination:
    server: https://kubernetes.default.svc
    namespace: default

  syncPolicy: {}
EOF
```
### Apply it to the cluster:
```bash
kubectl apply -n argocd -f argocd-lab/manifests/argocd-app-helm-nginx.yaml
```
### Sync the Helm app:
Sync (UI):
- ArgoCD UI → select helm-nginx → Sync → Synchronize
### Check resources:
```bash
kubectl get deploy,po,svc -n default -l app.kubernetes.io/name=nginx
```
### 2. Enable Auto-Sync and Self-Heal
```bash
cat > argocd-lab/manifests/argocd-app-helm-nginx.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: helm-nginx
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://charts.bitnami.com/bitnami
    chart: nginx
    targetRevision: "*"
    helm:
      values: |
        replicaCount: 1

        httpsEnabled: false
        containerPorts:
          http: 8080
        service:
          type: NodePort
          ports:
            http: 80
          nodePorts:
            http: 30081

        existingServerBlockConfigmap: my-nginx-serverblock

        staticSite:
          enabled: true
          content: |
            <!DOCTYPE html>
            <html>
            <body>
              <h1>ArgoCD + Helm Demo</h1>
              <p>Status: <b>v1</b></p>
            </body>
            </html>

        networkPolicy:
          enabled: false

  destination:
    server: https://kubernetes.default.svc
    namespace: default

  # 🔄 Auto-sync enabled
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```
### Apply the new Application definition
```bash
kubectl apply -n argocd -f argocd-lab/manifests/argocd-app-helm-nginx.yaml
```
### Commit and push so Git remains source of truth
```bash
git add argocd-lab/manifests/argocd-app-helm-nginx.yaml
git commit -m "helm-nginx: enable auto-sync and self-heal"
git push
```
### Sync the Helm app:
Sync (UI):
- ArgoCD UI → select helm-nginx → Sync → Synchronize
### 3. Test Auto-Sync & Self-Heal
### Delete a resource manually:
```bash
kubectl delete svc helm-nginx -n default
```
### Watch in ArgoCD UI:
- Open the helm-nginx app in ArgoCD.
- Within seconds you should see:
- Sync Status: OutOfSync → Synced
- Service reappears in the resource tree.
### Verify in cluster:
```bash
kubectl get svc helm-nginx -n default

```
## Task 4: Use Sync Waves and Hooks

### Create a Multi-Resource Deployment:
```bash
mkdir -p argocd-lab/manifests/sync-waves-demo
```
### ConfigMap (wave 0):
```bash
cat > argocd-lab/manifests/sync-waves-demo/configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: waves-demo-config
  annotations:
    # Ensure this applies before the Deployment
    argocd.argoproj.io/sync-wave: "0"
data:
  MESSAGE: "Hello from ConfigMap (wave 0)"
EOF
```
### Deployment (wave 1) uses the ConfigMap:
```bash
cat > argocd-lab/manifests/sync-waves-demo/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: waves-demo
  labels:
    app: waves-demo
  annotations:
    # Run after the ConfigMap
    argocd.argoproj.io/sync-wave: "1"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: waves-demo
  template:
    metadata:
      labels:
        app: waves-demo
    spec:
      containers:
        - name: alpine
          image: alpine:3.20
          command: ["/bin/sh","-c"]
          args:
            - 'echo "Container started, MESSAGE=$(MESSAGE)"; sleep 3600'
          env:
            - name: MESSAGE
              valueFrom:
                configMapKeyRef:
                  name: waves-demo-config
                  key: MESSAGE
EOF
```
### PreSync hook (Job runs before sync):
```bash
cat > argocd-lab/manifests/sync-waves-demo/presync-job.yaml << 'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: presync-greet
  annotations:
    # Run before syncing other resources
    argocd.argoproj.io/hook: PreSync
    # Clean up after success so repeated syncs stay tidy
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: hello
          image: busybox:1.36
          command: ["/bin/sh","-c"]
          args:
            - 'echo "PreSync: preparing environment..." && sleep 2 && echo "PreSync: done."'
  backoffLimit: 0
EOF
```
### Create the ArgoCD Application (manual sync so you can see the order):
```bash
cat > argocd-lab/manifests/argocd-app-sync-waves.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sync-waves-demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Sergey-Temkin/argocd-homework08.git
    targetRevision: HEAD
    path: argocd-lab/manifests/sync-waves-demo
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  # Keep manual so you can click SYNC and watch waves & hook order
  syncPolicy: {}
EOF
```
### Apply, commit, push:
```bash
kubectl apply -n argocd -f argocd-lab/manifests/argocd-app-sync-waves.yaml

git add argocd-lab/manifests/sync-waves-demo/*.yaml argocd-lab/manifests/argocd-app-sync-waves.yaml
git commit -m "Task 4: sync waves + PreSync hook demo"
git push
```
### Sync from the UI only (to observe order)
- Open ArgoCD UI → sync-waves-demo
- Click SYNC → SYNCHRONIZE
- Watch the timeline / events:
- PreSync Job presync-greet runs first (and gets deleted after success).
- wave 0: ConfigMap applies.
- wave 1: Deployment applies.
### Check deployment & pod:
```bash
kubectl get deploy,po -n default -l app=waves-demo
```
## Task 5: Multi-Cluster Management
### Create a second Kubernetes cluster (Minikube profile):
```bash
minikube start -p argo-lab   --driver=docker --cpus=2 --memory=2200 --disk-size=10g
minikube start -p argo-lab-2 --driver=docker --cpus=2 --memory=2200 --disk-size=10g
```
### Put both Minikube nodes on the same Docker network
```bash
docker network create argonet || true
docker network connect argonet argo-lab   || true
docker network connect argonet argo-lab-2 || true
```
### Create credentials on cluster 2 (service account, token, CA):
```bash
# Work against the second cluster
kubectl config use-context argo-lab-2

# SA + cluster-admin for Argo CD to use
kubectl -n kube-system create serviceaccount argocd-remote
kubectl create clusterrolebinding argocd-remote-admin \
  --clusterrole=cluster-admin --serviceaccount=kube-system:argocd-remote

# Reachable API IP (the container IP on argonet) + SNI (profile IP on cert)
export ARGO2_API=$(docker inspect -f '{{ (index .NetworkSettings.Networks "argonet").IPAddress }}' argo-lab-2)
export SNI=$(minikube -p argo-lab-2 ip)

# Long-lived token (K8s 1.24+)
export TOKEN=$(kubectl -n kube-system create token argocd-remote --duration=2160h)

# Base64 CA for this cluster (from kubeconfig CA file)
export CA=$(base64 -w0 "$(kubectl config view --raw --minify -o jsonpath='{.clusters[0].cluster.certificate-authority}')")

```
### Register cluster 2 in Argo CD (declarative Secret in cluster 1):
```bash
# Switch to the cluster where Argo CD runs
kubectl config use-context argo-lab

# (Optional) keep this secret file out of git
echo 'argocd-lab/manifests/cluster-*-secret.yaml' >> .gitignore

# Create the cluster secret manifest
mkdir -p argocd-lab/manifests
cat > argocd-lab/manifests/cluster-argo-lab-2-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: cluster-argo-lab-2
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
stringData:
  name: argo-lab-2
  server: https://$ARGO2_API:8443
  config: |
    {
      "bearerToken": "$TOKEN",
      "tlsClientConfig": {
        "caData": "$CA",
        "serverName": "$SNI"
      }
    }
EOF

# Apply it
kubectl apply -f argocd-lab/manifests/cluster-argo-lab-2-secret.yaml

```
### Remote app manifests (go into your repo):
```bash
mkdir -p argocd-lab/manifests/simple-nginx-remote

# ConfigMap
cat > argocd-lab/manifests/simple-nginx-remote/configmap.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: simple-nginx-remote-index
  labels:
    app: simple-nginx-remote
data:
  index.html: |
    <!DOCTYPE html>
    <html>
      <head><meta charset="utf-8"/><title>Remote Cluster App</title></head>
      <body style="font-family: Arial; margin: 40px;">
        <h1>ArgoCD Multi-Cluster Demo — SECOND CLUSTER</h1>
        <p>This page is served from the <b>argo-lab-2</b> Minikube cluster.</p>
      </body>
    </html>
EOF

# Deployment
cat > argocd-lab/manifests/simple-nginx-remote/deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-nginx-remote
  labels:
    app: simple-nginx-remote
spec:
  replicas: 1
  selector:
    matchLabels:
      app: simple-nginx-remote
  template:
    metadata:
      labels:
        app: simple-nginx-remote
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: web
              mountPath: /usr/share/nginx/html
      volumes:
        - name: web
          configMap:
            name: simple-nginx-remote-index
            items:
              - key: index.html
                path: index.html
EOF

# Service (NodePort 30090)
cat > argocd-lab/manifests/simple-nginx-remote/service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: simple-nginx-remote
  labels:
    app: simple-nginx-remote
spec:
  type: NodePort
  selector:
    app: simple-nginx-remote
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30090
EOF
```
### Commit & push:
```bash
git add argocd-lab/manifests/simple-nginx-remote/*.yaml .gitignore
git commit -m "Task 5: remote app manifests (simple-nginx-remote)"
git push
```
### Argo CD Application that targets cluster 2:
```bash
cat > argocd-lab/manifests/argocd-app-remote.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: simple-nginx-remote
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Sergey-Temkin/argocd-homework08.git
    targetRevision: HEAD
    path: argocd-lab/manifests/simple-nginx-remote
  destination:
    name: argo-lab-2
    namespace: default
  syncPolicy: {}
EOF

# Apply it
kubectl apply -n argocd -f argocd-lab/manifests/argocd-app-remote.yaml
```
### Sync:
# open UI
```bash
kubectl -n argocd port-forward svc/argocd-server 8085:443
# then visit https://localhost:8085, open app "simple-nginx-remote", Sync → Synchronize
```
### Verify on cluster 2:
```bash
kubectl --context argo-lab-2 -n default get deploy,po,svc -l app=simple-nginx-remote -o wide
```













