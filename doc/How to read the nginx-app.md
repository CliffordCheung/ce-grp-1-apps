
# This YAML defines an Argo CD Application resource, which tells Argo CD how to deploy and manage a Kubernetes application from a Git repository using GitOps. Here's a breakdown of each field:

# Header
yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

Declares this as an Argo CD Application resource.
apiVersion: argoproj.io/v1alpha1 is the version supported by Argo CD.

# Metadata
yaml
metadata:
  name: nginx
  namespace: argocd

name: Unique identifier for this Argo CD application (nginx).
namespace: Must be the namespace where Argo CD is installed (usually argocd).

# Spec
The heart of the configuration—this tells Argo CD what to deploy, where, and how.

# project
yaml
project: default
The Argo CD project this application belongs to. default is the default project unless you’ve defined a custom one.

# source
yaml
source:
  repoURL: https://github.com/aalimsee/ce-grp-1-apps.git
  targetRevision: HEAD
  path: nginx

repoURL: Git repository where your Kubernetes manifests or Helm chart lives.
targetRevision: Git branch, tag, or commit (HEAD means latest commit on the default branch).
path: Directory inside the repo (nginx) containing the Kubernetes YAML or Helm chart.

# destination
yaml
destination:
  server: https://kubernetes.default.svc
  namespace: default

server: https://kubernetes.default.svc means deploy to the same cluster where Argo CD is running.
namespace: Kubernetes namespace to deploy the application (here it's default).

# syncPolicy
yaml
syncPolicy:
  automated:
    selfHeal: true
    prune: true

Enables automated sync:
selfHeal: true: Argo CD will automatically fix any drift (e.g., manual changes).
prune: true: Deletes resources that are removed from the Git repo (ensures Git is source of truth).

# Summary
This Argo CD Application tells Argo CD to:
Track the nginx folder in your GitHub repo
Automatically deploy it to the default namespace in the cluster
Keep the deployment synced with Git using GitOps principles


# Here’s a full Argo CD Application manifest configured for the prod namespace:

yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/aalimsee/ce-grp-1-apps.git
    targetRevision: HEAD
    path: nginx
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      selfHeal: true
      prune: true