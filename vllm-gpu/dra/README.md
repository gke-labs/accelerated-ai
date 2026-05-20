# Serving with vLLM on GKE Standard using DRA

This recipe demonstrates how to use **Dynamic Resource Allocation (DRA)** to serve a vLLM inference server on GKE Standard. This approach uses the manual driver installation method, providing maximum control over the GPU hardware and resource management lifecycle.

## Why GKE Standard?

While GKE Autopilot offers managed DRA for some networking resources, **DRA for GPUs specifically requires GKE Standard** because manual installation of the DRA driver stack requires root-level privileged access to the node kernel and filesystem (e.g., `/var/lib/cdi`). Autopilot's hardened security model blocks these operations.

## Prerequisites

- **GKE Version:** This recipe requires **GKE v1.36** or later.
- **Release Channel:** You must use the **Rapid** release channel to access the DRA v1 API.
- **Tools:** `kubectl`, `helm`, and `gcloud` must be installed and configured.

---

## Phase 1: Infrastructure Setup

### A. Create the DRA-Enabled Cluster
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
    --machine-type=e2-standard-4
```

### B. Create GPU Node Pools with Mandatory DRA Labels
For DRA to work on GKE Standard, the nodes **must** have the label `nvidia.com/gpu.present=true`. Setting this at the node-pool level ensures it persists across Spot preemptions.

> **Note on GPU Availability:** This recipe uses the `--spot` flag. In many regions, **Preemptible GPU quota** is significantly easier to acquire than standard GPU quota, making Spot instances a more reliable path for provisioning high-demand GPUs like the NVIDIA L4.

```bash
gcloud container node-pools create dra-l4-pool \
    --cluster=$CLUSTER_NAME \
    --region=$REGION \
    --machine-type=g2-standard-4 \
    --accelerator=type=nvidia-l4,count=1 \
    --spot \
    --num-nodes=1 \
    --node-locations=us-central1-a,us-central1-b,us-central1-c \
    --node-labels=nvidia.com/gpu.present=true \
    --node-taints=nvidia.com/gpu=present:NoSchedule
```

---

## Phase 2: The Driver Stack

### A. NVIDIA OS-Level Driver
The OS driver must be loaded before the DRA driver can function. On COS, use the containerized installer.

**Technical Rationale:** The standard GKE installer is programmed to skip nodes that it detects are already "accelerator-ready" by GKE's default standards. We must patch the installer to force it to run, ensuring the underlying kernel modules required for DRA are properly loaded.

```bash
# Apply the installer
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/master/nvidia-driver-installer/cos/daemonset-preloaded.yaml

# FORCE the installer: Overwrites nodeAffinity to bypass GKE's default "skip" logic.
kubectl patch ds nvidia-driver-installer -n kube-system --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/affinity/nodeAffinity/requiredDuringSchedulingIgnoredDuringExecution/nodeSelectorTerms/0/matchExpressions", "value": [{"key": "cloud.google.com/gke-accelerator", "operator": "Exists"}]}]'
```

### B. NVIDIA DRA Driver (via Helm)
The DRA driver bridges the GPU to the Kubernetes DRA API. We must configure it to look in GKE's specific driver path (`/home/kubernetes/bin/nvidia`).

**Technical Rationale (Priority & Quota):** The driver defaults to `system-node-critical` priority. Many GKE projects lack resource quota for this class in non-system namespaces. Removing it allows the driver to schedule using default priority.

**Technical Rationale (Tolerations):** To land the driver pods on our tainted GPU nodes, we must inject tolerations. Since no `kubectl toleration` command exists, we use `patch`.

```bash
git clone https://github.com/kubernetes-sigs/dra-driver-nvidia-gpu.git
cd dra-driver-nvidia-gpu

# Install with GKE-specific driver root
helm upgrade -i --create-namespace --namespace dra-driver-nvidia-gpu dra-driver-nvidia-gpu ./deployments/helm/dra-driver-nvidia-gpu \
    --set gpuResourcesEnabledOverride=true \
    --set nvidiaDriverRoot=/home/kubernetes/bin/nvidia \
    --set image.tag=v0.4.0

