# EKS Read-Only Access for Non-Admin Users

## Understanding the Architecture

When you created the EKS cluster with your admin credentials, Terraform automatically added your IAM user to the cluster's `aws-auth` ConfigMap with admin permissions. To grant read-only access to a new user, you need to:

1. Ensure the user can authenticate to AWS (IAM level)
2. Map that IAM user to the EKS cluster (aws-auth ConfigMap)
3. Bind Kubernetes RBAC roles to that user (Kubernetes level)

---

## Step-by-Step Implementation

### Step 1: Create the IAM User (if not already created)
The new user needs basic AWS permissions to call EKS APIs. They don't need admin access.

**Terraform code:**
```hcl
resource "aws_iam_user" "readonly_user" {
  name = "eks-readonly-user"
}

resource "aws_iam_access_key" "readonly_user" {
  user = aws_iam_user.readonly_user.name
}

# Minimal policy for EKS access
resource "aws_iam_user_policy" "eks_readonly" {
  name = "eks-readonly-policy"
  user = aws_iam_user.readonly_user.name

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "eks:DescribeCluster",
          "eks:ListClusters"
        ]
        Resource = "*"
      }
    ]
  })
}
```

**Why:** The user needs minimal IAM permissions to describe and list EKS clusters. This is separate from Kubernetes RBAC.

---

### Step 2: Update aws-auth ConfigMap to Map IAM User to Kubernetes
This is the critical bridge between AWS IAM and Kubernetes. You need to add the IAM user to the ConfigMap.

**Using kubectl:**
```bash
kubectl edit configmap aws-auth -n kube-system
```

Add this under the `mapUsers` section:
```yaml
mapUsers: |
  - userarn: arn:aws:iam::ACCOUNT_ID:user/eks-readonly-user
    username: eks-readonly-user
    groups:
      - readonly-group
```

**Using Terraform (recommended):**
```hcl
resource "kubernetes_config_map_v1_data" "aws_auth" {
  metadata {
    name      = "aws-auth"
    namespace = "kube-system"
  }

  data = {
    mapUsers = yamlencode(concat(
      yamldecode(data.kubernetes_config_map_v1.aws_auth.data.mapUsers),
      [
        {
          userarn  = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:user/eks-readonly-user"
          username = "eks-readonly-user"
          groups   = ["readonly-group"]
        }
      ]
    ))
  }

  depends_on = [aws_eks_cluster.main]
}

data "kubernetes_config_map_v1" "aws_auth" {
  metadata {
    name      = "aws-auth"
    namespace = "kube-system"
  }
}

data "aws_caller_identity" "current" {}
```

**Why:** This maps the IAM user ARN to a Kubernetes username and assigns them to a group. The group is then used in RBAC bindings.

---

### Step 3: Create Kubernetes ClusterRole for Read-Only Access
Define what "read-only" means in Kubernetes terms.

**kubectl command:**
```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: readonly-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets", "daemonsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "watch"]
EOF
```

**Terraform:**
```hcl
resource "kubernetes_cluster_role_v1" "readonly" {
  metadata {
    name = "readonly-role"
  }

  rule {
    api_groups = [""]
    resources  = ["pods", "services", "configmaps"]
    verbs      = ["get", "list", "watch"]
  }

  rule {
    api_groups = ["apps"]
    resources  = ["deployments", "statefulsets", "daemonsets"]
    verbs      = ["get", "list", "watch"]
  }

  rule {
    api_groups = ["batch"]
    resources  = ["jobs", "cronjobs"]
    verbs      = ["get", "list", "watch"]
  }
}
```

**Why:** This defines the exact permissions. Adjust resources based on what the user needs to view.

---

### Step 4: Bind the ClusterRole to the User's Group
Connect the role to the group you created in Step 2.

**kubectl command:**
```bash
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: readonly-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: readonly-role
subjects:
- kind: Group
  name: readonly-group
  apiGroup: rbac.authorization.k8s.io
EOF
```

**Terraform:**
```hcl
resource "kubernetes_cluster_role_binding_v1" "readonly" {
  metadata {
    name = "readonly-binding"
  }

  role_ref {
    api_group = "rbac.authorization.k8s.io"
    kind      = "ClusterRole"
    name      = "readonly-role"
  }

  subject {
    kind      = "Group"
    name      = "readonly-group"
    api_group = "rbac.authorization.k8s.io"
  }
}
```

**Why:** This binds the permissions to the group, so any user in that group gets those permissions.

---

### Step 5: User Configuration - How They Access the Cluster

The new user needs to configure their AWS credentials and update their kubeconfig.

**User setup:**
```bash
# 1. Configure AWS credentials (using the access key from Step 1)
aws configure --profile eks-readonly
# Enter: Access Key ID, Secret Access Key, region, output format

# 2. Update kubeconfig
aws eks update-kubeconfig \
  --region us-east-1 \
  --name your-cluster-name \
  --profile eks-readonly

# 3. Test access
kubectl get pods --all-namespaces
```

**Why:** The user's AWS credentials must be configured so they can authenticate to the cluster via the aws-auth ConfigMap.

---

## Validation Steps

```bash
# As the new user, verify read access works
kubectl get pods -A
kubectl get deployments -A
kubectl get services -A

# Verify write access is denied
kubectl create deployment test --image=nginx  # Should fail with "forbidden"
```

---

## Common Gotchas

1. **aws-auth ConfigMap not updated**: If the user can't access the cluster, check that their IAM ARN is in the ConfigMap
2. **Wrong IAM ARN format**: Use `arn:aws:iam::ACCOUNT_ID:user/username`, not role ARN
3. **Credentials not configured**: User must have AWS credentials configured locally
4. **Secrets access**: If you don't want them seeing secrets, remove `secrets` from the ClusterRole
5. **Namespace-specific access**: If you only want access to specific namespaces, use `Role` and `RoleBinding` instead of `ClusterRole` and `ClusterRoleBinding`

---

## Summary of Changes

| Component | What Changes |
|-----------|--------------|
| AWS IAM | Create user + minimal policy |
| aws-auth ConfigMap | Add IAM user mapping |
| Kubernetes RBAC | Create ClusterRole + ClusterRoleBinding |
| User's machine | Configure AWS credentials + update kubeconfig |
