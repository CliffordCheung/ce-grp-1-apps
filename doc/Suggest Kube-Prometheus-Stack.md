

✅ Step-by-Step Integration
1. 📁 Folder Structure
repo-root/
├── apps/
│   ├── app-of-apps.yaml
│   ├── nginx-app.yaml
│   ├── aws-load-balancer-controller-app.yaml
│   └── monitoring-app.yaml   # <- NEW app
├── nginx/
│   ├── deployment.yaml
│   ├── ingress.yaml
│   └── service.yaml
└── monitoring/         # <- Optional custom values
    └── values.yaml     # (Optional) Custom overrides if needed

2. 📄 apps/monitoring-app.yaml
yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 58.1.0
    helm:
      values: |
        grafana:
          adminPassword: admin
          service:
            type: LoadBalancer
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      selfHeal: true
      prune: true

3. 🆕 Add Helm Repo to Argo CD (CLI)
bash
argocd repo add https://prometheus-community.github.io/helm-charts \
  --type helm \
  --name prometheus-community

4. 📦 Commit and Push
bash
git add apps/monitoring-app.yaml
git commit -m "Add kube-prometheus-stack monitoring app"
git push

5. 🔁 Sync via Argo CD
bash
argocd app sync app-of-apps
This will auto-discover and deploy kube-prometheus-stack.

6. 🔍 Access Grafana
Once deployed:

bash
kubectl get svc -n monitoring
Look for LoadBalancer on the Grafana service, e.g.:

ruby
kube-prometheus-stack-grafana   LoadBalancer   a.b.c.d.elb.amazonaws.com   80:xxxxx/TCP

Access it at http://<elb-url>
Login: admin / admin



# To expose Grafana from kube-prometheus-stack using your existing Route 53 + ACM (HTTPS) setup from Terraform, follow these steps:

✅ Goal
Expose http://<elb-url> as:

arduino
https://monitoring.ce-grp-1.sctp-sandbox.com
Secured by ACM and managed by Route 53.

🪜 Step-by-Step Setup
1. Confirm Terraform Created These:
✅ A record in Route 53:

hcl
resource "aws_route53_record" "monitoring" {
  zone_id = aws_route53_zone.primary.zone_id
  name    = "monitoring.ce-grp-1.sctp-sandbox.com"
  type    = "A"

  alias {
    name                   = "<grafana-elb-dns>"
    zone_id                = "<grafana-elb-zone-id>"
    evaluate_target_health = true
  }
}
✅ ACM certificate in us-east-1:

hcl
resource "aws_acm_certificate" "monitoring_cert" {
  domain_name       = "monitoring.ce-grp-1.sctp-sandbox.com"
  validation_method = "DNS"
  ...
}
2. Update Helm Values for Grafana to Use Ingress + HTTPS
Create or update this file:

monitoring/values.yaml
yaml
grafana:
  enabled: true
  adminPassword: admin

  service:
    enabled: false

  ingress:
    enabled: true
    ingressClassName: alb
    annotations:
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/target-type: ip
      alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:255945442255:certificate/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80, "HTTPS": 443}]'
      alb.ingress.kubernetes.io/ssl-redirect: '443'
    hosts:
      - monitoring.ce-grp-1.sctp-sandbox.com
    tls:
      - hosts:
          - monitoring.ce-grp-1.sctp-sandbox.com
        secretName: dummy-secret  # Not used, required for Kubernetes spec
⚠️ Note: ACM cert is handled at ALB — not by Kubernetes secretName.

3. Update Argo CD App to Use This values.yaml
Update apps/monitoring-app.yaml:

yaml
source:
  repoURL: https://prometheus-community.github.io/helm-charts
  chart: kube-prometheus-stack
  targetRevision: 58.1.0
  helm:
    valueFiles:
      - monitoring/values.yaml
OR inline it with:

yaml
    helm:
      values: |
        grafana:
          enabled: true
          ...

4. Push and Sync
bash
git add monitoring/values.yaml apps/monitoring-app.yaml
git commit -m "Expose Grafana via Route53 + HTTPS"
git push

argocd app sync kube-prometheus-stack

✅ Final Result
Visit:

arduino
https://monitoring.ce-grp-1.sctp-sandbox.com
To access Grafana over HTTPS, via ALB + ACM + Route 53, all Terraform and Argo CD integrated.

Let me know if you want to auto-create the Route 53 record with Terraform using ELB hostname from Argo CD sync.


