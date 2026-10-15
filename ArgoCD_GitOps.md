# Argo CD GitOps

## 1. Purpose

Argo CD menerapkan pendekatan GitOps: konfigurasi Kubernetes disimpan di Git, dibandingkan dengan kondisi cluster, lalu disinkronkan sesuai kebijakan. Repository harus direview dan dilindungi seperti kode aplikasi.

## 2. Suggested repository layout

```text
kubernetes-manifests/
├── apps/
│   └── nginx-test/
│       ├── deployment.yaml
│       └── service.yaml
├── environments/
│   ├── staging/
│   └── production/
└── argocd/
    └── applications/
```

Pisahkan konfigurasi per environment dan gunakan Kustomize atau Helm bila dibutuhkan. Jangan menyimpan secret plaintext di Git.

## 3. Install Argo CD

Gunakan instruksi instalasi resmi Argo CD untuk versi yang telah dipilih dan diuji. Contoh pola instalasi pada namespace khusus:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f <ARGOCD_INSTALL_MANIFEST_URL_YANG_TERVERIFIKASI>
kubectl get pods -n argocd
```

URL manifest dan versi harus ditetapkan setelah memeriksa dokumentasi resmi. Untuk production, jangan mengikuti branch tidak terikat versi tanpa proses perubahan.

## 4. Example Application

Contoh berikut mengasumsikan repository Git berisi manifest di direktori `apps/nginx-test`. Ganti URL repository dan branch/tag sesuai organisasi.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-test-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example-org/kubernetes-manifests.git
    targetRevision: main
    path: apps/nginx-test
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

Simpan sebagai `application.yaml`, lalu apply dari akun yang memiliki izin:

```bash
kubectl apply -f application.yaml
kubectl get applications -n argocd
```

Untuk production, pertimbangkan repository branch protection, review wajib, pinned revisions, dan kebijakan sinkronisasi yang disetujui.

## 5. Automated sync

Automated sync, `prune`, dan `selfHeal` mengubah cluster berdasarkan Git. Aktifkan hanya setelah proses review, rollback, dan ownership jelas.

Contoh opsi (tambahkan ke `spec.syncPolicy` setelah dievaluasi):

```yaml
automated:
  prune: true
  selfHeal: true
```

- `prune: true` dapat menghapus resource yang tidak lagi ada di Git.
- `selfHeal: true` dapat mengembalikan perubahan manual agar sesuai dengan Git.

Gunakan project dan destination restrictions agar aplikasi tidak bisa melakukan deploy di luar cakupan yang diizinkan.

## 6. Secrets

Jangan commit Kubernetes Secret berisi kredensial plaintext. Gunakan solusi secret management yang disetujui, seperti external secret manager atau enkripsi yang terintegrasi dengan alur GitOps. Batasi akses repository, Argo CD, dan service account.

## 7. Operations

```bash
kubectl get applications -n argocd
kubectl describe application <APP_NAME> -n argocd
kubectl get pods -n argocd
```

Periksa status sync/health, event, log controller/repo-server, kredensial Git, path manifest, dan izin target namespace.

## 8. Promotion and rollback

1. Ubah manifest melalui pull request.
2. Jalankan validasi YAML, policy, image scan, dan pengujian staging.
3. Review perubahan dan dampaknya.
4. Merge/promote ke revision production yang disetujui.
5. Verifikasi status Argo CD dan kesehatan aplikasi.
6. Rollback melalui revert commit atau revision sebelumnya yang diketahui baik.
7. Simpan catatan perubahan dan hasil validasi.

Jangan mengandalkan rollback aplikasi untuk membatalkan perubahan database yang tidak kompatibel; perubahan schema memerlukan rencana terpisah.
