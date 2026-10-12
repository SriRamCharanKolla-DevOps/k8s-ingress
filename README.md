Install OIDC provider
```
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster roboshop \
    --approve
```

Create IAM Policy
```
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v3.2.1/docs/install/iam_policy.json
```
```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json
```

Create service account
```
eksctl create iamserviceaccount \
--cluster=roboshop \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--region us-east-1 \
--approve
```

Install drivers
```
helm repo add eks https://aws.github.io/eks-charts
```

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=roboshop --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
```

---

## 📚 Common Mistakes & Learnings Reference

For complete root cause analyses, debugging guides, and error resolutions encountered during this setup, refer to:
👉 **[learnings.md](./learnings.md)**

### Quick Summary of Top Issues Solved:
1. **Deployment `spec.selector.template` Indentation Error:**
   * `template:` must be a direct sibling of `selector:` under `spec:`, not nested inside `selector:`.
2. **Ingress `spec.rules[0].backend` Hierarchy Error:**
   * In `networking.k8s.io/v1`, `paths`, `path`, and `backend` must be nested under `http:`.
3. **ALB 0 Listeners & `UnsupportedCertificate` Error:**
   * When using `alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}]'`, the ACM Certificate **must be in `Issued` status**.
   * If it is in `Pending validation`, AWS ELB will fail to create the listener and the ALB will have 0 listeners. Complete DNS CNAME validation in ACM/Route53.
4. **Reconciliation Lag After Certificate is `Issued`:**
   * Kubernetes `describe events` are cached for 1 hour; old errors don't vanish immediately.
   * AWS Load Balancer Controller doesn't watch AWS ACM directly. Re-trigger reconciliation with `kubectl annotate ingress app1 reconcile="$(date +%s)" --overwrite`.
5. **Route 53 Alias Routing:**
   * Create an `A` record with Alias pointing to the ALB DNS name so users can open `https://<domain>` with a valid SSL lock.