# Serving with vLLM on GKE Standard using DRA

This recipe demonstrates how to use **Dynamic Resource Allocation (DRA)** to serve a vLLM inference server on GKE Standard. This approach uses the manual driver installation method, providing maximum control over the GPU hardware and resource management lifecycle.

## Why GKE Standard?

While GKE Autopilot offers managed DRA for some networking resources, **DRA for GPUs specifically requires GKE Standard** because manual installation of the DRA driver stack requires root-level privileged access to the node kernel and filesystem (e.g., `/var/lib/cdi`). Autopilot's hardened security model blocks these operations.

## Prerequisites

- **GKE Version:** This recipe requires **GKE v1.36** or later.
- **Release Channel:** You must use the **Rapid** release channel to access the DRA v1 API.
- **Tools:** `kubectl`, `helm`, and `gcloud` must be installed and configured.

## 1. Create the DRA-Enabled Cluster

Create a GKE Standard cluster in the Rapid channel.

```bash
export CLUSTER_NAME="vllm-dra-standard-cluster"
export REGION="us-central1"
export PROJECT_ID=$(gcloud config get-value project)

gcloud container clusters create $CLUSTER_NAME \
    --region=$REGION \
    --release-channel=rapid \
    --cluster-version=1.36 \
    --num-nodes=1 \
    --machine-type=e2-standard-4 \
    --workload-pool=$PROJECT_ID.svc.id.goog
```

## 2. Create GPU Node Pools with Mandatory DRA Labels

For DRA to work on GKE Standard, the nodes **must** have the label `nvidia.com/gpu.present=true`. Setting this at the node-pool level ensures it persists across Spot preemptions.

### NVIDIA L4 GPU Pool
```bash
gcloud container node-pools create dra-l4-pool \
    --cluster=$CLUSTER_NAME \
    --region=$REGION \
    --machine-type=g2-standard-4 \
    --accelerator=type=nvidia-l4,count=1 \
    --spot \
    --num-nodes=1 \
    --node-locations=us-central1-a \
    --node-labels=nvidia.com/gpu.present=true \
    --node-taints=nvidia.com/gpu=present:NoSchedule
```

## 3. Install the NVIDIA Stack (Two-Phase Installation)

### Phase 1: NVIDIA OS-Level Driver
The OS driver must be loaded before the DRA driver can function. On COS, use the containerized installer.

**Critical Note:** The installer defaults to skipping nodes with GKE default drivers. We must patch it to force installation.

```bash
# Apply the installer
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/master/nvidia-driver-installer/cos/daemonset-preloaded.yaml

# FORCE the installer to run even if GKE default labels are present
kubectl patch ds nvidia-driver-installer -n kube-system --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/affinity/nodeAffinity/requiredDuringSchedulingIgnoredDuringExecution/nodeSelectorTerms/0/matchExpressions", "value": [{"key": "cloud.google.com/gke-accelerator", "operator": "Exists"}]}]'
```

### Phase 2: NVIDIA DRA Driver (via Helm)
The DRA driver bridges the GPU to the Kubernetes DRA API. We must configure it to look in GKE's specific driver path (`/home/kubernetes/bin/nvidia`).

1. **Clone and Install:**
   ```bash
   git clone https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu.git
   cd dra-driver-nvidia-gpu
   
   # Install with GKE-specific driver root
   helm upgrade -i --create-namespace --namespace dra-driver-nvidia-gpu dra-driver-nvidia-gpu ./deployments/helm/dra-driver-nvidia-gpu \
       --set gpuResourcesEnabledOverride=true \
       --set nvidiaDriverRoot=/home/kubernetes/bin/nvidia \
       --set image.tag=v0.4.0
   ```

2. **Patch for Quota and Taints:**
   GKE Standard projects often lack quota for the `system-node-critical` priority class used by the driver. We must also add tolerations for our GPU taints.

   ```bash
   # Remove Priority Class (to avoid quota issues)
   kubectl patch ds dra-driver-nvidia-gpu-kubelet-plugin -n dra-driver-nvidia-gpu --type='json' -p='[{"op": "remove", "path": "/spec/template/spec/priorityClassName"}]'
   kubectl patch deployment dra-driver-nvidia-gpu-controller -n dra-driver-nvidia-gpu --type='json' -p='[{"op": "remove", "path": "/spec/template/spec/priorityClassName"}]'

   # Add GPU Tolerations
   kubectl patch ds dra-driver-nvidia-gpu-kubelet-plugin -n dra-driver-nvidia-gpu --type='json' -p='[{"op": "add", "path": "/spec/template/spec/tolerations/-", "value": {"key": "nvidia.com/gpu", "operator": "Equal", "value": "present", "effect": "NoSchedule"}}]'
   kubectl patch deployment dra-driver-nvidia-gpu-controller -n dra-driver-nvidia-gpu --type='json' -p='[{"op": "add", "path": "/spec/template/spec/tolerations/-", "value": {"key": "nvidia.com/gpu", "operator": "Equal", "value": "present", "effect": "NoSchedule"}}]'
   ```

## 4. Fix Control Plane Connectivity (Konnectivity)

On clusters where the default pool is scaled to 0, the `konnectivity-agent` pods must be allowed to run on GPU nodes to enable `kubectl logs` and `exec`.

```bash
kubectl patch deployment konnectivity-agent -n kube-system --type='json' -p='[{"op": "add", "path": "/spec/template/spec/tolerations/-", "value": {"key": "nvidia.com/gpu", "operator": "Equal", "value": "present", "effect": "NoSchedule"}}]'
```

## 5. Deploy the vLLM Workload

### Step 1: Resource Claim Template (v1 API)
```bash
kubectl apply -f gpu-resource-claim-template.yaml
```

### Step 2: Hugging Face Secret
Ensure you have a secret named `hf-secret` with your `hf_token`.
```bash
kubectl create secret generic hf-secret --from-literal=hf_token=YOUR_TOKEN_HERE
```

### Step 3: Deploy vLLM
```bash
kubectl apply -f vllm-dra-deployment.yaml
kubectl apply -f vllm-dra-service.yaml
```

## 6. Troubleshooting and Verification

- **Verify GPU Claim:** `kubectl get resourceclaims` (Should be `allocated,reserved`)
- **Verify GPU in Container:** `kubectl exec <pod-name> -- nvidia-smi`
- **Check Driver Path:** If the Kubelet plugin fails its init-check, ensure `nvidiaDriverRoot` matches the host path (usually `/home/kubernetes/bin/nvidia` on GKE).
