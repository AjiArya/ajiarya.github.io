---
layout: single
title: Setup Minikube di KVM - Linux
category: kubernetes
toc: true
toc_sticky: true
toc_icon: "book-reader"
---

Halo semua, pada kesempatan kali ini kita akan mencoba *setup* minikube di atas KVM. Tutorial kali ini tidak akan dimulai dari memasang KVM terlebih dahulu, melainkan langsung saja memasang minikube dan menjalankan kubernetes *cluster* secara lokal (**standalone**). 

# Apa itu minikube?

minikube adalah cara cepat untuk melakukan *setup* kubernetes *cluster* secara local di macOS, Linux, dan Windows. minikube bertujuan untuk memudahkan *developer* mengembangkan aplikasinya diatas kubernetes dan memudahkan pengguna baru untuk berlatih menggunakan kubernetes tanpa perlu membuat *cluster* sungguhan (menggunakan kubeadm atau *tools* lainnya).


ditutorial kali ini kita akan mencoba minikube dengan menggunakan *driver* kvm2. Sebenarnya minikube memiliki *driver* lainnya yang memungkinkan kita meluncurkan kubernetes *cluster*, sebagai contoh:
* none (Bare Metal)
* kvm2
* docker
* podman
* virtualbox
* vmware
* [lainnya](https://minikube.sigs.k8s.io/docs/drivers/) 

# Prasyarat
* libvirt v1.3.1 atau lebih tinggi
* qemu-kvm v2.0 atau lebih tinggi

# 1. Unduh dan memasang minikube

Terdapat 3 jenis kemasan minikube yang bisa diunduh yaitu binary, paket DEB, & paket RPM.

## Binary
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

## paket DEB
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
```

## paket RPM
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.x86_64.rpm
sudo rpm -ivh minikube-latest.x86_64.rpm
```

# 2. Jalankan minikube

Jalankan minikube menggunakan *driver* kvm2
```bash
minikube start --driver=kvm2
```

(**Fakultatif**) Agar minikube selalu menggunakan *driver* kvm2 jalankan perintah berikut
```bash
minikube config set driver kvm2
```

{% include figure image_path="/assets/images/kubernetes/2020-12-09-setup-minikube_1.png" caption="Ilustrasi menjalankan minikube" %}

# 3. Validasi minikube

Jalankan minikube kubectl
```
minikube kubectl -- get pods -A
```

{% include figure image_path="/assets/images/kubernetes/2020-12-09-setup-minikube_2.png" caption="Ilustrasi menjalankan perintah minikube kubectl" %}

# Referensi
[minikube docs](https://minikube.sigs.k8s.io/docs/)
