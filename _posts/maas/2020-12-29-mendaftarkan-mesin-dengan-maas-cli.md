---
layout: single
title: Mendaftarkan mesin menggunakan MaaS CLI
category: maas
tags:
- maas
- maas-cli
- canonical
toc: true
toc_sticky: true
toc_icon: "book-reader"
---

Pada kesempatan kali ini saya akan membagikan cara bagaimana mendaftarkan mesin ke MaaS menggunakan CLI.

# Prasyarat
* MaaS sudah terinstall
* User MaaS admin sudah dibuat

# 1. Kumpulkan informasi mesin
## 1.1. Sebagai contoh kita memiliki mesin dengan informasi sebagai berikut

| MAC Address       | Power Type | IPMI Username | IPMI Password | IPMI IP Address |
|-------------------|------------|---------------|---------------|-----------------|
| 00:00:00:00:00:01 | ipmi       | admin         | rahasia       | 192.168.1.1     |

# 2. Login maas CLI
## 2.1. Dapatkan API key maas
```bash
export API_KEY=$(maas apikey --username admin --generate)
```

## 2.2. Masuk maas CLI menggunakan API Key yang telah didapat
```bash
maas login admin http://IP_ADDRESS_MAAS:5240/MAAS ${API_KEY}
```

# 3. Daftarkan mesin
## 3.1. Jalankan perintah maas dengan argumen sesuai dengan informasi server 
```bash
export hostname=<Nama mesin>
maas admin machines create \
    hostname=<Nama Mesin> \
    mac_addresses=00:00:00:00:00:01 \
    architecture=amd64 \
    power_type=ipmi \
    power_parameters_power_driver=LAN_2_0 \
    power_parameters_power_user=admin \
    power_parameters_power_pass=rahasia \
    power_parameters_power_address=192.168.1.1
```

## 3.2. Verifikasi mesin
```bash
maas admin machines read | jq '.[] | .hostname'
```

# Referensi
[MaaS Advanced CLI Tasks](https://maas.io/docs/advanced-cli-tasks)
