# 004 — Subnetting & IP Address Planning

## 🎯 Tujuan Lab

Merancang IP addressing plan untuk sebuah kantor dengan beberapa divisi, menggunakan teknik subnetting VLSM (Variable Length Subnet Mask) dan mengimplementasikannya dengan VLAN + Router-on-a-Stick di Cisco Packet Tracer.

---

## 📖 Konsep yang Dipelajari

| Konsep | Deskripsi |
|--------|-----------|
| **Subnetting** | Membagi network besar menjadi subnet-subnet kecil sesuai kebutuhan |
| **VLSM** | Variable Length Subnet Mask — tiap subnet bisa punya ukuran berbeda |
| **IP Planning** | Menentukan alokasi IP untuk setiap segmen network secara terstruktur |
| **VLAN** | Virtual LAN — memisahkan broadcast domain secara logical di switch (Layer 2) |
| **Router-on-a-Stick** | 1 kabel fisik ke router, dipecah jadi sub-interface untuk routing antar VLAN |
| **Trunk** | Port switch yang membawa traffic banyak VLAN sekaligus dalam 1 kabel |

### VLSM vs VLAN — Apa Bedanya?

| | VLSM | VLAN |
|---|---|---|
| **Layer** | 3 (IP/Router) | 2 (Switch) |
| **Fungsi** | Bagi blok IP secara efisien | Pisahkan broadcast domain |
| **Dikonfigurasi di** | Router / IP planning | Switch |
| **Analogi** | Sistem penomoran ruangan | Dinding/sekat antar ruangan |

VLSM dan VLAN **bukan pengganti**, tapi **partner** — VLAN membuat "ruangan", VLSM membuat "penomoran IP" untuk setiap ruangan.

---

## 🏢 Skenario

Kamu adalah network engineer di **PT. Nusantara Digital**. Perusahaan memiliki 1 gedung kantor dengan 4 divisi.

**Network yang diberikan oleh ISP:** `192.168.10.0/24`

Tugasmu: bagi network ini menjadi subnet-subnet menggunakan VLSM agar efisien.

---

## 📋 VLSM Addressing Table

Network: `192.168.10.0/24` dipecah menjadi:

| No | Divisi | Jumlah Host | Prefix | Network Address | Subnet Mask | Range IP | Broadcast | Gateway |
|----|--------|-------------|--------|-----------------|-------------|----------|-----------|---------|
| 1 | Engineering | 50 | /26 (62 usable) | 192.168.10.0 | 255.255.255.192 | .1 - .62 | .63 | 192.168.10.1 |
| 2 | Marketing | 25 | /27 (30 usable) | 192.168.10.64 | 255.255.255.224 | .65 - .94 | .95 | 192.168.10.65 |
| 3 | Finance | 12 | /28 (14 usable) | 192.168.10.96 | 255.255.255.240 | .97 - .110 | .111 | 192.168.10.97 |
| 4 | Server Farm | 6 | /28 (14 usable) | 192.168.10.112 | 255.255.255.240 | .113 - .126 | .127 | 192.168.10.113 |
| 5 | Management | 5 | /29 (6 usable) | 192.168.10.128 | 255.255.255.248 | .129 - .134 | .135 | 192.168.10.129 |
| 6 | P2P Link | 2 | /30 (2 usable) | 192.168.10.136 | 255.255.255.252 | .137 - .138 | .139 | 192.168.10.137 |

```
192.168.10.0/24
├── .0   - .63    Engineering  /26  (62 usable)
├── .64  - .95    Marketing    /27  (30 usable)
├── .96  - .111   Finance      /28  (14 usable)
├── .112 - .127   Server Farm  /28  (14 usable)
├── .128 - .135   Management   /29  (6 usable)
├── .136 - .139   P2P Link    /30  (2 usable)
└── .140 - .255   FREE (future use)
```

### Cara Menghitung VLSM

1. Urutkan divisi dari **jumlah host terbesar** ke terkecil
2. Subnet pertama mulai dari network address (`192.168.10.0`)
3. Subnet berikutnya = **broadcast sebelumnya + 1**
4. Gateway = **network address + 1**
5. Broadcast = **network address + jumlah alamat - 1**

