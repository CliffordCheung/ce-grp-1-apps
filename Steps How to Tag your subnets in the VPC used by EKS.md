Tag your subnets in the VPC used by EKS
🛠 You need to tag your subnets like this:

For public subnets (used for LoadBalancer, ALB):
text
Key:   kubernetes.io/role/elb
Value: 1

For private subnets (used for internal ALB):
text
Key:   kubernetes.io/role/internal-elb
Value: 1

✅ Find your VPC ID and Subnet IDs
Run:

bash
aws eks describe-cluster --name ce-grp-1-eks --region us-east-1 \
  --query "cluster.resourcesVpcConfig.subnetIds" --output text

Then tag the correct subnets using the AWS CLI:

bash
aws ec2 create-tags \
  --resources <subnet-id-1> <subnet-id-2> \
  --tags Key=kubernetes.io/role/elb,Value=1

aws ec2 create-tags \
--resources subnet-0ede2f806999dafd5 subnet-0e0248398281b091b subnet-002cf7ce617d6ac46 \
--tags Key=kubernetes.io/role/elb,Value=1

Do this for the public subnets where you want the ALB/NLB to be created.

If using internal ALB (private exposure only), use:

bash
--tags Key=kubernetes.io/role/internal-elb,Value=1

✅ Final Step

Once tagged:
Wait ~30–60 seconds
Re-sync the Argo CD app (optional):

bash
argocd app sync nginx-prod-webapp-aalimsee

Then check:

bash
kubectl get svc -n prod

You should see the EXTERNAL-IP populate with an ALB/NLB hostname.


```
lim@MacBookAir ce-grp-1-apps % argocd app sync nginx-prod-webapp-aalimsee


Name:               argocd/nginx-prod-webapp-aalimsee
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          prod
URL:                https://localhost:8080/applications/nginx-prod-webapp-aalimsee
Source:
- Repo:             https://github.com/aalimsee/ce-grp-1-apps.git
  Target:           HEAD
  Path:             nginx
SyncWindow:         Sync Allowed
Sync Policy:        Automated (Prune)
Sync Status:        Synced to HEAD (601cdb3)
Health Status:      Progressing

Operation:          Sync
Sync Revision:      601cdb3a937039fc38c743833c9577d482d1a755
Phase:              Succeeded
Start:              2025-05-27 11:34:27 +0800 +08
Finished:           2025-05-27 11:34:28 +0800 +08
Duration:           1s
Message:            successfully synced (all tasks run)

GROUP  KIND        NAMESPACE  NAME                        STATUS  HEALTH       HOOK  MESSAGE
       Service     prod       nginx-prod-webapp-aalimsee  Synced  Progressing        service/nginx-prod-webapp-aalimsee unchanged
apps   Deployment  prod       nginx-prod-webapp-aalimsee  Synced  Healthy            deployment.apps/nginx-prod-webapp-aalimsee unchanged
lim@MacBookAir ce-grp-1-apps % 
```

Perfect sync,  — Argo CD reports:

✅ App synced successfully
✅ Deployment is healthy
⚠️ Service is still in Progressing health state — this means the LoadBalancer service is waiting for the AWS ALB to be provisioned.

```
lim@MacBookAir ce-grp-1-apps % kubectl get svc -n prod
NAME                         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-prod-webapp-aalimsee   LoadBalancer   10.100.247.9   <pending>     80:31675/TCP   16m
lim@MacBookAir ce-grp-1-apps % 
```

Thanks  — the EXTERNAL-IP is still <pending>, which confirms:

❌ The AWS Load Balancer Controller still cannot find valid subnets in your VPC to deploy the ALB/NLB.

# Determine which subnets is public
aaronlim@MacBookAir ce-grp-1-apps % aws eks describe-cluster \
  --name ce-grp-1-eks \
  --region us-east-1 \
  --query "cluster.resourcesVpcConfig.subnetIds" \
  --output text

subnet-0ede2f806999dafd5        subnet-0e0248398281b091b        subnet-002cf7ce617d6ac46
aaronlim@MacBookAir ce-grp-1-apps % 
aaronlim@MacBookAir ce-grp-1-apps % aws ec2 describe-subnets \
  --subnet-ids subnet-0ede2f806999dafd5 subnet-0e0248398281b091b subnet-002cf7ce617d6ac46 \ 
  --query "Subnets[*].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,Public:MapPublicIpOnLaunch}" \
  --output table

---------------------------------------------------------------------
|                          DescribeSubnets                          |
+------------+--------------+----------------------------+----------+
|     AZ     |    CIDR      |            ID              | Public   |
+------------+--------------+----------------------------+----------+
|  us-east-1b|  14.0.2.0/24 |  subnet-0ede2f806999dafd5  |  False   |
|  us-east-1a|  14.0.1.0/24 |  subnet-0e0248398281b091b  |  False   |
|  us-east-1c|  14.0.3.0/24 |  subnet-002cf7ce617d6ac46  |  False   |
+------------+--------------+----------------------------+----------+
aaronlim@MacBookAir ce-grp-1-apps % 


# Test the NGINX endpoint
bash
curl http://k8s-prod-nginxpro-08bb8a3872-0694b6ab7711d1d7.elb.us-east-1.amazonaws.com
Or open it in your browser — you should see the default NGINX welcome page.


# Results of Argo ALB deployment
```
aaronlim@MacBookAir ce-grp-1-apps % argocd app sync nginx-prod-webapp-aalimsee


Name:               argocd/nginx-prod-webapp-aalimsee
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          prod
URL:                https://localhost:8080/applications/nginx-prod-webapp-aalimsee
Source:
- Repo:             https://github.com/aalimsee/ce-grp-1-apps.git
  Target:           HEAD
  Path:             nginx
SyncWindow:         Sync Allowed
Sync Policy:        Automated (Prune)
Sync Status:        Synced to HEAD (601cdb3)
Health Status:      Healthy

Operation:          Sync
Sync Revision:      601cdb3a937039fc38c743833c9577d482d1a755
Phase:              Succeeded
Start:              2025-05-27 12:28:49 +0800 +08
Finished:           2025-05-27 12:28:49 +0800 +08
Duration:           0s
Message:            successfully synced (all tasks run)

GROUP  KIND        NAMESPACE  NAME                        STATUS  HEALTH   HOOK  MESSAGE
       Service     prod       nginx-prod-webapp-aalimsee  Synced  Healthy        service/nginx-prod-webapp-aalimsee unchanged
apps   Deployment  prod       nginx-prod-webapp-aalimsee  Synced  Healthy        deployment.apps/nginx-prod-webapp-aalimsee unchanged
aaronlim@MacBookAir ce-grp-1-apps % 
aaronlim@MacBookAir ce-grp-1-apps % 
aaronlim@MacBookAir ce-grp-1-apps % 
aaronlim@MacBookAir ce-grp-1-apps % kubectl get svc -n prod
NAME                         TYPE           CLUSTER-IP     EXTERNAL-IP                                                                 PORT(S)        AGE
nginx-prod-webapp-aalimsee   LoadBalancer   10.100.247.9   k8s-prod-nginxpro-08bb8a3872-0694b6ab7711d1d7.elb.us-east-1.amazonaws.com   80:31675/TCP   70m
aaronlim@MacBookAir ce-grp-1-apps % 
```
