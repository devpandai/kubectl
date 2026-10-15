# Kubernetes Security Hardening

## 1. Objective

Checklist baseline untuk mengurangi risiko akses tidak sah, kebocoran secret, dan dampak kompromi workload. Sesuaikan dengan threat model, regulasi, dan distribusi Kubernetes yang digunakan.

## 2. Access control and identity

- Gunakan akun individual dan autentikasi terpusat jika tersedia.
- Terapkan RBAC dengan prinsip least privilege.
- Hindari penggunaan `cluster-admin` untuk aktivitas harian.
- Review RoleBinding dan ClusterRoleBinding secara berkala.
- Lindungi kubeconfig; file admin cluster tidak boleh masuk ke repository, ticket, atau chat publik.
- Batasi akses API server melalui firewall/VPN/private network.
- Aktifkan audit logging sesuai kebijakan dan simpan log secara terproteksi.

Audit binding berhak tinggi:

```bash
kubectl get clusterrolebindings
kubectl get rolebindings -A
```

## 3. Node and control plane

- Patch OS, Kubernetes, container runtime, dan CNI mengikuti jadwal yang diuji.
- Gunakan SSH key atau akses terkelola; nonaktifkan login password/root langsung jika sesuai kebijakan.
- Batasi SSH dan port control plane hanya dari sumber yang diperlukan.
- Lindungi etcd karena menyimpan data cluster yang sensitif.
- Aktifkan enkripsi data Kubernetes Secrets at rest sesuai kemampuan platform.
- Pastikan waktu node tersinkronisasi dan sertifikat dipantau masa berlakunya.
- Pisahkan jaringan manajemen dari jaringan aplikasi bila memungkinkan.

## 4. Workload security

- Gunakan image dari registry tepercaya dan pin tag versi atau digest.
- Scan image untuk vulnerability sebelum deployment.
- Jalankan container non-root jika aplikasi mendukung.
- Hindari `privileged: true`, host PID/network, dan `hostPath` kecuali benar-benar diperlukan.
- Set `resources.requests` dan `resources.limits`.
- Tambahkan readiness, liveness, dan startup probe yang sesuai.
- Terapkan Pod Security Admission, misalnya label `restricted`, setelah workload diuji.

Contoh label namespace (uji kompatibilitas workload terlebih dahulu):

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

## 5. Network security

- Terapkan NetworkPolicy untuk membatasi komunikasi antar-Pod.
- Pastikan CNI yang digunakan mendukung dan benar-benar menegakkan NetworkPolicy.
- Terapkan default-deny secara bertahap dan tambahkan rule DNS, ingress, egress, serta dependensi aplikasi yang diperlukan.
- Jangan mengekspos API server, etcd, kubelet, atau dashboard admin ke Internet.
- Gunakan TLS untuk traffic eksternal dan internal bila diperlukan oleh threat model.

## 6. Secrets

Jangan menyimpan password, token, private key, atau kubeconfig di Git. Kubernetes Secret secara default bukan otomatis aman hanya karena datanya ditampilkan sebagai base64.

Pilihan pengelolaan:
- External secret manager atau secret store yang disetujui.
- Encryption at rest untuk data API server/etcd.
- RBAC ketat dan pembatasan akses Secret.
- Rotasi kredensial dan audit akses.
- Jangan menaruh secret di argumen command yang tersimpan pada shell history.

## 7. Resource governance

- Gunakan `ResourceQuota` dan `LimitRange` per namespace.
- Terapkan PodDisruptionBudget untuk workload yang perlu tetap tersedia saat maintenance.
- Atur node taint/toleration dan affinity hanya sesuai kebutuhan.
- Pisahkan workload sensitif jika membutuhkan batas isolasi lebih kuat.

## 8. Verification checklist

- [ ] Akses API server dibatasi.
- [ ] RBAC ditinjau dan hak istimewa minimum diterapkan.
- [ ] Pod Security Admission atau kontrol setara aktif.
- [ ] NetworkPolicy diuji.
- [ ] Image scan dan patching berjalan.
- [ ] Secret tidak berada di repository.
- [ ] Audit log dan alert tersedia.
- [ ] Backup etcd dan aplikasi dienkripsi serta aksesnya dibatasi.
- [ ] Prosedur incident response dan rotasi kredensial diuji.

## 9. Incident response

Jika kredensial atau node diduga terkompromi: batasi akses terdampak, simpan bukti/log, cabut atau rotasi kredensial, evaluasi workload dan image, periksa audit trail, dan pulihkan dari sumber tepercaya. Jangan menghapus bukti sebelum kebutuhan investigasi dipertimbangkan.
