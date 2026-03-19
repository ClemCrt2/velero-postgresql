# 📦 velero‑postgresql‑lab  

**Goal** – This repository demonstrates, step‑by‑step, how to back up a **PostgreSQL 18** database running on a Kubernetes cluster with **Velero**, and then ship those backups to an **S3** bucket (AWS, MinIO, or any S3‑compatible service).  
This lab is aimed at DevOps / SRE teams that want to:

* provision a Kubernetes cluster with **Kubespray**  
* deploy PostgreSQL 18 via a StatefulSet (with a persistent PVC)  
* configure **Velero** to snapshot the PVCs and push the snapshots to S3  
* optionally view everything through **Argo CD** (GitOps)  

---

## 📚 Table of Contents  

1. [Overall Architecture](#overall-architecture)  
2. [Prerequisites](#prerequisites)  
3. [Repository Layout](#repository-layout)  
4. [Step‑by‑step Deployment](#step-by-step-deployment)  
5. [Testing & Validation](#testing--validation)  
6. [Contributing](#contributing)  
7. [License](#license)  

---

## Overall Architecture  

```
+-------------------+          +-------------------+          +-------------------+
|   K8s Cluster     |          |   Velero          |          |   S3 Bucket       |
| (Kubespray)       |  <--->   | (CRDs + plugins)  |  --->    | (AWS/MinIO/etc)  |
|                   |          |                   |          |                   |
|  ├─ PostgreSQL    |          |  ├─ Backup CR    |          |  └─ .tar objects |
|  │   (StatefulSet)│          |  └─ Schedule     |          |                   |
+-------------------+          +-------------------+          +-------------------+
```

* **Kubespray** creates the cluster (with CNI Cilium, etcd on hosts, containerd).  
* **PostgreSQL 18** runs in a `StatefulSet` with a `PersistentVolumeClaim` for data, installed with argocd.  
* **Velero** (installed via Helm) takes snapshots of the PVCs, compresses them, and pushes them to the configured S3 bucket.  
* An optional **Argo CD** `Application` keeps the PostgreSQL manifests in sync with the `postgresql/` directory.  

---

## Prerequisites  

| Component  | Remarks |
|-----------|---------|
| **Linux/macOS** | Tested on Debian 12|
| **containerd** | Installed with Kubespray |
| **kubectl** | 1.34+ | |
| **ansible** | 2.14+ | Used by Kubespray |
| **Helm** | 3.12+ | For installing Velero |
| **AWS CLI** | 2.x | To inspect the S3 bucket |
| **Argo CD** (optional) | 2.8+ | If you want the GitOps part and installed Postgresql with it|
| **Access to an S3 bucket** | Access‑key/secret‑key |

> **Note**: The lab also works with a local S3‑compatible store (MinIO or CEPH) so you don’t need a real AWS account.  

---

## Repository Layout  

```
.
├── kubespray/                 # Ansible inventory & vars to spin up the K8s cluster
│   ├── group_vars/
│   │   ├── all/
│   │   │   ├── all.yml
│   │   │   ├── containerd.yml
│   │   │   └── etcd.yml
│   │   └── k8s_cluster/
│   │       ├── addons.yml
│   │       ├── k8s-cluster.yml
│   │       ├── k8s-net-cilium.yml
│   │       ├── k8s-net-kube-router.yml
│   │       └── kube_control_plane.yml
│   ├── inventory.ini          # Ansible inventory (master + workers)
│   └── README.md              # 👉 [kubespray/README.md](kubespray/README.md)
│
├── postgresql/                # Manifests for PostgreSQL 18
│   ├── argocd-postgresql-integration.yaml
│   ├── manifests/
│   │   ├── configmap.yaml
│   │   ├── pvc-backup.yaml
│   │   ├── pvc.yaml
│   │   ├── service.yaml
│   │   └── statefulset.yaml
│   └── README.md              # 👉 [postgresql/README.md](postgresql/README.md)
│
├── velero/                    # Helm values & Velero documentation
│   ├── values.yaml            # Custom Helm values
│   └── README.md              # 👉 [velero/README.md](velero/README.md)
│
└── README.md                  # <‑ You are here!
```

Each sub‑directory contains its own `README.md` that explains the specific part:

* **kubespray/** – how to create the cluster, pick a CNI, tweak settings.  
* **postgresql/** – deploying deployment of PostgreSQL 18, backup PVC, optional Argo CD integration.  
* **velero/** – Helm installation, S3 plugin configuration, backup schedule.  

---

## Step‑by‑step Deployment  

> The commands below are a concise overview; for full details, see the individual READMEs.

### 1️⃣ Provision the Kubernetes Cluster with kubespray

```bash
git clone git clone https://github.com/kubernetes-sigs/kubespray
cd kubespray/
git checkout release-2.30

cd ..
git clone https://github.com/ClemCrt2/velero-postgresql-lab.git
git clone 
cd velero-postgresql-lab/kubespray

# Install Ansible dependencies
pip install -r requirements.txt

# Edit inventory.ini to match your infrastructure (IP, user, SSH key)
vi inventory.ini

# Run the playbook
cd ../../kubespray
ansible-playbook -i velero-postgresql-lab/kubespray/inventory.ini cluster.yml -b
```

After the run, `kubectl get nodes` should list all nodes in the `Ready` state.

### 2️⃣ Deploy PostgreSQL 18  

```bash
cd ../postgresql
kubectl apply -f manifests/
# (or via Argo CD)
kubectl apply -f argocd-postgresql-integration.yaml
```

* The `StatefulSet` creates a PVC called `postgres-data`.  
* A dedicated PVC (`pvc-backup.yaml`) will be used by Velero to store daily snapshots.

### 3️⃣ Install Velero (Helm)  

```bash
cd ../velero
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm repo update

helm install velero vmware-tanzu/velero \
  -n kube-velero --create-namespace \
  -f values.yaml
```

`values.yaml` contains a minimal S3 configuration, for example:

```yaml
configuration:
  provider: aws
  backupStorageLocation:
    name: default
    bucket: <YOUR_BUCKET>
    config:
      region: us-east-1
      s3ForcePathStyle: true          # true for MinIO
      s3Url: https://minio.example.com # MinIO endpoint (or AWS)
credentials:
  secretContents:
    cloud: |
      [default]
      aws_access_key_id=YOUR_ACCESS_KEY
      aws_secret_access_key=YOUR_SECRET_KEY
```

### 4️⃣ Create a Backup Schedule  

```bash
velero schedule create pg-backup \
  --schedule="0 2 * * *" \
  --include-resources=pods,services,persistentvolumeclaims \
  --selector app=postgresql
```

The schedule above triggers a backup every day at 02:00 UTC.

### 5️⃣ Verify Backups  

```bash
velero backup get
# or directly in the S3 bucket
aws s3 ls s3://<bucket>/velero/
```

You should see `velero-*.tar.gz` objects in the bucket.

### 6️⃣ (Optional) Add Argo CD  

```bash
kubectl apply -f argocd-postgresql-integration.yaml
# Open the Argo CD UI, create an Application, and sync it.
```

---

## Testing & Validation  

| Test | Command | Expected outcome |
|------|---------|------------------|
| **Cluster** | `kubectl get nodes` | All nodes listed as `Ready` |
| **PostgreSQL** | `kubectl get pods -l app=postgresql` | One pod in `Running` state |
| **Velero** | `velero backup get` | At least one recent backup appears |
| **S3** | `aws s3 ls s3://<bucket>/velero/` | List of `velero-*.tar.gz` files |
| **Restore** | `velero restore create --from-backup <backup‑name>` | PVC restored, PostgreSQL pod restarts successfully |

---

## Contributing  

Contributions are welcome!  

---

## License  

This project is licensed under the **MIT License**. See the `LICENSE` file at the repository root.

---

### 🎉  

You now have everything you need to reproduce the PostgreSQL 18 backup lab with Velero and S3. Feel free to open issues if you run into trouble or have ideas for improvements! 🚀