---

## 🖥️ Topologi

```
                        [R-CORE]
                  Gi0/0  Gi0/1   Gi0/2
                    │      │       │
                trunk  trunk   trunk
                    │      │       │
               [SW-ENG] [SW-OFFICE] [SW-SERVER]
               Fa0/24    Fa0/24     Fa0/24
                  │       │  │  │       │
               PC-Eng  PC-Mkt │ PC-Mgmt SRV-01
                       PC-Fin
```

### Perangkat

| Device | Model | Hostname |
|--------|-------|----------|
| Router | 2911 | R-CORE |
| Switch 1 | 2960 | SW-ENG |
| Switch 2 | 2960 | SW-OFFICE |
| Switch 3 | 2960 | SW-SERVER |
| PC | - | 1 per divisi (minimal) |
| Server | - | SRV-01 |

### VLAN Mapping

| VLAN ID | Name | Switch | Subnet |
|---------|------|--------|--------|
| 10 | ENGINEERING | SW-ENG | 192.168.10.0/26 |
| 20 | MARKETING | SW-OFFICE | 192.168.10.64/27 |
| 30 | FINANCE | SW-OFFICE | 192.168.10.96/28 |
| 40 | MANAGEMENT | SW-OFFICE | 192.168.10.128/29 |
| 50 | SERVER | SW-SERVER | 192.168.10.112/28 |

---

## ⚙️ Konfigurasi

### Urutan Konfigurasi

```
Step 1: Hostname semua device
Step 2: Buat VLAN di switch
Step 3: Assign access port ke VLAN
Step 4: Set trunk port ke router        ← JANGAN LUPA! (penyebab error paling umum)
Step 5: Konfigurasi sub-interface di router
Step 6: Assign IP ke PC/Server
Step 7: Verifikasi & Testing
```

> ⚠️ **Switch dulu, baru router.** Buat "ruangan" (VLAN) dulu, baru pasang "pintu" (gateway/sub-interface). Kalau router di-config duluan, sub-interface kirim traffic ke VLAN yang belum ada di switch.

---

### SW-ENG

```cisco
enable
configure terminal
hostname SW-ENG

! Buat VLAN
vlan 10
 name ENGINEERING
exit

! Assign access port untuk PC Engineering
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10
exit

! Trunk ke router (R-CORE Gi0/0)
interface FastEthernet0/24
 switchport mode trunk
exit
end
```

### SW-OFFICE

```cisco
enable
configure terminal
hostname SW-OFFICE

! Buat VLAN
vlan 20
 name MARKETING
vlan 30
 name FINANCE
vlan 40
 name MANAGEMENT
exit

! Assign access port
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 20
exit

interface FastEthernet0/2
 switchport mode access
 switchport access vlan 30
exit

interface FastEthernet0/3
 switchport mode access
 switchport access vlan 40
exit

! Trunk ke router (R-CORE Gi0/1)
interface FastEthernet0/24
 switchport mode trunk
exit
end
```

### SW-SERVER

```cisco
enable
configure terminal
hostname SW-SERVER

! Buat VLAN
vlan 50
 name SERVER
exit

! Assign access port untuk Server
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 50
exit

! Trunk ke router (R-CORE Gi0/2)
interface FastEthernet0/24
 switchport mode trunk
exit
end
```

### R-CORE

```cisco
enable
configure terminal
hostname R-CORE

! Nyalakan interface fisik
interface GigabitEthernet0/0
 no shutdown
exit

interface GigabitEthernet0/1
 no shutdown
exit

interface GigabitEthernet0/2
 no shutdown
exit

! Sub-interface VLAN 10 — Engineering (via SW-ENG)
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.192
exit

! Sub-interface VLAN 20 — Marketing (via SW-OFFICE)
interface GigabitEthernet0/1.20
 encapsulation dot1Q 20
 ip address 192.168.10.65 255.255.255.224
exit

! Sub-interface VLAN 30 — Finance (via SW-OFFICE)
interface GigabitEthernet0/1.30
 encapsulation dot1Q 30
 ip address 192.168.10.97 255.255.255.240
exit

! Sub-interface VLAN 40 — Management (via SW-OFFICE)
interface GigabitEthernet0/1.40
 encapsulation dot1Q 40
 ip address 192.168.10.129 255.255.255.248
exit

! Sub-interface VLAN 50 — Server Farm (via SW-SERVER)
interface GigabitEthernet0/2.50
 encapsulation dot1Q 50
 ip address 192.168.10.113 255.255.255.240
exit
end
```

