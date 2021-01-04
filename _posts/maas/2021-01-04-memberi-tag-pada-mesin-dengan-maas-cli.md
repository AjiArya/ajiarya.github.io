---
layout: single
title: Memberikan tag ke mesin menggunakan MaaS CLI
category: maas
tags:
- maas
- maas-cli
- canonical
toc: true
toc_sticky: true
toc_icon: "book-reader"
---

Pada kesempatan kali ini saya akan membagikan cara bagaimana memberikan tag ke mesin menggunakan menggunakan CLI

# Prasyarat
* MaaS sudah terinstall
* User MaaS admin sudah dibuat

# 1. Kumpulkan informasi mesin
## 1.1. Sebagai contoh kita memiliki mesin dengan informasi sebagai berikut

| MAC Address       | Power Type | IPMI Username | IPMI Password | IPMI IP Address | Hostname |
|-------------------|------------|---------------|---------------|-----------------|-----------------|
| 00:00:00:00:00:01 | ipmi       | admin         | rahasia       | 192.168.1.1     | samplemachine |

# 2. Login maas CLI
## 2.1. Dapatkan API key maas
```bash
export API_KEY=$(maas apikey --username admin --generate)
```

## 2.2. Masuk maas CLI menggunakan API Key yang telah didapat
```bash
maas login admin http://IP_ADDRESS_MAAS:5240/MAAS ${API_KEY}
```

# 3. Berikan tag pada mesin
## 3.1. Jalankan perintah maas dengan argumen sesuai dengan informasi server 
```bash
export hostname=<Nama mesin>

# Contoh
export hostname=samplemachine
export systemid=$(maas admin machines read | jq -r '.[] | select(.hostname=='\"$hostname\"') | .system_id')

# Buat tag jika belum dibuat
maas admin tags create name=<Nama tag>

# Contoh
maas admin tags create name=tagcontoh

# Berikan tag pada mesin
maas admin tag update-nodes <Nama tag> add=$systemid

# Contoh
maas admin tag update-nodes tagcontoh add=$systemid
```

## 3.2. Verifikasi mesin
Baca tag mesin
```bash
maas admin machines read | jq '.[] | {hostname: .hostname, tags: .tag_names}' --compact-output
```

atau

Daftar mesin yang menggunakan tag `tagcontoh`
```bash
maas admin tag machines tagcontoh | jq '.[] | .hostname'
```

# Referensi
[CLI Tag Management](https://maas.io/docs/cli-tag-management)
