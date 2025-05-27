# ce-grp-1-apps
Capstone project cohort 9

# EKS + ALB + Argo CD + HTTPS Deployment
## Overview

This deployment sets up a production-grade Kubernetes application on Amazon EKS, using:

* **AWS Route 53** for custom domain DNS resolution
* **AWS ACM** for managed TLS certificates
* **AWS ALB Ingress Controller** for routing and HTTPS termination
* **Argo CD** for GitOps-based continuous delivery

---

## Domain

* **Hosted Zone**: `sctp-sandbox.com`
* **Subdomain**: `ce-grp-1.sctp-sandbox.com`

---

## Components

### 1. Route 53 (Terraform)

```hcl
resource "aws_route53_record" "ce_grp_1" {
  zone_id = data.aws_route53_zone.primary.zone_id
  name    = "ce-grp-1"
  type    = "A"

  alias {
    name                   = "k8s-prod-nginxing-75a4ff3d14-280868929.us-east-1.elb.amazonaws.com"
    zone_id                = "Z32O12XQLNTSW2"
    evaluate_target_health = true
  }
}
```

### 2. ACM Certificate (Terraform)

```hcl
resource "aws_acm_certificate" "cert_ce_grp_1" {
  domain_name       = "ce-grp-1.sctp-sandbox.com"
  validation_method = "DNS"
}
```

---

## Ingress Configuration (ingress.yaml)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: prod
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:<acct>:certificate/<cert-id>
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80, "HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  rules:
  - host: ce-grp-1.sctp-sandbox.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-prod-webapp-aalimsee
            port:
              number: 80
  tls:
  - hosts:
    - ce-grp-1.sctp-sandbox.com
    secretName: dummy
```

---

## Argo CD Application (nginx-app.yaml)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-prod-webapp-aalimsee
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
      prune: true
      selfHeal: true
```

---

## Result

✅ HTTPS secured application available at:

```
https://ce-grp-1.sctp-sandbox.com
```

✅ Managed by Argo CD and fully GitOps-enabled.



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