### End Devices — IP Assignment

Klik PC/Server → Desktop → IP Configuration:

| Device | IP Address | Subnet Mask | Default Gateway |
|--------|------------|-------------|-----------------|
| PC-Eng | 192.168.10.2 | 255.255.255.192 | 192.168.10.1 |
| PC-Mkt | 192.168.10.66 | 255.255.255.224 | 192.168.10.65 |
| PC-Fin | 192.168.10.98 | 255.255.255.240 | 192.168.10.97 |
| PC-Mgmt | 192.168.10.130 | 255.255.255.248 | 192.168.10.129 |
| SRV-01 | 192.168.10.114 | 255.255.255.240 | 192.168.10.113 |

---

## ✅ Verifikasi & Testing

### 1. Cek VLAN di Switch

```cisco
show vlan brief
```

Pastikan VLAN ada dan port assignment benar.

### 2. Cek Trunk di Switch

```cisco
show interfaces trunk
```

Pastikan port trunk muncul dan VLAN allowed. **Kalau output kosong = trunk belum aktif.** Ini penyebab paling umum inter-VLAN gagal.

### 3. Cek Sub-interface di Router

```cisco
show ip interface brief
```

Semua sub-interface harus **up/up** dengan IP yang sesuai VLSM table.

### 4. Cek Routing Table

```cisco
show ip route
```

Semua subnet harus muncul sebagai **Connected (C)**:

```
C    192.168.10.0/26   is directly connected, GigabitEthernet0/0.10
C    192.168.10.64/27  is directly connected, GigabitEthernet0/1.20
C    192.168.10.96/28  is directly connected, GigabitEthernet0/1.30
C    192.168.10.112/28 is directly connected, GigabitEthernet0/2.50
C    192.168.10.128/29 is directly connected, GigabitEthernet0/1.40
```

### 5. Ping Test

```
# Dari PC-Eng, ping semua:
ping 192.168.10.1      ← gateway sendiri (harus reply)
ping 192.168.10.66     ← PC Marketing (inter-VLAN)
ping 192.168.10.98     ← PC Finance (inter-VLAN)
ping 192.168.10.130    ← PC Management (inter-VLAN)
ping 192.168.10.114    ← Server (inter-VLAN)
```

Semua harus **Reply**. Kalau timeout, lihat Troubleshooting di bawah.

---

## 🔧 Troubleshooting Guide

### Ping ke Gateway Gagal (PC ↔ Router)

| Cek | Command | Yang Dicari |
|-----|---------|-------------|
| VLAN ada di switch? | `show vlan brief` | VLAN muncul dengan port yang benar |
| Port masuk VLAN yang benar? | `show vlan brief` | Port PC tercantum di VLAN yang sesuai |
| Trunk aktif? | `show interfaces trunk` | Port ke router muncul sebagai trunk |
| Sub-interface up? | `show ip interface brief` (di router) | Status & Protocol = up/up |
| Kabel nyambung? | Lihat visual di Packet Tracer | Kedua ujung kabel harus **hijau** |

### Ping Antar VLAN Gagal (PC ↔ PC beda VLAN)

| Cek | Command | Yang Dicari |
|-----|---------|-------------|
| PC bisa ping gateway sendiri? | `ping [gateway]` dari PC | Kalau gagal, masalah di VLAN/trunk (lihat atas) |
| Gateway kedua VLAN up? | `show ip interface brief` (di router) | Kedua sub-interface harus up/up |
| IP PC benar? | Cek di PC → Desktop → IP Config | IP, subnet mask, dan gateway sesuai tabel |
| Subnet mask benar? | Cek di PC → Desktop → IP Config | Salah subnet mask = gak bisa route |
| Routing table lengkap? | `show ip route` (di router) | Semua subnet muncul sebagai Connected |

