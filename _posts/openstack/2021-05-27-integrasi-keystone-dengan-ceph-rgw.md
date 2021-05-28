---
layout: single
title: Integrasi Keystone dengan Ceph RGW
category: openstack
tag:
- openstack
- ceph
- object storage
toc: true
toc_sticky: true
toc_icon: "book-reader"
---

Pada kesempatan kali ini saya akan membagikan cara bagaimana mengintegrasikan keystone dengan rados gateway.

**Catatan**<br>Liat tutorial deploy klaster ceph beserta dengan ceph rados gateway. Disini:<br>**Akan hadir**
{: .notice--info}

# Diagram
{% include figure image_path="/assets/images/openstack/keystone_radosgw/keystone_radosgw.png" caption="Diagram OpenStack & Ceph" %}

# Prasyarat
* Klaster Ceph dengan Ceph Object Gateway
* Klaster OpenStack

# Panduan

# 1. Tahap keystone

## 1.1. Buat service swift
```
openstack service create --name=swift \
  --description="Swift Service" \
  object-store
```

## 1.2. Buat endpoint service swift
```
openstack endpoint create --region RegionOne swift admin 'https://swift.btech.id/swift/v1/AUTH_$(project_id)s'
openstack endpoint create --region RegionOne swift internal 'https://swift.btech.id/swift/v1/AUTH_$(project_id)s'
openstack endpoint create --region RegionOne swift public 'https://swift.btech.id/swift/v1/AUTH_$(project_id)s'
```

# 2. Tahap ceph radosgw

## 2.1. Ubah konfigurasi ceph.conf
```
[client.rgw.aaceph01.rgw0]
host = aaceph01
keyring = /var/lib/ceph/radosgw/ceph-rgw.aaceph01.rgw0/keyring
log file = /var/log/ceph/ceph-rgw-aaceph01.rgw0.log
rgw frontends = beast endpoint=192.168.10.111:8080
rgw dns name = swift.btech.id

# Konfigurasi terkait keystone
rgw keystone api version = 3
rgw keystone url = https://internal.btech.id:35357
rgw keystone admin user = admin
rgw keystone admin password path = /etc/ceph/keystone-pass
rgw keystone admin domain = RegionOne
rgw keystone admin project = admin
rgw swift account in url = true
```

## 2.2. Buat file untuk menyimpan password
```
echo "{ADMIN_PASS}" > /etc/ceph/keystone-pass
sudo chmod 600 /etc/ceph/keystone-pass
sudo chown ceph:ceph /etc/ceph/keystone-pass 
```

## 2.3. Restart service daemon ceph-radosgw
```
sudo systemctl restart ceph-radosgw@{rgw_id} 
```

# Referensi
[Dokumen Ceph - Integrasi Keystone](https://docs.ceph.com/en/pacific/radosgw/keystone/)  
[Dokumen Ceph - Referensi Konfigurasi radosgw](https://docs.ceph.com/en/latest/radosgw/config-ref/#keystone-settings)
