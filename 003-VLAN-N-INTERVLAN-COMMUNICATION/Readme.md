# 🧪 Lab 03 — VLAN 

Lab ini membahas segmentasi jaringan menggunakan VLAN dan komunikasi antar VLAN menggunakan router.

---

## Objectives
- Memahami konsep VLAN
- Segmentasi network


---

## What is VLAN?

VLAN (Virtual LAN) adalah pembagian jaringan secara logical dalam satu switch fisik.

Tujuan:
- Membatasi broadcast
- Meningkatkan keamanan
- Mempermudah manajemen

---

## VLAN Types

| Type       | Description                    |
|------------|--------------------------------|
| Default    | VLAN 1 bawaan switch            |
| Data       | VLAN user                       |
| Management | VLAN manajemen device           |
| Native     | VLAN tanpa tag di trunk         |
| Voice      | VLAN IP Phone                   |

---

## Device

| Device | Model      |
|--------|------------|
| Router | Cisco 1941 |
| Switch | Cisco 2960 |
| PC     | Generic    |

---

## Topology

```
PC1(V10)  PC2(V10)
    \      /
     Switch --- Router
    /      \
PC3(V20)  PC4(V20)
```

---

## VLAN & IP Plan

### VLAN

| VLAN | Name | Network        |
|------|------|----------------|
| 10   | HR   | 192.168.10.0/24|
| 20   | IT   | 192.168.20.0/24|

### IP

| Device | VLAN | IP             | Gateway        |
|--------|------|----------------|----------------|
| PC1    | 10   | 192.168.10.2   | 192.168.10.1   |
| PC2    | 10   | 192.168.10.3   | 192.168.10.1   |
| PC3    | 20   | 192.168.20.2   | 192.168.20.1   |
| PC4    | 20   | 192.168.20.3   | 192.168.20.1   |

---

---

## 🔍 Check Available Interfaces

Sebelum melakukan konfigurasi, pastikan terlebih dahulu nama interface yang tersedia
pada router dan switch untuk menghindari kesalahan konfigurasi.

Gunakan perintah berikut:
```
show ip interface brief
```


Contoh output:

```
Interface IP-Address OK? Method Status Protocol
FastEthernet0/0 unassigned YES unset up up
FastEthernet0/1 unassigned YES unset down down
GigabitEthernet0/0 unassigned YES unset up up
```

Keterangan:
- **Interface** : Nama port yang tersedia
- **IP-Address** : IP yang terpasang
- **Status/Protocol** : Kondisi interface (harus `up/up`)
- **up/up** : Interface aktif
- **down/down** : Interface mati / belum terhubung

Langkah ini penting untuk memastikan apakah device menggunakan
FastEthernet atau GigabitEthernet sebelum melakukan konfigurasi.


## Switch Configuration

```
enable
conf t

vlan 10
 name HR
vlan 20
 name IT
exit

interface range fa0/1-2
 switchport mode access
 switchport access vlan 10
exit

interface range fa0/3-4
 switchport mode access
 switchport access vlan 20
exit

interface fa0/24
 switchport mode trunk
exit
```

---

## Router Configuration

```
enable
conf t

interface fa0/0
 no shutdown
exit

interface fa0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
exit

interface fa0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
exit

end
write
```

---

## Testing

```
show vlan brief
show ip int brief
ping 192.168.20.2
```

---

## Difference: Switch vs Router

| Aspect  | Switch | Router |
|---------|--------|--------|
| Layer   | L2     | L3     |
| Address | MAC    | IP     |
| Routing | No     | Yes    |
| Gateway | No     | Yes    |

---

## Result

VLAN 10 dan VLAN 20 berhasil berkomunikasi melalui router.

---

## Author
Amir