### Troubleshooting Flowchart

```
PC gak bisa ping PC lain?
│
├── Coba ping gateway sendiri dulu
│   ├── ❌ Gateway gak reply
│   │   ├── Cek VLAN assignment di switch (show vlan brief)
│   │   ├── Cek trunk port (show interfaces trunk)
│   │   ├── Cek sub-interface (show ip interface brief)
│   │   └── Cek kabel (hijau di kedua ujung?)
│   │
│   └── ✅ Gateway reply
│       ├── Cek IP, subnet mask, gateway PC tujuan
│       ├── Cek VLAN & trunk di switch tujuan
│       └── Cek sub-interface VLAN tujuan di router
```

### Kesalahan Umum

| Masalah | Penyebab | Solusi |
|---------|----------|--------|
| Trunk tidak aktif | Lupa `switchport mode trunk` | Config trunk di port ke router |
| IP salah oktet | Pakai `192.168.20.x` bukan `192.168.10.x` | Ganti IP sesuai VLSM table |
| Subnet mask salah | Tukar subnet mask antar divisi | Cek tabel VLSM, setiap prefix beda mask |
| Interface down | Lupa `no shutdown` di router | Masuk interface fisik, ketik `no shutdown` |
| PC salah VLAN | Port access assign ke VLAN yang salah | Cek `show vlan brief`, reassign port |
| encapsulation belum di-set | Lupa `encapsulation dot1Q [vlan-id]` | Tambahkan di sub-interface router |

### Command Cheat Sheet

| Command | Dimana | Fungsi |
|---------|--------|--------|
| `show vlan brief` | Switch | Lihat semua VLAN dan port assignment |
| `show interfaces trunk` | Switch | Lihat trunk port aktif |
| `show interfaces Fa0/1 switchport` | Switch | Detail mode port tertentu |
| `show ip interface brief` | Router | Status semua interface dan IP |
| `show ip route` | Router | Routing table |
| `no vlan [id]` | Switch (config) | Hapus VLAN |
| `hostname [name]` | Semua device (config) | Ganti hostname |

---

## 🏭 Production Best Practices

1. **Selalu dokumentasikan IP plan** — di production, ini jadi source of truth. Tanpa dokumentasi, troubleshoot jadi nightmare.

2. **Gateway selalu di `.1`** — konsistensi = less human error.

3. **Sisakan ruang untuk growth** — jangan pakai `/26` kalau divisi Engineering mungkin berkembang ke 80 orang. Di production, pertimbangkan 2-3 tahun ke depan.

4. **Reserved IP ranges:**
   - `.1` → Gateway
   - `.2 - .10` → Network devices (AP, switch management, dll)
   - `.11 - .254` → End devices / DHCP pool

5. **Jangan pakai `/24` untuk semuanya** — flat network = broadcast storm, security risk, dan susah di-manage.

6. **Point-to-Point link pakai `/30` atau `/31`** — hemat IP, standar di production untuk link antar router.

7. **Trunk port harus explicit** — jangan rely pada DTP auto negotiation. Selalu set `switchport mode trunk` secara manual.

8. **Verifikasi setelah config** — selalu cek `show vlan brief`, `show interfaces trunk`, dan `show ip interface brief` setelah konfigurasi.

---

## 📝 Catatan untuk README di GitHub


- Screenshot topologi dari Packet Tracer
  <img width="1441" height="1131" alt="image" src="https://github.com/user-attachments/assets/30ab3195-32d2-41ff-af72-f8cefd2318e6" />

- Hasil ping test (screenshot)
- File `.pkt` (Packet Tracer save file)

---

## ⏭️ Lab Selanjutnya

→ **005_DHCP_Server** — Konfigurasi DHCP di router + DHCP relay agar tidak perlu assign IP manual ke setiap device.
