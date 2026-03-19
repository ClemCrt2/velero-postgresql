# 📂 `postgresql/` – PostgreSQL 18 Deployment & Velero Backup Preparation  

This sub‑directory contains everything you need to:

* **Deploy PostgreSQL 18** on a functional Kubernetes cluster.  
* Use **Ceph‑CSI** (`sc‑ceph‑rbd`) as the persistent‑storage driver (PVC of 50 Gi).  
* Create a **dedicated backup PVC** that will be used by Velero.  
* Configure a **`backup_user`** (replication rights) and adjust `pg_hba.conf` so Velero can connect in *trust* mode (no password).  

---  

## 1️⃣ Quick Context  

| Item | Value |
|------|-------|
| **Namespace** | `postgresql-integration` |
| **PostgreSQL version** | 18 (official image `postgres:18.3`) |
| **Primary PVC** | 50 Gi, `storageClassName: sc-ceph-rbd` (Ceph‑CSI) |
| **Backup PVC** | 50 Gi (or larger, according to your retention needs), same `storageClassName` |
| **Ceph‑CSI driver** | <https://github.com/ceph/ceph-csi> (already deployed and running in the cluster, with storageclass available) |
| **Backup user** | `backup_user` – role `REPLICATION` + `LOGIN` |
| **`pg_hba.conf` entry** | `host all backup_user 0.0.0.0/0 trust` (or the Velero pod’s CID IP) |


> Statefulset manifest include velero pre-hook command to snapshot all database before uploading data to S3.
> The pre-hook timeout need to be adjust to the size of the DB. more than 30secondes for 5Go databases. This timeout don't put the base on read-only mode so you can put 500m if you like

---  

## 2️⃣ Directory Layout  

```
postgresql/
├── argocd-postgresql-integration.yaml   # Optional Argo CD manifest
├── manifests/
│   ├── configmap.yaml                   # Custom pg_hba.conf & postgresql.conf
│   ├── pvc.yaml                         # Primary PVC
│   ├── pvc-backup.yaml                  # Backup‑only PVC
│   ├── service.yaml                     # ClusterIP Service (port 5432)
│   └── statefulset.yaml                 # PostgreSQL 18 StatefulSet
└── README.md                            # (you are here)
```

---  

## 3️⃣ Detailed Manifests  


### 3.1 `pvc.yaml` (primary PVC)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: postgresql-integration
spec:
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 50Gi
  storageClassName: sc-ceph-rbd
```

### 3.2 `pvc-backup.yaml` (backup PVC)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-backup
  namespace: postgresql-integration
spec:
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 50Gi          # Adjust according to your retention policy
  storageClassName: sc-ceph-rbd
```

### 3.3 `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: postgresql-integration
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
      protocol: TCP
```

### 3.4 `secret.yaml` (to add on manifests folder)

apiVersion: v1
kind: Secret
metadata:
  name: postgresql-secret
  namespace: postgresql-integration
  labels:
    app.kubernetes.io/name: postgresql
type: Opaque
stringData:
  POSTGRES_USER: "postgres"
  POSTGRES_PASSWORD: "my-secure-password"
  POSTGRES_DB: "postgres"


## 4️⃣ Deployment Steps  

```bash
# Create the namespace (if it does not already exist)
kubectl create namespace postgresql-integration

# Apply the manifests
kubectl apply -f manifests/secret.yaml
kubectl apply -f manifests/pvc.yaml
kubectl apply -f manifests/pvc-backup.yaml
kubectl apply -f manifests/configmap.yaml
kubectl apply -f manifests/service.yaml
kubectl apply -f manifests/statefulset.yaml
```

After a few minutes, verify everything is up:

```bash
kubectl -n postgresql-integration get pods -l app=postgres
kubectl -n postgresql-integration exec -it postgresql-0 -- psql
```

You should connect to postgres with posgres user and password set in secret.yaml 

# Create backup_user in postgresql

(postgres)# CREATE ROLE backup_user WITH LOGIN REPLICATION;
(postgres)# \du

And validate with a manual snapshot of the bdd


kubectl -n postgresql-integration exec -it postgresql-0 -- pg_basebackup --username=backup_user --pgdata=/var/backup/daily-backup --progress

---  

## 6️⃣ Quick Velero Example (summary)  


Example Velero command:

```bash
velero backup create pg-backup \
  --include-namespaces postgresql-integration \
  --ttl 720h0m0s   # 30 days retention
```

---  

## 7️⃣ Cleanup (if you need to remove everything)  

```bash
kubectl -n postgresql-integration delete statefulset postgres
kubectl -n postgresql-integration delete pvc postgres-data postgres-backup
kubectl -n postgresql-integration delete configmap pg-config
kubectl -n postgresql-integration delete service postgres
kubectl delete namespace postgresql-integration
```

---  

## 8️⃣ Additional References  

* **Ceph‑CSI driver** – <https://github.com/ceph/ceph-csi>  
* **Official PostgreSQL 18 docs** – <https://www.postgresql.org/docs/18/>  
* **Velero – CSI volume backup** – <https://velero.io/docs/main/csi/>  

---  

### 🎉 You now have a fully functional PostgreSQL 18 instance, backed by a Ceph PVC, with a dedicated backup PVC ready for Velero to consume. Proceed to the Velero configuration in the `../velero` folder. Happy hacking!  
