# ce-grp-1-apps
Capstone project cohort 9

# Port-forward ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# After Git Push
## How to Deploy It
Apply it once:

bash
kubectl apply -f apps/app-of-apps.yaml -n argocd

Then sync it:

bash
argocd app sync app-of-apps (this will not work before port-forward not perform)

Now Argo CD will automatically sync:
nginx-prod-webapp-aalimsee
aws-load-balancer-controller

Run these:

bash
kubectl port-forward svc/argocd-server -n argocd 8080:443

Then in another terminal:

bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo

Then:

bash
argocd login localhost:8080 --username admin --password <copied-password> --insecure

Then:

bash
argocd app sync app-of-apps

Let me know if you want to configure Ingress and HTTPS next — I can generate the full Argo CD Ingress manifest using your domain (admin-argocd.sctp-sandbox.com)

# What This Means
Argo CD pulled your apps/ directory from Git
It found the nginx and alb controller apps
It automatically deployed everything to your cluster
Your GitOps is live and working 💥

