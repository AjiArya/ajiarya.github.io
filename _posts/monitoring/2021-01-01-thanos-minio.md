---
layout: single
title: Thanos Prometheus dengan Minio
category: monitoring
tags:
- thanos
- prometheus
- monitoring
- minio
toc: true
toc_sticky: true
toc_icon: "book-reader"
---

Pada kesempatan kali ini saya akan membagikan cara bagaimana setup Thanos untuk pengumpulan metrik dari banyak Prometheus dan kemudian datanya akan dimasukkan ke *object storage* Minio

# Topologi
{% include figure image_path="/assets/images/monitoring/thanos_minio/thanos_topology_whitebg.png" caption="Topologi Thanos" %}

| Nama VM | NIC                                                               | Spesifikasi                                                                        |
|---------|-------------------------------------------------------------------|------------------------------------------------------------------------------------|
| thanos  | ens3: 192.168.10.60<br>ens4: 192.168.20.60<br>ens5: 192.168.30.60 | vCPU: 2<br>RAM: 6G<br>Storage: 50G<br>OS: Ubuntu 20.04 (Focal)                     |
| minio   | ens3: 192.168.10.59<br>ens4: 192.168.20.59<br>ens5: 192.168.30.59 | vCPU: 2<br>RAM: 6G<br>Storage: 50G, 10G, 10G, 10G, 10G<br>OS: Ubuntu 20.04 (Focal) |
| prom01  | ens3: 192.168.10.61                                               | vCPU: 2<br>RAM: 4G<br>Storage: 50G<br>OS: Ubuntu 20.04 (Focal)                     |
| prom02  | ens3: 192.168.20.61                                               | vCPU: 2<br>RAM: 4G<br>Storage: 50G<br>OS: Ubuntu 20.04 (Focal)                     |
| prom03  | ens3: 192.168.30.61                                               | vCPU: 2<br>RAM: 4G<br>Storage: 50G<br>OS: Ubuntu 20.04 (Focal)                     |


**Catatan**<br>Tutorial ini dibuat dan diuji coba menggunakan KVM Guest atau Virtual Machine.
{: .notice--info}

# 1. *Setup* Host
Lakukan di host `thanos` dan `minio` 

## 1.1. Masukkan tiap host ke dalam /etc/hosts
```bash
sudo vim /etc/hosts
```

* /etc/hosts

```bash
127.0.0.1 localhost
192.168.10.59 minio
192.168.10.60 thanos
192.168.10.61 prom01
192.168.20.61 prom02
192.168.30.61 prom03
```

## 1.2. Memperbarui paket-paket pada host
```bash
sudo apt update
sudo apt upgrade
```

# 2. *Setup* minio
Lakukan tahapan berikut pada host `minio`

## 2.1. Setup disk untuk minio
Format disk
```bash
sudo mkfs.xfs /dev/vdb
sudo mkfs.xfs /dev/vdc
sudo mkfs.xfs /dev/vdd
sudo mkfs.xfs /dev/vde
```

Sunting file `/etc/fstab` tambahkan baris-baris berikut
```bash
sudo vim /etc/fstab
```

* /etc/fstab

```bash
/dev/vdb /mnt/data1 xfs defaults 0 0
/dev/vdc /mnt/data2 xfs defaults 0 0
/dev/vdd /mnt/data3 xfs defaults 0 0
/dev/vde /mnt/data4 xfs defaults 0 0
```

Buat direktori untuk mounting data minio
```bash
sudo mkdir /mnt/data{1..4}
```

*Mount* direktori
```bash
mount -a
```

