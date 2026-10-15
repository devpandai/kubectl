# Kubernetes Troubleshooting

## 1. Initial diagnostics

Mulai dari kondisi cluster dan event:

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
kubectl cluster-info
```

Untuk masalah spesifik, gunakan `kubectl describe` dan logs sebelum mengubah konfigurasi.

## 2. Node NotReady

Periksa:

```bash
kubectl describe node <NODE_NAME>
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 200 --no-pager
sudo systemctl status containerd
sudo journalctl -u containerd -n 200 --no-pager
df -h
free -h
```

Periksa konektivitas ke API server, status CNI, ruang disk, waktu sistem, sertifikat, dan konfigurasi runtime. Hindari restart semua node sekaligus.

## 3. Pod Pending

```bash
kubectl describe pod <POD_NAME> -n <NAMESPACE>
kubectl get events -n <NAMESPACE> --sort-by=.lastTimestamp
kubectl describe nodes
```

Penyebab umum: kapasitas CPU/RAM tidak cukup, taint/toleration tidak cocok, PVC belum tersedia, node selector/affinity tidak terpenuhi, atau kuota namespace habis.

## 4. CrashLoopBackOff

```bash
kubectl logs <POD_NAME> -n <NAMESPACE>
kubectl logs <POD_NAME> -n <NAMESPACE> --previous
kubectl describe pod <POD_NAME> -n <NAMESPACE>
```

Periksa konfigurasi aplikasi, environment variables, Secret/ConfigMap, dependency, permission, probe, dan exit code. Jangan menyalin secret ke tiket atau log publik.

## 5. ImagePullBackOff / ErrImagePull

Periksa nama image/tag, registry, DNS, konektivitas, rate limit, sertifikat registry, dan `imagePullSecrets`.

```bash
kubectl describe pod <POD_NAME> -n <NAMESPACE>
kubectl get secret -n <NAMESPACE>
```

Pastikan credential registry valid dan hanya tersedia untuk namespace/workload yang membutuhkan.

## 6. Service tidak dapat diakses

```bash
kubectl get svc,endpointslices -n <NAMESPACE>
kubectl get pods -n <NAMESPACE> --show-labels
kubectl describe svc <SERVICE_NAME> -n <NAMESPACE>
```

Periksa selector Service, label Pod, `targetPort`, readiness, policy jaringan, dan jalur routing. Lakukan pengujian dari Pod diagnostik yang disetujui dan hapus setelah selesai.

## 7. DNS bermasalah

```bash
kubectl get pods -n kube-system
kubectl logs -n kube-system -l k8s-app=kube-dns
```

Periksa CoreDNS, konfigurasi DNS Pod, `resolv.conf`, NetworkPolicy, dan konektivitas node. Label selector log bisa berbeda menurut distribusi.

## 8. PVC Pending

```bash
kubectl get pvc -A
kubectl describe pvc <PVC_NAME> -n <NAMESPACE>
kubectl get storageclass
kubectl get pv
```

Periksa apakah StorageClass default tersedia, provisioner berjalan, kapasitas/zone cocok, dan backend storage sehat.

## 9. Ingress / TLS

```bash
kubectl get ingress -A
kubectl describe ingress <INGRESS_NAME> -n <NAMESPACE>
kubectl get pods,svc -A
```

Periksa DNS, load balancer, ingress controller/Gateway, service backend, aturan host/path, secret sertifikat, masa berlaku sertifikat, dan firewall.

## 10. High CPU or memory

```bash
kubectl top nodes
kubectl top pods -A
kubectl describe pod <POD_NAME> -n <NAMESPACE>
```

`kubectl top` memerlukan Metrics Server yang berfungsi. Bandingkan penggunaan dengan requests/limits, periksa OOMKilled, autoscaling, dan tren monitoring.

## 11. Safe troubleshooting workflow

1. Catat waktu, dampak, perubahan terakhir, dan gejala.
2. Kumpulkan status, event, logs, serta metrik.
3. Tentukan lingkup: node, namespace, workload, network, atau storage.
4. Buat satu perubahan terkontrol pada satu waktu.
5. Verifikasi hasil dan dampak samping.
6. Rollback jika perubahan memperburuk kondisi.
7. Catat root cause dan tindakan pencegahan.

Hindari `kubeadm reset`, penghapusan PVC, penghapusan resource sistem, atau restart seluruh cluster sebagai langkah awal.