# Here’s the Terraform code to connect your ALB-managed Grafana Ingress with Route 53 + HTTPS (ACM) using your domain monitoring.ce-grp-1.sctp-sandbox.com.

✅ Assumptions
You already deployed:

ALB via Kubernetes Ingress (via Argo CD)

ACM cert for monitoring.ce-grp-1.sctp-sandbox.com

Hosted zone for sctp-sandbox.com

ALB hostname is known (e.g., from kubectl get ingress -n monitoring)

🪜 Step-by-Step Terraform Setup
1. ACM Certificate (if not already)
hcl
resource "aws_acm_certificate" "monitoring" {
  domain_name       = "monitoring.ce-grp-1.sctp-sandbox.com"
  validation_method = "DNS"

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name = "monitoring-ce-grp-1"
  }
}
2. ACM DNS Validation
hcl
resource "aws_route53_record" "monitoring_cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.monitoring.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      type   = dvo.resource_record_type
      record = dvo.resource_record_value
    }
  }

  zone_id = aws_route53_zone.sctp_sandbox.zone_id
  name    = each.value.name
  type    = each.value.type
  records = [each.value.record]
  ttl     = 300
}
3. Validate Certificate
hcl
resource "aws_acm_certificate_validation" "monitoring" {
  certificate_arn         = aws_acm_certificate.monitoring.arn
  validation_record_fqdns = [for record in aws_route53_record.monitoring_cert_validation : record.fqdn]
}
4. Route 53 A Record (Alias to ALB)
hcl
data "aws_lb" "grafana_alb" {
  name = "k8s-monitoring-grafana-xxxx" # <-- Replace with actual ALB name
}

resource "aws_route53_record" "grafana" {
  zone_id = aws_route53_zone.sctp_sandbox.zone_id
  name    = "monitoring.ce-grp-1.sctp-sandbox.com"
  type    = "A"

  alias {
    name                   = data.aws_lb.grafana_alb.dns_name
    zone_id                = data.aws_lb.grafana_alb.zone_id
    evaluate_target_health = true
  }
}
🔁 Final Integration Flow
ALB Ingress (Argo CD) exposes Grafana.

Terraform ACM cert validates via DNS.

Terraform creates Route 53 alias to point to ALB.

You access Grafana at:

arduino
https://monitoring.ce-grp-1.sctp-sandbox.com
Let me know if you'd like to auto-detect the ALB name using a label filter or use external data sources from Kubernetes.

# aws-load-balancer-controller-app is already available do we need another ALB-managed Grafana ingress?


✅ Short Answer:
No, you do not need another ALB or controller for Grafana.
You can reuse the existing ALB Ingress Controller (aws-load-balancer-controller) to expose Grafana via ALB, just like your nginx app.

🧠 Why?
The aws-load-balancer-controller can manage multiple Ingress objects across namespaces.

Each Ingress just defines its own alb.ingress.kubernetes.io/certificate-arn, host, and path.

AWS will handle:

Routing for each host (monitoring.ce-grp-1.sctp-sandbox.com)

Certificate via ACM

DNS via Route 53

✅ What You Do Need
Just create a new Ingress in monitoring namespace (via Helm values) like this:

Example: monitoring/values.yaml
yaml
grafana:
  ingress:
    enabled: true
    ingressClassName: alb
    annotations:
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/target-type: ip
      alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:xxxx
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80, "HTTPS": 443}]'
      alb.ingress.kubernetes.io/ssl-redirect: '443'
    hosts:
      - monitoring.ce-grp-1.sctp-sandbox.com
    tls:
      - hosts:
          - monitoring.ce-grp-1.sctp-sandbox.com
        secretName: dummy-tls  # Not actually used by ALB
Terraform Route 53 Alias
As mentioned earlier:

hcl
resource "aws_route53_record" "grafana" {
  zone_id = aws_route53_zone.sctp_sandbox.zone_id
  name    = "monitoring.ce-grp-1.sctp-sandbox.com"
  type    = "A"

  alias {
    name                   = <grafana-ingress-alb-dns>
    zone_id                = <zone_id>
    evaluate_target_health = true
  }
}
✅ Final Result
You’ll access:

App: https://ce-grp-1.sctp-sandbox.com

Grafana: https://monitoring.ce-grp-1.sctp-sandbox.com

Both served via:

✅ 1 ALB

✅ 1 Ingress Controller

✅ Separate Ingress rules with Route 53 + ACM

Let me know if you’d like to template that values.yaml directly into your repo.