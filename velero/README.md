# 📦 `velero/` – Velero 1.18 Installation & Configuration  

This directory contains everything required to install **Velero 1.18** (the latest stable release at the time of writing) and to configure it for incremental backups of the PostgreSQL 18 PVCs to one or more **S3‑compatible buckets**.  

Velero will:

* **Take incremental snapshots** of the PVCs (thanks to the CSI driver).  
* **Store the snapshots** in the configured S3 bucket(s).  
* **Validate data integrity** (checksums, restore‑test hooks).  
* **Secure the data** (encryption at‑rest via S3, IAM policies, TLS).  
* **Manage the backup lifecycle** (TTL, retention, automatic clean‑up).  

---  

## Table of Contents  

1. [Why Velero 1.18?](#why-velero118)  
2. [Prerequisites](#prerequisites)  
3. [Helm‑based deployment](#helm‑based-deployment)  
4. [Configuration via `values.yaml`](#configuration-via-valuesyaml)  
5. [Creating a backup schedule (incremental)](#creating-a-backup-schedule-incremental)  
6. [Verification & restore](#verification--restore)  
7. [Cleanup](#cleanup)  
8. [Further reading & links](#further‑reading--links)  

---  

## Why Velero 1.18?  

| Feature | Velero 1.18 |
|---------|-------------|
| **Incremental CSI snapshots** | Supported for Ceph‑CSI, AWS EBS, GCE PD, Azure Disk, etc. |
| **Built‑in S3 plugin** | `velero-plugin-for-aws` (works with any S3‑compatible endpoint). |
| **Backup lifecycle policies** | TTL, `--snapshot-volumes`, `--wait`, automatic prune. |
| **Data integrity** | Checksums, optional `restic` integration for file‑level backups. |
| **Security** | TLS for the Velero server, IAM‑based bucket access, optional server‑side encryption (SSE‑S3 / SSE‑KMS). |
| **Extensible** | Supports additional plugins (e.g., `velero-plugin-for-csi`). |

---  

## Prerequisites  

| Requirement|Requirement|Notes|
|---|-----------|-----|
|Kubernetes|v1.34 +|Cluster already provisioned (see `../kubespray`).|
|Helm|v3.12 +|Helm client installed on your workstation.|
|S3 bucket|Existing bucket (AWS, MinIO, Ceph RGW, etc.)|You need an access‑key/secret‑key pair with **PutObject**, **GetObject**, **DeleteObject** permissions.|
|CSI driver|Ceph‑CSI (`sc‑ceph‑rbd`) already installed|Velero will use the CSI snapshot feature.|
|Namespace|`kube-velero` |You may change it in `values.yaml`.|

---  

## Helm‑based deployment  

All the Helm‑related files live in this folder. The **only** thing you normally have to edit is `values.yaml`.  

```bash
# 1️⃣ Add the Velero chart repo
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm repo update

# 2️⃣ Install Velero (or upgrade if already present)
helm upgrade --install velero vmware-tanzu/velero \
  -n kube-velero --create-namespace \
  -f values.yaml
```

> The command creates the `velero` namespace (if it does not exist) and installs the chart using the configuration you supplied in `values.yaml`.  

---  

## Configuration via `values.yaml`  

Below is a **minimal but functional** example. Adjust the fields marked with `<>` to match your environment.

```yaml
# ──────────────────────────────────────────────────────────────
# Helm values for Velero 1.18
# ──────────────────────────────────────────────────────────────
configuration:
  # Provider name – “aws” works for any S3‑compatible endpoint
  provider: aws
  # Backup storage location – you can define several of them
  backupStorageLocation:
    - name: default
      bucket: <YOUR_BUCKET_NAME>
      config:
        region: us-east-1                # any string; required by the plugin
        s3ForcePathStyle: true           # true for MinIO / Ceph RGW
        s3Url: <YOUR_S3_ENDPOINT>        # e.g. https://minio.example.com
        # optional: enable server‑side encryption
        # sse: aws:kms
        # kmsKeyId: <KMS_KEY_ARN>

# Credentials – stored as a secret named “velero” in the velero namespace
credentials:
  secretContents:
    cloud: |
      [default]
      aws_access_key_id = <YOUR_ACCESS_KEY>
      aws_secret_access_key = <YOUR_SECRET_KEY>


  # Example schedule – you can also create schedules via `velero schedule create`
schedule:
  - name: pg-incremental
    schedule: "0 2 * * *"          # every day at 02:00 UTC
    ttl: 720h0m0s                 # 30 days
    includeResources:
      - persistentvolumeclaims
      - pods
      - services
    selector:
      matchLabels:
        app: postgres

---  

## Creating a backup schedule (incremental)  

If you prefer to create schedules **after** the Helm install, use the CLI:

```bash
# Example: daily incremental backup of the PostgreSQL namespace
velero schedule create pg-daily-incremental \
  --schedule="0 2 * * *" \
  --include-namespaces postgresql-integration \
  --selector app=postgres \
  --ttl 720h0m0s          # 30 days
```

Because the CSI plugin is present, Velero will:

1. **Take a snapshot of the PVC** (`postgres-data`).`) using Ceph‑CSI.  
2. **Detect that a previous snapshot exists** and store only the **delta** (incremental).  
3. **Upload the snapshot metadata** and the delta files to the configured S3 bucket.  

The first backup is a full snapshot; subsequent ones are incremental, dramatically reducing storage consumption and network traffic.

---  

## Verification & Restore  

### List existing backups  

```bash
kubectl get backupstoragelocation -A
velero backup get
```


You should see something like:

```
NAME                     STATUS      ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
pg-daily-incremental-1   Completed   0        0          2026-03-20 02:00:12 +00:00      30d       default            app=postgres
```

### Inspect a backup  

```bash
velero backup describe pg-daily-incremental-1 --details
```

Look for the **Snapshot ID** and the **S3 object** listed under “Backup storage location”.

### Restore a backup  

```bash
velero restore create --from-backup pg-daily-incremental-1 \
  --namespace-mappings postgresql-integration=postgresql-integration-restore
```

Velero will:

* Re‑create the PVC (using the same `storageClassName`).‑ceph‑rbd`).  
* Pull the incremental snapshots from S3, apply them in order, and finally attach the restored volume to a new PostgreSQL pod.  

After the restore finishes, verify the data:

```bash
kubectl -n postgresql-integration-restore exec -it \
  $(kubectl -n postgresql-integration-restore get po -l app=postgres -o jsonpath='{.items[0].metadata.name}') \
  -- psql -U admin -c "\l"
```

---  

# Usefull commands

https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

aws --version
sudo cat /root/.aws/config
    [profile ceph]
    output = json
    endpoint_url = http://XXXXXXXXXXX/
    region = us-east-1
sudo cat /root/.aws/credentials
    [ceph]
    aws_access_key_id = XXXXXXXXXXXXXXXXXXXXX
    aws_secret_access_key = YYYYYYYYYYYYYYYYYYYYYYYY

aws --profile=ceph s3 ls
aws --profile=ceph s3 ls --summarize --human-readable --recursive "s3://my-bucket"


# Remove the backup bucket objects (if you want to free S3 storage)
aws s3 rm s3://<YOUR_BUCKET_NAME>/velero/ --recursive
# or, for MinIO:
mc rm --recursive myminio/<YOUR_BUCKET_NAME>/velero/
```

---  

## Further Reading & Links  

| Topic | Link |
|-------|------|
| Velero official documentation (v1.18) | <https://velero.io/docs/v1.18/> |
| CSI snapshot support | <https://velero.io/docs/v1.18/csi/> |
| Velero Helm chart | <https://github.com/vmware-tanzu/helm-charts/tree/main/charts/velero> |
| Ceph‑CSI driver (used for PVCs) | <https://github.com/ceph/ceph-csi> |
| S3‑compatible storage (MinIO, Ceph RGW) | <https://min.io/> / <https://docs.ceph.com/en/latest/radosgw/> |
| Security best practices for Velero | <https://velero.io/docs/v1.18/security/> |
| Backup lifecycle management | <https://velero.io/docs/v1.18/backup-storage/> |

---  

### 🎉 You now have a fully‑functional Velero 1.18 installation that:

* Takes **incremental** snapshots of your PostgreSQL PVCs.  
* Stores them securely in an **S3‑compatible bucket**.  
* Handles **integrity checks**, **encryption**, and **automatic TTL‑based pruning** cleanup.  

Proceed to the `../postgresql` folder to see the PostgreSQL deployment that Velero will back up, or start creating backup schedules right away with the `velero schedule create` command shown above. Happy backing‑up!  
