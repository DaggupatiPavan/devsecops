# 03. Advanced Security & Operations

## 1. Runtime Security (Falco)

Falco is a cloud-native runtime security tool. It parses Linux system calls from the kernel at runtime to assert the stream against a powerful rule engine.

### How it works
- **Driver**: Kernel module or eBPF probe captures syscalls.
- **Engine**: Matches syscalls against rules (YAML).
- **Alerts**: Sends alerts to Slack, Fluentd, etc.

### Example Rule
```yaml
- rule: Shell in Container
  desc: Detect bash running inside a container
  condition: container.id != host and proc.name = bash
  output: Shell spawned in container (user=%user.name container_id=%container.id)
  priority: WARNING
```

## 2. Etcd Internals & Backup

Etcd is the brain of Kubernetes. Losing it means losing the cluster.

### Internals
- **Raft Consensus**: Leader election and log replication.
- **Storage**: MVCC (Multi-Version Concurrency Control) key-value store.
- **Compaction**: Old versions of keys must be compacted to save space.

### Backup & Restore
- **Snapshot**: `etcdctl snapshot save backup.db`
- **Restore**: `etcdctl snapshot restore backup.db --data-dir /var/lib/etcd-new`
- **Disaster Recovery**: If quorum is lost, you must restore from snapshot on a new cluster.

## 3. Secrets Encryption (KMS)

By default, Secrets are base64 encoded in etcd.
- **EncryptionConfiguration**: Encrypts secrets at rest in etcd.
- **KMS Provider**: Uses an external Key Management Service (AWS KMS, Google Cloud KMS) to manage the encryption keys (Envelope Encryption).

## 4. Backup & DR (Velero)

Velero is the standard for K8s backup.
- **Backs up**:
    1.  **Kubernetes Objects**: YAML manifests (stored in S3/MinIO).
    2.  **Persistent Volumes**: Snapshots (via cloud provider API or Restic/Kopia).
- **Use Case**: Disaster Recovery, Cluster Migration, Replication.
