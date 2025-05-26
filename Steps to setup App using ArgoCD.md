# Deploy a Sample App Using Argo CD
You’ll deploy a simple NGINX web app from a Git repo using Argo CD GitOps.

🛠️ Step 4.1: Create a Git Repo Structure (if not done)
You can create a Git repo like this:

ce-grp-1-apps/
├── nginx/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml

Example contents:

nginx/deployment.yaml

yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80

nginx/service.yaml

yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer

nginx/kustomization.yaml

yaml
resources:
  - deployment.yaml
  - service.yaml
Push this folder (nginx/) to your GitHub repo (e.g., https://github.com/aalimsee/ce-grp-1-apps.git).

🧩 Step 4.2: Register App in Argo CD
Create a Kubernetes manifest for the Argo CD app (or use CLI).

Option A: CLI Method

bash
argocd app create nginx \
  --repo https://github.com/aalimsee/ce-grp-1-apps.git \
  --path nginx \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

Option B: YAML Method

yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/aalimsee/ce-grp-1-apps.git
    targetRevision: HEAD
    path: nginx
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      selfHeal: true
      prune: true

Apply it:

bash
kubectl apply -f nginx-app.yaml

✅ Step 4.3: Verify Deployment
bash
kubectl get svc -n default
Look for an external EXTERNAL-IP of the nginx service (type: LoadBalancer).

Then open it in your browser — you should see the NGINX welcome page.

