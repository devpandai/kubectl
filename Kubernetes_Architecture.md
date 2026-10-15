# Kubernetes Architecture

## 1. Purpose

Dokumen ini menjelaskan arsitektur referensi Kubernetes untuk menjalankan aplikasi production. Sesuaikan alamat IP, kapasitas, dan komponen dengan lingkungan aktual sebelum implementasi.

## 2. Reference topology

Contoh topologi non-HA untuk lab atau lingkungan kecil:

```text
                         Internet / Corporate Network
                                    |
                             Firewall / Router
                                    |
                         Load Balancer / VIP (opsional)
                                    |
                         Control Plane: 192.168.100.10
                                    |
                    +---------------+---------------+
                    |               |               |
              Worker 01        Worker 02        Worker 03
           192.168.100.11   192.168.100.12   192.168.100.13
                    |
             Application Pods
                    |
       Service -> Ingress -> External Clients
```

Untuk production dengan kebutuhan high availability (HA), gunakan minimal tiga control-plane node, endpoint API yang stabil di depan control plane, dan datastore etcd yang dirancang untuk quorum. Topologi tepat bergantung pada distribusi Kubernetes dan lingkungan.

## 3. Example node inventory

| Hostname | Example IP | Role |
|---|---|---|
| `k8s-cp-01` | `192.168.100.10` | Control plane |
| `k8s-worker-01` | `192.168.100.11` | Worker |
| `k8s-worker-02` | `192.168.100.12` | Worker |
| `k8s-worker-03` | `192.168.100.13` | Worker |

Alamat di atas hanya contoh. Pastikan IP tidak bertabrakan dan dapat dirutekan antar-node.

## 4. Core components

- **API server:** endpoint API Kubernetes.
- **etcd:** datastore cluster; lindungi akses dan backup-nya.
- **Scheduler:** memilih node untuk Pod.
- **Controller manager:** menjaga resource sesuai desired state.
- **kubelet:** mengelola Pod pada node.
- **Container runtime:** misalnya containerd.
- **CNI plugin:** menyediakan konektivitas jaringan Pod.
- **CoreDNS:** resolusi DNS internal.
- **Ingress controller / Gateway:** menerima traffic HTTP(S) dari luar cluster.
- **Storage provisioner:** menyediakan persistent storage bila diperlukan.
- **Metrics and logging:** observability dan alerting.

## 5. Network planning

Rencanakan CIDR yang tidak overlap dengan LAN, VPN, node, atau jaringan lain.

| Network | Example CIDR | Notes |
|---|---|---|
| Node network | `192.168.100.0/24` | Contoh saja |
| Pod network | `10.244.0.0/16` | Harus sesuai CNI |
| Service network | `10.96.0.0/12` | Contoh default yang umum |

Port yang perlu dipertimbangkan sesuai desain:
- Kubernetes API server TCP `6443`.
- Kubelet TCP `10250` dari komponen yang berwenang.
- etcd TCP `2379-2380` hanya antar anggota control plane yang relevan.
- Port CNI bergantung pada plugin dan mode yang dipilih.
- NodePort TCP/UDP `30000-32767` hanya jika digunakan dan dibutuhkan.

Jangan membuka port tersebut ke Internet secara langsung. Terapkan firewall dengan sumber dan tujuan yang spesifik.

## 6. Traffic flow

1. Client mengakses alamat aplikasi.
2. Load balancer atau ingress controller menerima request.
3. Ingress/Gateway merutekan request ke Service.
4. Service memilih Pod berdasarkan label selector.
5. Readiness probe menentukan apakah Pod layak menerima traffic.

## 7. Production design checklist

- [ ] Control plane HA dan endpoint API stabil bila diperlukan.
- [ ] Kapasitas CPU, RAM, disk, dan bandwidth diuji dengan beban aktual.
- [ ] Pod dan Service CIDR tidak overlap.
- [ ] NetworkPolicy diterapkan sesuai kebutuhan CNI.
- [ ] Storage persisten dan backup diuji.
- [ ] TLS, RBAC, audit, dan secret management dikonfigurasi.
- [ ] Monitoring, log, dan alert tersedia.
- [ ] Upgrade, restore, dan disaster recovery diuji.
- [ ] Image memakai versi/tag yang terkontrol; hindari `latest` untuk rilis production.

## 8. Decisions to record

Catat distribusi Kubernetes, versi, container runtime, CNI, ingress/Gateway, storage class, registry, DNS, sertifikat, backup target, RPO/RTO, dan pemilik operasional cluster.
