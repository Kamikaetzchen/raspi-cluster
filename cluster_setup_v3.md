# Raspberry Pi Kubernetes Cluster Setup (v3)

This is a refined and fully annotated guide for setting up a Raspberry Pi 3B+ Kubernetes (k3s) cluster with NFS, MinIO (S3), Traefik, ArgoCD, and automated TLS certificate management using acme.sh.

---

## 🛠 Pre-Setup: System Checks

### Power & Thermal Checks

```bash
# Check throttling status (0x0 means no throttling)
vcgencmd get_throttled

# Measure CPU temperature
vcgencmd measure_temp

# Print CPU frequency in GHz
vcgencmd measure_clock arm | awk 'BEGIN { FS="=" } { printf("%.2fGHz\n", $2 / 1000000000) }'
```

These commands help ensure that the Pi is not underpowered or overheating, which could cause stability issues during cluster operation.

---

## ⚡ Bootstrap k3s

### On the Server Node (raspi01)

Basic installation:

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--write-kubeconfig-mode 644" sh -
```

Lightweight version (recommended):

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik --disable metrics-server --write-kubeconfig-mode 644" sh -
```

### Option C: Leanest Setup (Disable more defaults)

If you want to minimize resource usage further (recommended when using MetalLB and NFS/MinIO):

- `traefik`: disabled, because Traefik will be installed manually via Helm.
- `metrics-server`: disabled to save RAM.
- `servicelb`: disabled because MetalLB is used for LoadBalancer IPs.
- `local-storage`: disabled because NFS or MinIO is used for persistent storage.

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik --disable metrics-server --disable servicelb --disable local-storage --write-kubeconfig-mode 644" sh -
```

This disables Traefik and metrics-server, reducing resource usage on Pi hardware.

### On Agent Nodes (raspi02–raspi05)

Get the join token from raspi01:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Join the cluster:

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.178.101:6443 K3S_TOKEN=<token> sh -
```

---

## ⌨️ Shell Quality of Life

### Bash completion and alias

```bash
apt update
apt install bash-completion

# Enable kubectl autocompletion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

# Alias for kubectl
echo "alias k=kubectl" >> ~/.bashrc
echo "complete -o default -F __start_kubectl k" >> ~/.bashrc
source ~/.bashrc
```

---

## 🗄 NFS Storage (raspi02)

### Partition & Format

```bash
fdisk /dev/sda
# Example:
# n -> 1 -> Enter -> +400G -> Enter

mkfs.ext4 /dev/sda1 -L nfs
mkfs.ext4 /dev/sda2 -L s3
```

### Mount Points & fstab

```bash
mkdir -p /mnt/nfs /mnt/s3
blkid  # Get PARTUUIDs

nano /etc/fstab
PARTUUID=<uuid-nfs>  /mnt/nfs  ext4  defaults,noatime  0  2
PARTUUID=<uuid-s3>   /mnt/s3   ext4  defaults,noatime  0  2

mount -a
df -h | grep /mnt
```

### Install NFS Server

```bash
apt-get install -y nfs-kernel-server

nano /etc/exports
/mnt/nfs 192.168.178.0/24(rw,sync,no_subtree_check,no_root_squash)

exportfs -ra
systemctl restart nfs-kernel-server
```

### Mount on Other Nodes

```bash
mount -t nfs 192.168.178.102:/mnt/nfs /mnt

nano /etc/fstab
192.168.178.102:/mnt/nfs  /mnt  nfs  defaults,nofail,x-systemd.automount  0  0
```

---

## 🗃 MinIO (S3) on raspi02

### Install MinIO

```bash
wget https://dl.min.io/server/minio/release/linux-arm/minio -O /usr/local/bin/minio
chmod +x /usr/local/bin/minio

useradd -r minio-user -s /sbin/nologin
mkdir -p /etc/minio
chown minio-user:minio-user /mnt/s3
```

### Configure MinIO

```bash
cat <<EOF > /etc/minio/minio.env
MINIO_VOLUMES="/mnt/s3"
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=changeMe123
EOF

chmod 600 /etc/minio/minio.env
chown minio-user:minio-user /etc/minio/minio.env
```

### MinIO Systemd Service

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
ExecStart=/usr/local/bin/minio server ${MINIO_VOLUMES} --console-address ":9001"
Restart=always
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

Start MinIO:

```bash
systemctl daemon-reexec
systemctl daemon-reload
systemctl enable --now minio
```

---

## 🎛 Helm Installation

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

---

## 🚀 Optional: ArgoCD for GitOps

### Quick Full Install (YAML method)

If you just want a fast, full-featured ArgoCD setup without Helm customization:

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
kubectl apply -f https://raw.githubusercontent.com/Kamikaetzchen/raspi-cluster/refs/heads/master/k8s/argocd/app-of-apps.yaml
```

For production or customized deployments, use the Helm method below.

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

## 🌐 Traefik (raspi03)

```bash
kubectl label node raspi03 ingress=true
helm repo add traefik https://traefik.github.io/charts
helm repo update

