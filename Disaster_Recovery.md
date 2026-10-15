# Kubernetes Disaster Recovery

## 1. Objective

Dokumen ini menjadi kerangka pemulihan saat cluster, control plane, storage, jaringan, atau layanan pendukung mengalami kegagalan besar.

Setiap layanan harus memiliki RPO dan RTO yang disepakati. Angka target harus ditentukan berdasarkan kebutuhan bisnis dan diuji, bukan diasumsikan dari keberadaan backup.

## 2. Disaster scenarios

| Scenario | Example response |
|---|---|
| Satu worker gagal | Cordon/drain sesuai kondisi, perbaiki atau ganti node |
| Control plane gagal | Pulihkan control plane atau etcd sesuai topology |
| etcd corrupt/lost | Gunakan prosedur restore etcd yang cocok dengan versi/topology |
| Storage gagal | Pulihkan volume dari backup/snapshot dan validasi data |
| Region/site hilang | Aktifkan site DR dan pulihkan data serta layanan |
| Credential compromise | Isolasi akses, rotasi kredensial, investigasi, lalu rebuild bila perlu |

## 3. Preparation

- Dokumentasikan topology, IP, DNS, firewall, versi, CNI, ingress, storage, dan dependency.
- Simpan manifest dan konfigurasi infrastruktur di repository yang dilindungi.
- Backup etcd, database, dan persistent data ke lokasi terpisah.
- Enkripsi backup dan batasi aksesnya.
- Pastikan image aplikasi tersedia di registry yang dapat diakses saat pemulihan.
- Simpan prosedur akses darurat secara aman.
- Tetapkan incident commander, pemilik aplikasi, jalur komunikasi, RPO, dan RTO.
- Uji restore dan failover berkala.

## 4. Recovery sequence

Urutan aktual bergantung pada kegagalan. Sebagai baseline:

1. Deklarasikan incident dan pastikan keselamatan serta akses administrator.
2. Identifikasi lingkup, titik kegagalan, dan kemungkinan kompromi keamanan.
3. Lindungi bukti dan hindari tindakan destruktif sebelum snapshot/log yang diperlukan diamankan.
4. Pulihkan jaringan, DNS, load balancer, dan akses manajemen.
5. Pulihkan control plane/etcd mengikuti runbook resmi untuk versi dan topology.
6. Validasi API server, node, CNI, DNS, dan storage.
7. Pulihkan database dan persistent volumes sesuai dependensi aplikasi.
8. Terapkan workload dari GitOps atau manifest tepercaya.
9. Verifikasi TLS, ingress, autentikasi, integrasi eksternal, dan background jobs.
10. Jalankan uji fungsional serta pemeriksaan integritas data.
11. Alihkan traffic secara bertahap dan pantau error/latency.
12. Catat waktu pemulihan, kehilangan data, dan tindakan lanjutan.

## 5. Control plane / etcd recovery

Restore etcd adalah operasi berisiko tinggi. Gunakan prosedur resmi yang cocok dengan versi Kubernetes/etcd, jenis topology, dan metode deployment. Jangan menyalin snapshot ke cluster aktif atau menjalankan perintah restore generik tanpa runbook yang telah diuji.

Checklist:
- [ ] Snapshot dipilih berdasarkan waktu dan validitas.
- [ ] Versi dan konfigurasi etcd diketahui.
- [ ] Anggota cluster dan endpoint dipetakan.
- [ ] Snapshot dapat diverifikasi.
- [ ] Prosedur isolasi cluster dan restore jelas.
- [ ] Sertifikat/config yang diperlukan tersedia dengan aman.
- [ ] Setelah restore, API server dan komponen control plane tervalidasi.

## 6. Application and data recovery

- Pulihkan database dari backup konsisten dan replay log bila tersedia.
- Pulihkan persistent volume sesuai urutan dependensi.
- Periksa ownership, permissions, encryption keys, dan konfigurasi storage.
- Validasi data melalui pemeriksaan aplikasi/database, bukan hanya status Pod.
- Hindari menjalankan dua instance aktif dari sistem stateful yang tidak mendukung active-active.

## 7. DR test plan

Minimal, jadwalkan latihan yang sesuai kritikalitas layanan:

- Restore database dan volume di environment terisolasi.
- Rebuild worker dari image/config standar.
- Simulasi kehilangan control plane pada environment non-production.
- Deploy aplikasi dari repository GitOps ke cluster pemulihan.
- Verifikasi DNS, sertifikat, registry, secret manager, dan integrasi eksternal.
- Ukur waktu pemulihan aktual dan bandingkan dengan RTO/RPO.

## 8. Post-incident review

Catat timeline, root cause, dampak pengguna, kehilangan data, RPO/RTO aktual, efektivitas alert, dan tindakan pencegahan. Perbarui runbook serta lakukan uji ulang untuk memastikan perbaikan efektif.
