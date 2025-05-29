

Here’s a compact yet complete Argo CD + kubectl cheat sheet to keep at your fingertips while managing your GitOps-based EKS deployments.

🧠 Argo CD Cheat Sheet
🔍 App Management
bash
argocd app list                            # List all apps
argocd app get <app-name>                 # Describe app
argocd app sync <app-name>                # Force sync (Git -> cluster)
argocd app delete <app-name>              # Delete app
argocd app refresh <app-name>             # Refresh state from Git
argocd app diff <app-name>                # Show diffs between live & desired
argocd app logs <app-name>                # View logs (if any)

🛠 App Creation
bash
argocd app create <app-name> \
  --repo <repo-url> \
  --path <path-in-repo> \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace <namespace> \
  --sync-policy automated

🔐 Argo CD Auth
bash
argocd login <argocd-server>              # Login (use --insecure if needed)
argocd account list                       # See available accounts
argocd logout <argocd-server>             # Logout

💻 kubectl Cheat Sheet
📦 Basic Resource Management
bash
kubectl get pods -n <ns>                 # List pods
kubectl describe pod <name> -n <ns>      # Details of a pod
kubectl logs <pod> -n <ns>               # Logs of a pod
kubectl exec -it <pod> -n <ns> -- /bin/sh  # Shell into container
kubectl delete pod <name> -n <ns>        # Delete a pod

📁 Apply / Delete YAML
bash
kubectl apply -f <file>.yaml             # Apply manifest
kubectl delete -f <file>.yaml            # Delete resources in manifest

🌐 Services & Ingress
bash
kubectl get svc -n <ns>                  # Services in namespace
kubectl get ingress -n <ns>              # Ingress rules
kubectl describe ingress <name> -n <ns>  # Details of ingress

🧩 Namespaces, Config, Info
bash
kubectl get ns                           # List all namespaces
kubectl config get-contexts              # Show all contexts
kubectl config use-context <context>     # Switch kube context
kubectl cluster-info                     # Cluster endpoint info

🎯 Bonus: ALB Debugging Helpers
bash
kubectl get ingress -A                           # Check if Ingress is created
kubectl get svc -n kube-system | grep alb        # See if ALB controller is running
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller