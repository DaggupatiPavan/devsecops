# 04. AI/ML & Emerging Tech

## 1. Kubernetes for AI/ML

Running training and inference workloads on K8s.

### GPU Scheduling
- **Device Plugin**: NVIDIA Device Plugin advertises GPU capacity to the scheduler.
- **Resource Request**:
  ```yaml
  resources:
    limits:
      nvidia.com/gpu: 1
  ```
- **MIG (Multi-Instance GPU)**: Partition a single physical A100 GPU into multiple smaller instances for better utilization.

### Kubeflow
The "ML Toolkit for Kubernetes".
- **Notebooks**: JupyterHub.
- **Pipelines**: Argo Workflows for ML pipelines.
- **Training Operators**: PyTorchJob, TFJob.
- **Serving**: KServe (Serverless inference).

## 2. WebAssembly (Wasm)

Wasm is a binary instruction format for a stack-based virtual machine.
- **Why Wasm on K8s?**:
    - **Startup**: Microseconds (vs seconds for containers).
    - **Size**: Tiny binaries.
    - **Security**: Sandboxed by default.
- **Runwasi**: A containerd shim that allows K8s to run Wasm modules directly (bypassing Docker/Linux container overhead).

## 3. Cluster API (CAPI)

"Kubernetes to manage Kubernetes".
- **Concept**: Use CRDs (`Cluster`, `Machine`, `MachineSet`) to provision and manage clusters across providers (AWS, Azure, vSphere).
- **Declarative Lifecycle**: Upgrade a cluster by editing the `MachineDeployment` version field.
- **Providers**:
    - **CAPA**: Cluster API Provider AWS.
    - **CAPZ**: Cluster API Provider Azure.
    - **CAPV**: Cluster API Provider vSphere.
