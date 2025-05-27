Let's move forward with the Ingress + HTTPS + Route 53 setup for your EKS application.

✅ Goal Recap
You want:

🔒 HTTPS via ACM certificate

🌐 Custom domain like ce-grp-1.sctp-sandbox.com via Route 53

🌉 ALB Ingress controller to route traffic to your NGINX app

📦 Prerequisites Checklist (Already Done or Confirmed)
✅ EKS cluster is running
✅ AWS Load Balancer Controller installed
✅ Public subnets properly tagged (kubernetes.io/role/elb = 1)
✅ ACM cert issued for ce-grp-1.sctp-sandbox.com in us-east-1
✅ Route 53 hosted zone for sctp-sandbox.com exists

✅ Step 1: Create the Ingress YAML
Save this as nginx/ingress.yaml:

yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: prod
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:YOUR-CERT-ID
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
    secretName: nginx-tls  # This name is symbolic; ACM manages the cert
🔁 Replace YOUR-CERT-ID with your ACM certificate ARN.

✅ Step 2: Add ingress.yaml to Git and Sync
bash
git add nginx/ingress.yaml
git commit -m "Add ALB Ingress with ACM HTTPS support"
git push
argocd app sync nginx-prod-webapp-aalimsee
✅ This will create an internet-facing ALB listener for HTTPS and connect it to your NGINX service.

✅ Step 3: Set Up Route 53 DNS
Go to your Route 53 hosted zone for sctp-sandbox.com and create a Record:

Name: ce-grp-1

Type: A (Alias to ALB)

Alias target: Select the ALB DNS from the Ingress output (e.g., k8s-prod-nginxingr...elb.amazonaws.com)

Routing policy: Simple

TTL: Default

✅ Step 4: Test It
bash
curl -I https://ce-grp-1.sctp-sandbox.com
You should get:

http
HTTP/2 200
...
Or open in browser:
https://ce-grp-1.sctp-sandbox.com

✅ Want me to generate Argo CD Application for the Ingress too?
Let me know if you:

Want to manage the Ingress as a separate app

Need to install cert-manager (for Let’s Encrypt instead of ACM)

Want YAML templated for multiple apps using Ingress

