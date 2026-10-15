# Kubernetes Installation Guide

## 1. Scope and assumptions

Panduan ini merupakan baseline instalasi menggunakan Ubuntu Server, `containerd`, `kubeadm`, `kubelet`, dan `kubectl`. Contoh perintah harus diverifikasi terhadap versi Kubernetes dan CNI yang dipilih sebelum digunakan pada production.

> **Penting:** Instalasi ini membentuk control plane tunggal, bukan HA. Untuk HA, ikuti desain multi-control-plane resmi Kubernetes dan siapkan load balancer/API endpoint serta etcd yang sesuai.

## 2. Example inventory

| Hostname | IP | Role |
|---|---|---|
| `k8s-cp-01` | `192.168.100.10` | Control plane |
| `k8s-worker-01` | `192.168.100.11` | Worker |
| `k8s-worker-02` | `192.168.100.12` | Worker |

Ganti IP sesuai jaringan aktual. Pastikan hostname unik, DNS atau `/etc/hosts` konsisten, waktu tersinkronisasi, dan komunikasi antar-node tersedia.

## 3. Prepare all nodes

Jalankan pada setiap node:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl ca-certificates gpg apt-transport-https
sudo hostnamectl set-hostname <hostname-node>
```

Tambahkan pemetaan host jika DNS internal belum tersedia:

```text
192.168.100.10 k8s-cp-01
192.168.100.11 k8s-worker-01
192.168.100.12 k8s-worker-02
```

Simpan entri tersebut di `/etc/hosts`.

### Disable swap (baseline kubeadm)

```bash
sudo swapoff -a
```

Nonaktifkan konfigurasi swap pada `/etc/fstab` jika memang tidak dibutuhkan sistem. Periksa konfigurasi swapfile atau swap partition sebelum mengubahnya.

### Kernel modules and sysctl

```bash
cat <<'EOF' | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<'EOF' | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

## 4. Install containerd

Paket `containerd` dari repository OS dapat digunakan jika versinya sesuai kebutuhan. Untuk production, tentukan sumber paket dan versi yang disetujui.

```bash
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
```

Edit `/etc/containerd/config.toml` dan atur `SystemdCgroup = true` pada bagian konfigurasi runtime runc yang sesuai dengan versi containerd. Pastikan hanya nilai yang tepat yang diubah.

```bash
sudo systemctl enable --now containerd
sudo systemctl restart containerd
sudo systemctl --no-pager --full status containerd
```

## 5. Install kubeadm, kubelet, kubectl

Pilih versi minor Kubernetes yang didukung dan konsisten. Contoh di bawah menggunakan placeholder `<MINOR>`; ganti dengan minor version yang benar, misalnya `v1.xx`, setelah memeriksa dokumentasi resmi dan ketersediaan paket.

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gpg
sudo mkdir -p -m 755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/<MINOR>/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/<MINOR>/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

`kubelet` dapat restart berulang sebelum cluster diinisialisasi; hal tersebut dapat terjadi pada tahap ini.

## 6. Initialize the control plane

Pilih Pod CIDR yang cocok dengan CNI dan tidak overlap dengan jaringan lain. Contoh:

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.100.10 \
  --pod-network-cidr=10.244.0.0/16
```

Simpan output `kubeadm join` secara aman. Jangan membagikan token atau konfigurasi admin ke pihak yang tidak berwenang.

Konfigurasikan `kubectl` untuk user administrator yang sesuai:

```bash
mkdir -p "$HOME/.kube"
sudo cp /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
kubectl get nodes
```

File `admin.conf` memiliki hak akses tinggi. Batasi akses dan jangan commit ke Git.

## 7. Install CNI

Pilih satu CNI yang kompatibel dengan versi Kubernetes dan kebutuhan routing/NetworkPolicy. Ikuti dokumentasi resmi proyek untuk versi yang dipilih. Contoh berikut hanya ilustrasi pola instalasi; manifest dan CIDR harus sesuai konfigurasi aktual:

```bash
kubectl apply -f <CNI_MANIFEST_URL_YANG_TERVERIFIKASI>
```

Tunggu komponen jaringan siap:

```bash
kubectl get pods -n kube-system
kubectl get nodes
```

Jangan memasang beberapa CNI sekaligus tanpa prosedur migrasi yang didukung.

## 8. Join worker nodes

Jalankan perintah `kubeadm join` yang dihasilkan saat `kubeadm init` pada setiap worker, menggunakan token dan hash yang valid. Jangan gunakan placeholder secara literal.

Dari control plane:

```bash
kubectl get nodes -o wide
```

Pastikan seluruh node berstatus `Ready`.

Jika token kedaluwarsa, buat token baru dengan `kubeadm token create --print-join-command` pada control plane dan tangani output sebagai informasi sensitif.

## 9. Create namespace and deploy a test workload

```bash
kubectl create namespace production
```

Simpan sebagai `nginx-test.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-test
  template:
    metadata:
      labels:
        app: nginx-test
    spec:
      containers:
        - name: nginx
          image: nginx:1.28
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 5
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-test
  namespace: production
spec:
  selector:
    app: nginx-test
  ports:
    - name: http
      port: 80
      targetPort: 80
  type: ClusterIP
```

Apply dan verifikasi:

```bash
kubectl apply -f nginx-test.yaml
kubectl rollout status deployment/nginx-test -n production
kubectl get pods,svc -n production
```

## 10. Post-install validation

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get events -A --sort-by=.lastTimestamp
kubectl cluster-info
```

Tambahkan ingress/Gateway, storage class, metrics, logging, TLS, policies, backup, dan alerting sesuai desain. Jangan menganggap cluster siap production hanya karena node berstatus `Ready`.

## 11. Rollback / reset warning

`kubeadm reset` dapat menghapus konfigurasi node dan mengganggu workload. Jangan menjalankannya sebagai langkah troubleshooting rutin. Rencanakan prosedur decommission/reset dan backup terlebih dahulu.