# Remove Priority Class: Prevents scheduling failure due to "QuotaExceeded" on system priority.
kubectl patch ds dra-driver-nvidia-gpu-kubelet-plugin -n dra-driver-nvidia-gpu --type='json' -p='[{"op": "remove", "path": "/spec/template/spec/priorityClassName"}]'
kubectl patch deployment dra-driver-nvidia-gpu-controller -n dra-driver-nvidia-gpu --type='json' -p='[{"op": "remove", "path": "/spec/template/spec/priorityClassName"}]'

# Add GPU Tolerations: Allows the driver itself to schedule onto the tainted GPU nodes.
kubectl patch ds dra-driver-nvidia-gpu-kubelet-plugin -n dra-driver-nvidia-gpu --type='json' -p='[{"op": "add", "path": "/spec/template/spec/tolerations/-", "value": {"key": "nvidia.com/gpu", "operator": "Equal", "value": "present", "effect": "NoSchedule"}}]'
kubectl patch deployment dra-driver-nvidia-gpu-controller -n dra-driver-nvidia-gpu --type='json' -p='[{"op": "add", "path": "/spec/template/spec/tolerations/-", "value": {"key": "nvidia.com/gpu", "operator": "Equal", "value": "present", "effect": "NoSchedule"}}]'
```

### C. Fix Control Plane Connectivity (Konnectivity)
**Note:** This step is only required if your cluster's default (non-GPU) node pool is scaled to zero. 

**Technical Rationale:** If GPU nodes are the only nodes running, system components like `konnectivity-agent` (which powers `kubectl logs` and `exec`) must be able to "tolerate" the GPU taints. Without this patch, you will lose the ability to inspect or debug your vLLM pods if the default pool is empty.

```bash
kubectl patch deployment konnectivity-agent -n kube-system --type='json' -p='[{"op": "add", "path": "/spec/template/spec/tolerations/-", "value": {"key": "nvidia.com/gpu", "operator": "Equal", "value": "present", "effect": "NoSchedule"}}]'
```

---

## Phase 3: Workload Provisioning

### A. Device Class and Resource Claim Template
DRA requires a `DeviceClass` to define how resources are requested and a `ResourceClaimTemplate` for the workload to use.

```bash
kubectl apply -f gpu-device-class.yaml
kubectl apply -f gpu-resource-claim-template.yaml
```

### B. Deploy vLLM
Ensure you have a secret named `hf-secret` with your `hf_token`.

```bash
kubectl create secret generic hf-secret --from-literal=hf_token=YOUR_TOKEN_HERE
kubectl apply -f vllm-dra-deployment.yaml
kubectl apply -f vllm-dra-service.yaml
```

### C. Verify Inference
Once the pod is `Running` and the server has started (check logs for "Application startup complete"), you can test the endpoint.

1. **Port-forward the service:**
   ```bash
   kubectl port-forward service/vllm-dra-service 8081:8081
   ```

2. **Send a request:**
   ```bash
   curl -X POST http://localhost:8081/v1/chat/completions \
     -H "Content-Type: application/json" \
     -d '{
       "model": "google/gemma-3-1b-it",
       "messages": [{"role": "user", "content": "What is DRA in Kubernetes?"}],
       "max_tokens": 50
     }'
   ```

---

## Troubleshooting and Verification

- **Verify GPU Availability (Inside Node):** `kubectl exec <pod-name> -- nvidia-smi`
- **Verify GPU Claim:** `kubectl get resourceclaims` (Should show `allocated,reserved`)
- **Check Driver Path:** If the Kubelet plugin fails its init-check, ensure `nvidiaDriverRoot` matches the host path (usually `/home/kubernetes/bin/nvidia` on GKE).
- **Check Device Classes:** `kubectl get deviceclasses` (Ensure `gpu.nvidia.com` exists).