helm upgrade --install traefik traefik/traefik \
  --namespace traefik --create-namespace \
  -f https://raw.githubusercontent.com/Kamikaetzchen/raspi-cluster/master/traefik-values.yaml
```

Verify:

```bash
kubectl get pods -n traefik
```

### Uninstall Traefik
If you need to remove Traefik completely:

```bash
helm uninstall traefik -n traefik
kubectl delete svc -n traefik traefik
kubectl delete deploy,rs,pod -n traefik --all
kubectl delete namespace traefik
```

---

## 🏷 TLS Certificates with acme.sh

### Install acme.sh

```bash
curl https://get.acme.sh | sh
source ~/.bashrc
acme.sh --version
acme.sh --upgrade --auto-upgrade
acme.sh --set-default-ca --server letsencrypt
```

### Configure do.de API

Get API Key from the do.de website.

```bash
export DO_LETOKEN="<api-key>"
# Add to ~/.bashrc for persistence
```

### Register Account

```bash
acme.sh --register-account -m you@example.com --server letsencrypt
```

### Issue Wildcard Cert

```bash
acme.sh --issue --dns dns_doapi -d kamikatze.eu -d '*.kamikatze.eu'
```

### Install Cert for Traefik

```bash
acme.sh --install-cert -d kamikatze.eu -d '*.kamikatze.eu' \
  --key-file /root/.acme.sh/kamikatze.eu_ecc/kamikatze.eu.key \
  --fullchain-file /root/.acme.sh/kamikatze.eu_ecc/fullchain.cer \
  --reloadcmd "kubectl create secret tls traefik-cert \
    --cert=/root/.acme.sh/kamikatze.eu_ecc/fullchain.cer \
    --key=/root/.acme.sh/kamikatze.eu_ecc/kamikatze.eu.key \
    -n traefik --dry-run=client -o yaml | kubectl apply -f - && \
    kubectl rollout restart deployment traefik -n traefik"
```

### Verify Cron

```bash
crontab -l
# 30 22 * * * "/root/.acme.sh"/acme.sh --cron --home "/root/.acme.sh" > /dev/null
```

### Test Renewal

```bash
acme.sh --renew -d kamikatze.eu --force
```

Check cert in Kubernetes:

```bash
kubectl get secret traefik-cert -n traefik -o jsonpath="{.data.tls\.crt}" | base64 -d | openssl x509 -noout -dates
```

---

## 🛜 MetalLB Installation

```bash
kubectl create namespace metallb-system
helm repo add metallb https://metallb.github.io/metallb
helm repo update

helm upgrade --install metallb metallb/metallb -n metallb-system
kubectl apply -f https://raw.githubusercontent.com/Kamikaetzchen/raspi-cluster/master/metallb-config.yaml
```

Verify:

```bash
kubectl get pods -n metallb-system
```

If a LoadBalancer IP doesn't respond to ping:

- Use `curl` or `nc` (MetalLB announces only TCP/UDP).
- Check ARP table on your LAN: `arp -an | grep <IP>`
- Validate MetalLB logs:
  ```bash
  kubectl logs -n metallb-system -l app.kubernetes.io/component=speaker
  ```

---

## 🔧 Troubleshooting

### 1. DNS Resolution Issues

- Verify CoreDNS is running:
  ```bash
  kubectl get pods -n kube-system -l k8s-app=kube-dns
  ```
- Check pod DNS:
  ```bash
  kubectl exec -it <pod> -- nslookup kubernetes.default
  ```

### 2. MetalLB IP Not Responding

- Check assigned IP:
  ```bash
  kubectl describe svc traefik -n traefik | grep IPAllocated
  ```
- Verify ARP:
  ```bash
  arp -an | grep <LoadBalancer-IP>
  ```
- Test service port:
  ```bash
  nc -zv <LoadBalancer-IP> 80
  ```

### 3. Traefik Not Routing

- Inspect ingress routes:
  ```bash
  kubectl get ingress -A
  ```
- Check Traefik logs:
  ```bash
  kubectl logs -n traefik -l app.kubernetes.io/name=traefik
  ```

### 4. Certificate Not Updating

- Check acme.sh logs:
  ```bash
  tail -f ~/.acme.sh/acme.sh.log
  ```
- Verify cert in Kubernetes:
  ```bash
  kubectl get secret traefik-cert -n traefik -o jsonpath="{.data.tls\\.crt}" | base64 -d | openssl x509 -noout -dates
  ```

---

✅ Cluster is now fully automated with NFS, MinIO, ArgoCD, Traefik, MetalLB, and automatic Let's Encrypt wildcard TLS.



