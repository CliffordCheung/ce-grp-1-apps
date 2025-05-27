
# Create the IRSA for ALB Controller
Use this exact command (based on your AWS account and cluster):

bash
eksctl create iamserviceaccount \
  --cluster ce-grp-1-eks \
  --region us-east-1 \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::255945442255:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

✅ This will:
Create the aws-load-balancer-controller service account
Attach the IAM role
Annotate it for IRSA

