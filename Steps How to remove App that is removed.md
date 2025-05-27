
# Issue and Fix
```
aaronlim@MacBookAir ce-grp-1-apps % argocd app get apps

{"level":"fatal","msg":"rpc error: code = InvalidArgument desc = app is not allowed in project \"ingress-aalimsee\", or the project does not exist","time":"2025-05-27T21:58:08+08:00"}
aaronlim@MacBookAir ce-grp-1-apps % argocd app delete apps --cascade

Are you sure you want to delete 'apps' and all its resources? [y/n] y
{"level":"fatal","msg":"rpc error: code = InvalidArgument desc = app is not allowed in project \"ingress-aalimsee\", or the project does not exist","time":"2025-05-27T21:58:49+08:00"}
aaronlim@MacBookAir ce-grp-1-apps % ```



This is a tricky but known issue: Argo CD won’t let you even delete an application if it references a non-existent project.

❌ rpc error: ... app is not allowed in project "ingress-aalimsee"

✅ Solution: Patch the Application's Project to "default" (via kubectl)
Since Argo CD stores applications as Kubernetes CRs (Custom Resources), you can patch it manually.

🛠 Step 1: Patch the apps Application
bash
kubectl patch application apps -n argocd --type merge -p '{"spec":{"project":"default"}}'

🧹 Step 2: Delete it after patching
Once patched, now delete it cleanly:

bash
argocd app delete apps --cascade

Let me know once you’ve run that — or if you'd prefer I give you the full kubectl delete method too (bypasses Argo CD CLI).