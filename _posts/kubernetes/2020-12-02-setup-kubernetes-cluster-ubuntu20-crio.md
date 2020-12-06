---
layout: single
title: Setup Kubernetes Cluster on Ubuntu 20.04 with CRI-O Engine
category: kubernetes
toc: true
toc_sticky: true
toc_icon: "book-reader"
---

Halo semua, dalam kesempatan kali ini saya akan berbagi cara *setup* Kubernetes dengan menggunakan `kubeadm` versi 1.19 di Ubuntu 20.04 (Focal) dengan menggunakan container engine CRI-O.

# Topologi
Berikut topologi yang digunakan:
![Topologi Kubernetes](/assets/images/kubernetes/2020-12-02-setup_kubernetes_cluster_v119.png)

| Nama VM      | NIC                  | Spesifikasi                        |
|--------------|----------------------|------------------------------------|
| k8s-master   | ens3: 10.10.10.51/24 | vCPU: 4<br>RAM: 8G<br>Storage: 80G |
| k8s-worker01 | ens3: 10.10.10.52/24 | vCPU: 4<br>RAM: 8G<br>Storage: 80G |
| k8s-worker02 | ens3: 10.10.10.53/24 | vCPU: 4<br>RAM: 8G<br>Storage: 80G |

**Catatan**<br>Tutorial ini dibuat dan diuji coba menggunakan KVM Guest atau Virtual Machine.
{: .notice--info}

# 1. *Setup* Host 

Lakukan disemua host

## 1.1. Masukkan setiap host ke dalam **/etc/hosts**
```bash
sudo vim /etc/hosts
```

* /etc/hosts

```bash
127.0.0.1 localhost
10.10.10.51 k8s-master
10.10.10.52 k8s-worker01
10.10.10.53 k8s-worker02
```

## 1.2. Memperbarui paket-paket pada host
```bash
sudo apt update
sudo apt upgrade
```

# 2. *Setup* CRI-O

Jalankan *command* di bawah pada semua host

## 2.1. Lakukan konfigurasi prasyarat
```bash
sudo modprobe overlay
sudo modprobe br_netfilter

# Set up required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

## 2.2. Pasang *repository* CRI-O
```bash
export OS=xUbuntu_20.04   # OS Version
export VERSION=1.19       # Cri-O Version

cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /
EOF
cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list
deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /
EOF

curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers-cri-o.gpg add -
```

## 2.3. Instal CRI-O
```bash
sudo apt-get update
sudo apt-get install cri-o cri-o-runc
```

## 2.4. Jalankan servis CRI-O
```bash
sudo systemctl enable --now crio
```

# 3. *Setup* Kubernetes

Jalankan *command* di bawah pada semua host

## 3.1. Pasang *repository* Kubernetes
```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

## 3.2. install paket-paket Kubernetes
```bash
sudo apt-get update
sudo apt-get install -y kubelet=1.19.4-00 kubeadm=1.19.4-00 kubectl=1.19.4-00
sudo apt-mark hold kubelet kubeadm kubectl
```

Jalankan *command* di bawah pada master

## 3.3. Membuat file konfigurasi *control plane* Kubernetes
```bash
vim init.yaml
```

* init.yaml

```bash
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.10.10.51               # Sesuaikan dengan IP Addr host
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/crio/crio.sock
  name: k8s-master                            # Sesuaikan dengan hostname host
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
controlPlaneEndpoint: k8s-master:6443         # Sesuaikan dengan hostname host
kind: ClusterConfiguration
kubernetesVersion: v1.19.4
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16                    # Menggunakan Subnet 10.244.0.0/16 untuk flannel
  serviceSubnet: 10.96.0.0/12
scheduler: {}
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

## 3.4. Inisiasi *cluster* menggunakan `kubeadm`
```bash
sudo kubeadm init --config init.yaml
```

## 3.5. (Opsional) Dapatkan *command* untuk join cluster
Catat output dari *command* di bawah
```bash
sudo kubeadm token create --print-join-command
```

## 3.6. Salin file `kubeconfig` agar user bisa mengakses *cluster* Kubernetes
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Jalankan *command* di bawah ini disemua worker

## 3.7. Worker bergabung ke *cluster*
*Command* di bawah hanyalah contoh
```bash
sudo kubeadm join k8s-master:6443 --token 12v372.zgar4m9gtvcy82t4 \
--discovery-token-ca-cert-hash sha256:8c4bf4cfda563e260751c6860ec67613a39a2b5df24dbda8b2b1d1256b12d201
```

Jalankan *command* di bawah pada master

## 3.8. Verifikasi worker sudah masuk kedalam *cluster* Kubernetes 

```bash
kubectl get nodes
```

Contoh output
```bash
student@k8s-master:~$ kubectl get nodes
NAME           STATUS   ROLES    AGE   VERSION
k8s-master     Ready    master   18h   v1.19.4
k8s-worker01   Ready    <none>   18h   v1.19.4
k8s-worker02   Ready    <none>   18h   v1.19.4
```

# 4. Memasang Container Network Interface (CNI)

Pada tutorial kali ini CNI yang digunakan adalah **flannel**

## 4.1. Pasang flannel dengan menggunakan *manifest*

Manifest bisa didapatkan [di sini](https://github.com/coreos/flannel)

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

## 4.2. Verifikasi *pods* flannel sudah berjalan
```bash
kubectl -n kube-system get pods --selector app=flannel -o wide
```

Contoh output
```bash
student@k8s-master:~$ kubectl -n kube-system get pods --selector app=flannel -o wide
NAME                    READY   STATUS    RESTARTS   AGE     IP            NODE           NOMINATED NODE   READINESS GATES
kube-flannel-ds-kdx92   1/1     Running   0          2m41s   10.10.10.52   k8s-worker01   <none>           <none>
kube-flannel-ds-nm6ws   1/1     Running   0          2m41s   10.10.10.53   k8s-worker02   <none>           <none>
kube-flannel-ds-qvpwb   1/1     Running   0          2m41s   10.10.10.51   k8s-master     <none>           <none>
```

# Referensi
[Instal Container Runtime - CRI-O](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cri-o)  
[*deploy* Kubernetes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)  
[kubeadm token](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/)  
[Flannel](https://github.com/coreos/flannel)