## 2.2. Unduh minio
Kunjungi: [Situs unduh minio](https://min.io/download#/linux)

```bash
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv minio /usr/local/bin/minio
```

## 2.3. Buat service minio
Buat user minio-user
```bash
sudo useradd -r minio-user -s /sbin/nologin
```

Ganti pemilik direktori `/mnt/data{1..4}`
```bash
sudo chown minio-user:minio-user /mnt/data{1..4}
```

Buat file *environment variable*
```bash
sudo vim /etc/default/minio
```

* /etc/default/minio

```text
MINIO_ACCESS_KEY="minio"
MINIO_VOLUMES="/mnt/data{1...4}"
MINIO_OPTS="-C /etc/minio --address 0.0.0.0:9000"
MINIO_SECRET_KEY="miniostorage"
```

Unduh file service systemd
```bash
sudo wget https://raw.githubusercontent.com/minio/minio-service/master/linux-systemd/minio.service \
    -O /etc/systemd/system/minio.service
```


## 2.4. Jalankan minio
```bash
sudo systemctl enable --now minio
sudo systemctl status minio
```

## 2.5. Buat Bucket `thanos`

* Login Minio Browser lalu buat bucket dengan nama `thanos`

# 3. *Setup* Prometheus, Node Exporter, & Thanos (sidecar & store)
Lakukan tahapan berikut pada host `prom01`, `prom02`, & `prom03`

## 3.1. Unduh & pasang Prometheus
Buat user `prometheus`
```bash
sudo useradd --no-create-home --shell /bin/false prometheus
```

Unduh & pasang Prometheus
```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.23.0/prometheus-2.23.0.linux-amd64.tar.gz
tar xf prometheus-2.23.0.linux-amd64.tar.gz
sudo cp prometheus-2.23.0.linux-amd64/prometheus /usr/local/bin/prometheus
```

Buat file konfigurasi Prometheus
```bash
sudo mkdir /etc/prometheus
sudo bash -c 'cat <<EOF > /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: site{1..3} # Sunting bagian ini sesuaikan dengan urutan
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets: 
      - localhost:9090
  - job_name: 'node_exporter'
    static_configs:
    - targets:
      - localhost:9100
  - job_name: 'thanos-sidecar'
    static_configs:
    - targets:
      - localhost:10902
EOF'
```

Buat file service Prometheus
```bash
sudo mkdir /var/lib/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
sudo bash -c 'cat<<EOF > /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file /etc/prometheus/prometheus.yml \
  --storage.tsdb.path /var/lib/prometheus/ \
  --storage.tsdb.max-block-duration=2h \
  --storage.tsdb.min-block-duration=2h \
  --web.enable-admin-api \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
EOF'
```

Jalankan service Prometheus
```bash
sudo systemctl enable --now prometheus
sudo systemctl status prometheus
```

## 3.2. Unduh & pasang Node Exporter
Buat user `node_exporter`
```bash
sudo useradd --no-create-home --shell /bin/false node_exporter
```

Unduh & pasang Node Exporter
```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.0.1/node_exporter-1.0.1.linux-amd64.tar.gz
tar xf node_exporter-1.0.1.linux-amd64.tar.gz
sudo cp node_exporter-1.0.1.linux-amd64/node_exporter /usr/local/bin/node_exporter
```

Buat file service Node Exporter
```bash
sudo bash -c 'cat<<EOF > /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF'
```

Jalankan service Node Exporter
```bash
sudo systemctl enable --now node_exporter
sudo systemctl status node_exporter
```

## 3.3. Unduh & pasang Thanos
Kunjungi: https://github.com/thanos-io/thanos/releases

Unduh & pasang Thanos
```bash
wget https://github.com/thanos-io/thanos/releases/download/v0.17.2/thanos-0.17.2.linux-amd64.tar.gz
tar xf thanos-0.17.2.linux-amd64.tar.gz
sudo cp thanos-0.17.2.linux-amd64/thanos /usr/local/bin/thanos
```

Buat file konfigurasi untuk Thanos
```bash
sudo bash -c 'cat<<EOF > /etc/prometheus/bucket.yml
type: S3
config:
  bucket: thanos
  endpoint: ${IP_MINIO}:9000 # Sunting baris ini
  insecure: true
  access_key: minio
  secret_key: miniostorage
EOF'
```

Verifikasi konfigurasi akses ke *object storage*
```bash
thanos tools bucket ls --objstore.config-file=/etc/prometheus/bucket.yml
```

Contoh keluaran (berhasil)
```bash
student@prom01:~$ thanos tools bucket ls --objstore.config-file=/etc/prometheus/bucket.yml
level=info ts=2021-01-01T09:21:43.120094815Z caller=main.go:98 msg="Tracing will be disabled"
level=info ts=2021-01-01T09:21:43.120414407Z caller=factory.go:46 msg="loading bucket configuration"
level=info ts=2021-01-01T09:21:43.629540968Z caller=fetcher.go:458 component=block.BaseFetcher msg="successfully synchronized block metadata" duration=508.442945ms cached=0 returned=0 partial=0
level=info ts=2021-01-01T09:21:43.629995442Z caller=tools_bucket.go:261 msg="ls done" objects=0
level=info ts=2021-01-01T09:21:43.633908676Z caller=main.go:160 msg=exiting
```

Buat file service Thanos Sidecar
```bash
sudo bash -c 'cat<<EOF > /etc/systemd/system/thanos-sidecar.service
[Unit]
Description=Thanos Sidecar
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/thanos sidecar \
    --prometheus.url=http://0.0.0.0:9090 \
    --grpc-address=0.0.0.0:10901 \
    --http-address=0.0.0.0:10902 \
    --tsdb.path /var/lib/prometheus/ \
    --objstore.config-file /etc/prometheus/bucket.yml

[Install]
WantedBy=multi-user.target
EOF'
```

Buat file service Thanos Store
```bash
sudo bash -c 'cat<<EOF > /etc/systemd/system/thanos-store.service
[Unit]
Description=Thanos Store
Wants=network-online.target
After=network-online.target

[Service]
User=root
Group=root
Type=simple
ExecStart=/usr/local/bin/thanos store \
  --data-dir=/var/lib/prometheus-store/ \
  --objstore.config-file=/etc/prometheus/bucket.yml \
  --http-address=0.0.0.0:10906 \
  --grpc-address=0.0.0.0:10905

[Install]
WantedBy=multi-user.target
EOF'
```

Jalankan Thanos Sidecar & Thanos Store
```bash
sudo systemctl enable --now thanos-sidecar thanos-store
sudo systemctl status thanos-sidecar thanos-store
```

# 4. Install Thanos Querier/Query
Lakukan tahapan berikut pada host `thanos`

## 4.1. Unduh dan pasang Thanos
```bash
wget https://github.com/thanos-io/thanos/releases/download/v0.17.2/thanos-0.17.2.linux-amd64.tar.gz
tar xf thanos-0.17.2.linux-amd64.tar.gz
sudo mv thanos-0.17.2.linux-amd64/thanos /usr/local/bin/thanos
```

## 4.2. Buat service Thanos Querier/Query
```bash
sudo bash -c 'cat<<EOF > /etc/systemd/system/thanos-querier.service
[Unit]
Description=Thanos Query
Wants=network-online.target
After=network-online.target

[Service]
User=root
Group=root
Type=simple
ExecStart=/usr/local/bin/thanos query \
     --http-address=0.0.0.0:9090 \
     --grpc-address=0.0.0.0:10903 \
     --store=prom01:10901 \
     --store=prom02:10901 \
     --store=prom03:10901

[Install]
WantedBy=multi-user.target
EOF'
```

## 4.3. Jalankan service Thanos Querier/Query
```bash
sudo systemctl enable --now thanos-querier
sudo systemctl status thanos-querier
```

# Hasil

* IP_HOST_THANOS:9090
{% include figure image_path="/assets/images/monitoring/thanos_minio/thanos.png" caption="Thanos Dashboard" %}

* IP_HOST_MINIO:9090
{% include figure image_path="/assets/images/monitoring/thanos_minio/minio.png" caption="Penyimpanan metrik Minio" %}

Dengan Thanos kita dimudahkan untuk mengumpulkan metrik dari banyak Prometheus sehingga lebih tersentralisasi dan dengan itu jika kita ingin melihat metrik atau menampilkan grafik dari metrik tersebut kita hanya perlu menggunakan endpoint Thanos saja

Sekian, Terima Kasih

# Referensi
[Logo Thanos](https://cncf-branding.netlify.app/img/projects/thanos/icon/color/thanos-icon-color.png)  
[Logo Prometheus](https://upload.wikimedia.org/wikipedia/commons/thumb/3/38/Prometheus_software_logo.svg/1200px-Prometheus_software_logo.svg.png)  
[Logo Minio](https://min.io/resources/img/logo/MINIO_Bird.png)  
[Dokumentasi Minio](https://docs.min.io/)  
[Dokumentasi Thanos](https://thanos.io/tip/thanos/getting-started.md/)  
[Setup Minio](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-object-storage-server-using-minio-on-ubuntu-18-04)  
[Setup Thanos](https://medium.com/@mail2ramunakerikanti/thanos-for-prometheus-f7f111e3cb75)
