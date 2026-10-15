# Kubernetes Backup and Restore

## 1. Scope

Backup cluster harus mencakup lebih dari output `kubectl get all`. Rencana backup perlu mempertimbangkan state etcd, manifest/configuration, persistent volume, database eksternal, secret, sertifikat, serta konfigurasi infrastruktur.

Tetapkan target **RPO** (maksimum kehilangan data yang diterima) dan **RTO** (target waktu pemulihan) dengan pemilik aplikasi.

## 2. Backup inventory

| Component | Backup method | Notes |
|---|---|---|
| etcd | Snapshot resmi etcd/kubeadm | Sangat penting untuk state control plane |
| Manifests | Git/version control | Jangan commit secret plaintext |
| Persistent volumes | Snapshot/backup dari storage platform | Ikuti prosedur storage |
| External databases | Native backup/database tooling | Kubernetes backup tidak menggantikan backup DB |
| Certificates and config | Secure backup | Enkripsi dan batasi akses |
| Registry images | Registry replication/backup | Pastikan image rilis masih tersedia |

## 3. Export manifests for reference

```bash
mkdir -p "$HOME/k8s-export"
kubectl get namespaces -o yaml > "$HOME/k8s-export/namespaces.yaml"
kubectl get deployments,statefulsets,daemonsets,services,ingresses,configmaps,pvc -A -o yaml \
  > "$HOME/k8s-export/workloads.yaml"
```

Ini adalah ekspor referensi, bukan backup lengkap yang dapat menjamin pemulihan cluster. Output dapat mengandung informasi sensitif; simpan secara aman. Gunakan GitOps/manifests yang bersih sebagai sumber deklaratif utama.

## 4. etcd snapshot

Gunakan prosedur resmi sesuai cara etcd dipasang dan versi yang berjalan. Pada kubeadm dengan etcd lokal, akses sertifikat client biasanya berada di `/etc/kubernetes/pki/etcd/`. Pastikan path dan hak akses diverifikasi sebelum menjalankan.

Contoh pola perintah untuk etcd lokal; sesuaikan versi `etcdctl`, endpoint, dan path sertifikat:

```bash
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  snapshot save /secure-backup/etcd-$(date +%F-%H%M%S).db
```

Pastikan direktori backup sudah dibuat, memiliki izin yang benar, dan berada di storage yang sesuai. Jangan menganggap perintah berhasil sebelum memeriksa exit status dan hasil snapshot:

```bash
sudo ETCDCTL_API=3 etcdctl snapshot status /secure-backup/<nama-snapshot>.db --write-out=table
```

Lakukan transfer terenkripsi ke lokasi terpisah, atur retensi, dan batasi akses.

## 5. Persistent volume and databases

- Gunakan snapshot atau backup yang didukung storage driver.
- Untuk database, gunakan backup native yang konsisten; pertimbangkan point-in-time recovery bila diperlukan.
- Backup volume yang sedang aktif harus mengikuti panduan aplikasi/storage untuk menjaga konsistensi.
- Uji pemulihan pada environment terisolasi.

## 6. Restore planning

Restore etcd dapat mengganti state control plane dan berisiko membuat resource tidak konsisten dengan workload/storage. Ikuti prosedur restore resmi untuk versi Kubernetes/etcd yang tepat.

Sebelum restore:
1. Deklarasikan incident dan tentukan snapshot yang akan digunakan.
2. Catat versi Kubernetes, etcd, CNI, dan konfigurasi cluster.
3. Isolasi control plane sesuai runbook.
4. Pastikan snapshot dapat dibaca dan checksum/validitasnya diverifikasi.
5. Ikuti prosedur resmi restore untuk topology yang digunakan.
6. Pulihkan konfigurasi dan data persisten sesuai urutan dependensi.
7. Validasi node, Pod, Service, PVC, aplikasi, DNS, dan integritas data.
8. Dokumentasikan hasil serta penyebab kegagalan.

Jangan menjalankan perintah restore contoh dari versi berbeda tanpa memeriksa dokumentasi resmi yang cocok.

## 7. Backup schedule example

| Data | Example cadence | Retention |
|---|---|---|
| etcd snapshot | Harian dan sebelum perubahan besar | Berdasarkan RPO/kebijakan |
| Application DB | Sesuai transaksi dan RPO | Berdasarkan kebijakan |
| Persistent volumes | Harian atau sesuai perubahan data | Berdasarkan kebijakan |
| Git/manifests | Setiap perubahan | Riwayat repository |
| Restore test | Berkala, misalnya per kuartal | Simpan hasil uji |

Jadwal ini contoh awal, bukan rekomendasi universal. Sesuaikan dengan kritikalitas data dan biaya.

## 8. Restore test checklist

- [ ] Snapshot dapat diverifikasi.
- [ ] Kredensial dan sertifikat yang diperlukan tersedia secara aman.
- [ ] Backup berada di lokasi terpisah dari cluster.
- [ ] Restore diuji pada environment terisolasi.
- [ ] Aplikasi dan database lolos pemeriksaan integritas.
- [ ] RPO/RTO aktual dicatat.
- [ ] Runbook diperbarui berdasarkan hasil uji.
