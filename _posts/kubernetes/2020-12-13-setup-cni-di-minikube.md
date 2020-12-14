---
layout: single
title: Setup CNI di minikube
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

Halo semua, pada kesempatan kali ini saya akan membagikan cara bagaimana untuk memasang CNI di minikube. Secara *default* minikube akan menggunakan CNI *auto* atau *bridge* lalu bagaimana jika ingin menggunakan CNI lain? Jawabannya adalah dengan menggunakan flag `--cni` ketika menjalankan minikube.

Kenapa saya perlu mengganti CNI? semisalnya ingin mencoba fitur **Network Policy** Kubernetes kita memerlukan CNI yang bisa mendukung fitur tersebut.

Berikut list CNI yang bisa digunakan:
- bridge
- calico
- cilium
- flannel
- kindnet
- atau *path file* manifest CNI

# Prasyarat
* Minikube sudah terinstal (Jika belum ikuti [Tutorial ini](/kubernetes/setup-minikube-di-kvm))

# Buat cluster dengan minikube

Buat cluster yang menggunakan CNI

```bash
minikube start --cni <CNI>
```

Contoh
```bash
minikube start --cni calico
```
