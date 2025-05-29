

Next Steps (Final Layer: HTTPS Access)
Update your Ingress YAML (if not already done) to use the ACM cert:

yaml
alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:your-account-id:certificate/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

You can get it from:

bash
terraform output

or

bash
aws acm list-certificates

Confirm ALB is serving HTTPS

Run:

bash
kubectl get ingress -n prod

Then test:

bash
curl -I https://ce-grp-1.sctp-sandbox.com
You should get a HTTP/2 200 OK (or 301/302 if redirect).

✅ You’re now live with:

Internet-facing ALB Ingress

ACM-managed HTTPS TLS

Route 53 domain-based access

Argo CD GitOps syncing all infrastructure + app YAML

