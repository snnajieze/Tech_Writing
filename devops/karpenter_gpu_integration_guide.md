# Integrate GPU Nodes with Karpenter — Quick Guide

**Goal:** When a pod requests a GPU, Karpenter will provision a GPU EC2 node, the NVIDIA plugin will expose the GPU to Kubernetes, and your pod will run.

---

## 🧰 Prerequisites
- An existing EKS cluster (or Kubernetes cluster) with Karpenter installed.
- Karpenter properly configured (IAM, EC2 permissions, discovery tags).
- IAM roles allowing GPU instance types.
- `kubectl` access to the cluster.

---

## 🚀 Quick Summary
1. Allow GPU instance types in Karpenter.
2. Deploy NVIDIA device plugin.
3. Create a pod that requests a GPU.
4. Karpenter provisions a GPU node.
5. NVIDIA plugin exposes the GPU resource.

---

## 🧩 Step-by-Step Setup

### 1️⃣ Allow GPU Instance Types in Karpenter

Create **EC2NodeClass** and **NodePool** manifests:

**ec2nodeclass-gpu.yaml**
```yaml
apiVersion: karpenter.k8s.aws/v1alpha1
kind: EC2NodeClass
metadata:
  name: gpu-nodeclass
spec:
  amiFamily: AL2
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-eks-cluster
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-eks-cluster
  instanceProfile: "KarpenterNodeInstanceProfile-my-eks-cluster"
```

**nodepool-gpu.yaml**
```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: gpu-nodepool
spec:
  template:
    spec:
      nodeClassRef:
        name: gpu-nodeclass
        kind: EC2NodeClass
      requirements:
        - key: "karpenter.k8s.aws/instance-family"
          operator: In
          values: ["g4dn","g5","p3","p4","p5"]
        - key: "kubernetes.io/arch"
          operator: In
          values: ["amd64"]
      taints:
        - key: "nvidia.com/gpu"
          value: "true"
          effect: NoSchedule
  limits:
    cpu: "200"
```

Apply:
```bash
kubectl apply -f ec2nodeclass-gpu.yaml
kubectl apply -f nodepool-gpu.yaml
```

---

### 2️⃣ Install NVIDIA Device Plugin
```bash
kubectl apply -f https://github.com/NVIDIA/k8s-device-plugin/raw/main/nvidia-device-plugin.yml
```

Or use the **NVIDIA GPU Operator** for full driver management.

---

### 3️⃣ Create a GPU-Requesting Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
spec:
  tolerations:
    - key: "nvidia.com/gpu"
      operator: "Exists"
      effect: "NoSchedule"
  containers:
  - name: cuda-sample
    image: nvidia/cuda:12.3.2-base-ubuntu22.04
    command: ["nvidia-smi", "-L"]
    resources:
      limits:
        nvidia.com/gpu: 1
        cpu: "1"
        memory: "1Gi"
```

Apply:
```bash
kubectl apply -f gpu-test.yaml
```

---

### 4️⃣ Watch Karpenter & Node Creation

```bash
kubectl get pods -A
kubectl get nodes -o wide
kubectl describe pod gpu-test
kubectl logs -n karpenter deploy/karpenter
```

You should see a new node (e.g., `g5.xlarge`) created and your pod running.

---

### 5️⃣ Verify GPU Access

```bash
kubectl logs gpu-test
# or
kubectl exec -it gpu-test -- nvidia-smi
```

Expected: GPU info displayed (e.g., driver, memory, CUDA cores).

---

## 🩻 Troubleshooting

| Issue | Check |
|-------|--------|
| Pod stuck Pending | `kubectl describe pod` for scheduling errors |
| Node not created | IAM permissions, subnet tags, or instance type mismatch |
| GPU not visible | NVIDIA plugin or drivers missing |
| Driver errors | Use GPU Operator or official GPU AMIs |

---

## 🧠 Best Practices

- Use **taints/tolerations** to isolate GPU workloads.  
- Keep **dedicated NodePools** for GPUs.  
- Mix **Spot + On-Demand** GPUs for cost efficiency.  
- Use **Karpenter consolidation** to clean idle GPU nodes.  
- Always verify `nvidia.com/gpu` resource appears on node.  
- Monitor GPU metrics for utilization and scaling.

---

## ✅ Final Checklist

- [ ] Karpenter IAM roles correct  
- [ ] GPU families allowed in NodePool  
- [ ] NVIDIA plugin or GPU Operator installed  
- [ ] Pod requests `nvidia.com/gpu`  
- [ ] GPU node visible in cluster  
- [ ] Verified via `nvidia-smi`  

---

## 🧩 Next Steps
- Deploy full GPU Operator for production.
- Add autoscaling policies based on GPU utilization.
- Configure Spot fallback for cost optimization.
