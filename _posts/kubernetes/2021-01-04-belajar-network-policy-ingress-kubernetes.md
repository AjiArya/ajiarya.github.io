---
layout: single
title: Belajar Network Policy Kubernetes - Ingress
category: kubernetes
tag:
- kubernetes
- container
- container orchestration
- minikube
- network policy
- cni
toc: true
toc_sticky: true
toc_icon: "book-reader"
---

Pada kesempatan kali ini saya akan membagikan bagaimana cara menggunakan object kubernetes yang bernama `NetworkPolicy` atau `netpol`. Terdapat 2 `policy` yang dapat diterapkan (`ingress` dan `egress`) namun untuk bahasan kali ini kita akan terfokus pada `ingress`

# Prasyarat
Untuk mengikuti lab ini:
* Minikube terpasang dengan CNI Calico atau CNI yang mendukung `netpol` lainnya.  (Jika belum ikuti [Tutorial ini](/kubernetes/setup-cni-di-minikube/))

Untuk implementasi:
* Kubernetes Cluster dengan CNI yang mendukung `netpol`

# Buat `namespace` untuk mempraktikkan `netpol`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: jakarta
  labels:
    name: jakarta
---
apiVersion: v1
kind: Namespace
metadata:
  name: bandung
  labels:
    name: bandung
```

Disini kita berikan `labels` agar kita bisa menggunakan `namespaceSelector`

# Contoh 1: Memperbolehkan `namespace` terpilih
Contoh berikut akan hanya memperbolehkan pod yang berada pada namespace `jakarta` yang dapat mengakses pod yang berada pada namespace `bandung`

Buat `pod` & `service` pada namespace `bandung`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bdgnginx
  namespace: bandung
  labels:
    app: bdgnginx
spec:
  containers:
  - name: bdgnginx
    image: nginx
    ports:
    - containerPort: 80
      name: http
---
apiVersion: v1
kind: Service
metadata:
  name: bdgnginxsvc
  namespace: bandung
spec:
  ports:
  - port: 80
    targetPort: http
  selector:
    app: bdgnginx
```

Buat `netpol` pada namespace `bandung`
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: contoh1
  namespace: bandung
spec:
  podSelector: {} # Memilih semua pod pada namespace bandung
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: jakarta
    ports:
    - port: 80
```

Uji coba akses pod `bdgnginx` dari namespace `default` & `jakarta`
```bash
# Dapatkan IP Address Service bdgnginxsvc 
kubectl -n bandung get svc

# Buat container untuk menjalankan perintah curl
kubectl -n default run curlfromdefault --image=radial/busyboxplus:curl -i --tty --rm
kubectl -n jakarta run curlfromjkt --image=radial/busyboxplus:curl -i --tty --rm

# Didalam container curlfromdefault
curl <Service IP/CLUSTER-IP>

# Didalam container curlfromjkt
curl <Service IP/CLUSTER-IP>
```

* akses pod `bdgnginx` hanya bisa dari pod `curlfromjkt` karena memiliki label `app: aman`

Hapus `netpol` untuk melanjutkan ke contoh 2
```bash
kubectl -n bandung delete netpol contoh1
```

# Contoh 2: Memperbolehkan `pod` terpilih
Contoh berikut akan memperbolehkan `pod` terpilih dari namespace yang sama

Buat `pod` baru pada namespace `bandung`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curlfrombdg1
  namespace: bandung
  labels:
    app: aman
spec:
  containers:
  - name: curlfrombdg1
    image: nginx
---
apiVersion: v1
kind: Pod
metadata:
  name: curlfrombdg2
  namespace: bandung
  labels:
    app: tidakaman
spec:
  containers:
  - name: curlfrombdg2
    image: nginx
```

Buat `netpol` baru pada namespace bandung
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: contoh2
  namespace: bandung
spec:
  podSelector:
    matchLabels:
      app: bdgnginx
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: aman
    ports:
    - port: 80
```

Uji coba akses pod `bdgnginx` dari namespace `jakarta`
```bash
kubectl -n bandung exec curlfrombdg1 -it -- curl --connect-timeout 5 curl <Service IP/CLUSTER-IP>
kubectl -n bandung exec curlfrombdg2 -it -- curl --connect-timeout 5 curl <Service IP/CLUSTER-IP>
```

* akses pod `bdgnginx` hanya bisa dari pod `curlfrombdg1` karena memiliki label `app: aman`

Hapus `netpol` untuk melanjutkan ke contoh 3
```bash
kubectl -n bandung delete netpol contoh2
```

# Contoh 3: Memperbolehkan `pod` dari `namespace` terpilih
Buat `pod` baru pada namespace `jakarta`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curlfromjkt1
  namespace: jakarta
  labels:
    app: aman
spec:
  containers:
  - name: curlfromjkt1
    image: nginx
---
apiVersion: v1
kind: Pod
metadata:
  name: curlfromjkt2
  namespace: jakarta
  labels:
    app: tidakaman
spec:
  containers:
  - name: curlfromjkt2
    image: nginx
```

Buat `netpol` baru pada namespace bandung
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: contoh3
  namespace: bandung
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: jakarta
      podSelector:
        matchLabels:
          app: aman
    ports:
    - port: 80
```

Uji coba akses pod `bdgnginx` dari namespace `jakarta`
```bash
kubectl -n jakarta exec curlfromjkt1 -it -- curl 10.107.50.56
kubectl -n jakarta exec curlfromjkt2 -it -- curl --connect-timeout 5 curl 10.107.50.56
```

* akses pod `bdgnginx` hanya bisa dari pod `curlfromjkt1` karena memiliki label `app: aman`

```bash
# Buat pod pada namespace default
kubectl -n default run curlfromdefault --image=radial/busyboxplus:curl -i --tty --rm

# curl dari dalam container
curl <Service IP/CLUSTER-IP>
```

Hapus `netpol` dan semua pod bisa mengakses `bdgnginx`
```bash
kubectl -n bandung delete netpol contoh3
```

# Contoh 4: Menerapkan `netpol` ke `pod` terpilih
`netpol` berikut akan hanya berlaku untuk pod dengan label `app: contoh`
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: contoh4
  namespace: bandung
spec:
  podSelector:
    matchLabels:
      app: contoh
  <RULES Policy>
```

# Referensi
[Kubernetes Docs - Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
