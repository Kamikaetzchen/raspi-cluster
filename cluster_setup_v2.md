# Raspberry Pi Kubernetes Cluster Setup

This guide sets up a lightweight K3s-based Kubernetes cluster on Raspberry Pi 3B+ devices, including NFS and S3-compatible storage (MinIO), with optional ArgoCD deployment for GitOps.

---

## 🐧 K3s Installation

### Server Node (raspi01):

#### Option A: Basic K3s
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--write-kubeconfig-mode 644" sh -
```

#### Option B: Minimal (No Traefik or Metrics Server)
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik --disable metrics-server --write-kubeconfig-mode 644" sh -
```

#### Option C: Leanest Setup (Disable more defaults)
Disables additional unused features to reduce RAM and CPU usage on the control plane node:

- `traefik`: external ingress is handled manually
- `metrics-server`: no resource metrics collection
- `servicelb`: not using K3s' built-in load balancer
- `local-storage`: using NFS or other storage instead

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="\
  --disable traefik \
  --disable metrics-server \
  --disable servicelb \
  --disable local-storage \
  --write-kubeconfig-mode 644" sh -
```

> You may also disable `coredns`, `cloud-controller`, or `network-policy` if your use case allows it. See the K3s docs for full flag reference.

### Agent Nodes (raspi02–05):

```bash
# Get token from raspi01:
sudo cat /var/lib/rancher/k3s/server/node-token

# On each agent node:
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.178.101:6443 K3S_TOKEN=<your-node-token> sh -
```

---

## 📦 NFS & S3-Compatible Storage (on raspi02)

### Partition and Format USB Drive
```bash
fdisk /dev/sda
# Follow prompts: n -> 1 -> Enter -> +400G -> Enter

mkfs.ext4 /dev/sda1 -L nfs
mkfs.ext4 /dev/sda2 -L s3
```

### Mount Points
```bash
mkdir -p /mnt/nfs /mnt/s3
```

### fstab Setup
```bash
blkid  # Get PARTUUIDs

nano /etc/fstab
PARTUUID=f7fad32b-01  /mnt/nfs  ext4  defaults,noatime  0  2
PARTUUID=f7fad32b-02  /mnt/s3   ext4  defaults,noatime  0  2

mount -a
```

### Install and Configure NFS Server
```bash
apt-get install -y nfs-kernel-server

nano /etc/exports
/mnt/nfs 192.168.178.0/24(rw,sync,no_subtree_check,no_root_squash)

exportfs -ra
systemctl restart nfs-kernel-server
```

### Mount NFS on Other Nodes
```bash
mount -t nfs 192.168.178.102:/mnt/nfs /mnt

nano /etc/fstab
192.168.178.102:/mnt/nfs  /mnt  nfs  defaults,nofail,x-systemd.automount  0  0
```

---

## 🗃️ MinIO S3-Compatible Storage (still raspi02)

### Install MinIO
```bash
wget https://dl.min.io/server/minio/release/linux-arm/minio -O /usr/local/bin/minio
chmod +x /usr/local/bin/minio

useradd -r minio-user -s /sbin/nologin
mkdir -p /etc/minio
chown minio-user:minio-user /mnt/s3
```

### MinIO Environment Configuration
```bash
cat <<EOF > /etc/minio/minio.env
MINIO_VOLUMES="/mnt/s3"
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=changeMe123
EOF

chmod 600 /etc/minio/minio.env
chown minio-user:minio-user /etc/minio/minio.env
```

### Systemd Service
```bash
nano /etc/systemd/system/minio.service

[Unit]
Description=MinIO S3-compatible object storage
After=network-online.target
Wants=network-online.target

[Service]
User=minio-user
Group=minio-user
EnvironmentFile=/etc/minio/minio.env
ExecStart=/usr/local/bin/minio server ${MINIO_VOLUMES} --console-address "0.0.0.0:9001"
Restart=always
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### Start MinIO
```bash
systemctl daemon-reexec
systemctl daemon-reload
systemctl enable --now minio
```

---

## 📦 Install Helm

Install Helm on any node with `kubectl` access (e.g. raspi01):
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify:
```bash
helm version
```

---

## 🚀 Optional: ArgoCD for GitOps

### Label raspi04 to pin ArgoCD
```bash
kubectl label node raspi04 argocd=true
```

### Create a slimmed-down ArgoCD config
Save this as `argocd-values.yaml` or fetch it from your Git repo.

```yaml
dex:
  enabled: false

notifications:
  enabled: false

applicationSet:
  enabled: true
  nodeSelector:
    argocd: "true"

redis:
  enabled: true
  nodeSelector:
    argocd: "true"

controller:
  replicas: 1
  nodeSelector:
    argocd: "true"

server:
  replicas: 1
  nodeSelector:
    argocd: "true"

repoServer:
  nodeSelector:
    argocd: "true"
```

### Add ArgoCD Helm repo and install
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  --namespace argocd --create-namespace \
  -f argocd-values.yaml
```

### Get Admin Password
```bash
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

### Port-Forward UI from a PC
```powershell
ssh -L 8080:localhost:8080 kamikatze@raspi01
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then visit [https://localhost:8080](https://localhost:8080) in your browser.

---

## ✅ Next Steps

- Deploy Traefik ingress controller on raspi03
- Set up DNS records to point to raspi03
- Add cert-manager or Caddy/Let's Encrypt if HTTPS is desired
- Deploy apps via Ingress or ArgoCD

