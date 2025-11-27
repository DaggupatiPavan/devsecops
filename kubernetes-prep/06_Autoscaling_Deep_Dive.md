# 06. Autoscaling Deep Dive

## 1. Horizontal Pod Autoscaler (HPA)

Automatically scales the number of Pods based on observed CPU utilization or custom metrics.

### How it works
1.  **Metrics Server**: Aggregates resource usage (CPU/Memory) from Kubelets.
2.  **HPA Controller**: Queries Metrics Server every 15s (default).
3.  **Calculation**: `TargetNumPods = Ceil(CurrentPods * CurrentMetric / TargetMetric)`

### Example
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

## 2. Vertical Pod Autoscaler (VPA)

Automatically adjusts the CPU and Memory requests/limits for your Pods.
- **Recommender**: Monitors history and calculates recommendations.
- **Updater**: Evicts pods that need updates.
- **Admission Controller**: Mutates new pods with recommended resources.

> [!WARNING]
> VPA and HPA should generally NOT be used together on CPU/Memory, as they can conflict.

## 3. Cluster Autoscaler (CA)

Automatically resizes the underlying node pool.

### Scale Up
Triggered when there are **Pending Pods** that failed scheduling due to insufficient resources.
- CA talks to Cloud Provider (AWS ASG, GKE Node Pool) to provision a new node.

### Scale Down
Triggered when a node is **Underutilized** (e.g., < 50% requested) for a specific time (e.g., 10m).
- CA evicts pods to other nodes and terminates the empty node.

## 4. KEDA (Kubernetes Event-driven Autoscaling)

KEDA allows you to scale based on external events (Kafka topics, SQS queues, Prometheus metrics) rather than just CPU/Memory.

### Architecture
- **Agent**: Activates/Deactivates deployments (scales from 0 to 1 and back).
- **Metrics Adapter**: Exposes external metrics to the HPA (scales from 1 to N).

### Example: Scale on SQS Queue Length
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: aws-sqs-scaledobject
spec:
  scaleTargetRef:
    name: my-deployment
  minReplicaCount: 0  # Can scale to zero!
  maxReplicaCount: 5
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/123/my-queue
      queueLength: "5" # Target 5 messages per pod
      awsRegion: "us-east-1"
```
