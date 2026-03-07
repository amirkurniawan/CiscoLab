# 010 — NAT & PAT (Network Address Translation)

## 🎯 Tujuan Lab

Mengkonfigurasi NAT dan PAT agar semua device internal (private IP) bisa **akses internet menggunakan 1 public IP**. Tanpa NAT, private IP (192.168.x.x) tidak bisa berkomunikasi ke internet — karena ISP tidak mengenal dan tidak mau routing private IP.

---

## 📖 Konsep yang Dipelajari

| Konsep | Deskripsi |
|--------|-----------|
| **NAT** | Network Address Translation — ubah IP private jadi IP public |
| **Static NAT** | 1 private IP → 1 public IP (permanent, 1:1) |
| **Dynamic NAT** | Pool private IP → pool public IP (first-come first-served) |
| **PAT (Overload)** | Semua private IP → 1 public IP (pakai port number untuk bedakan) |
| **Inside Local** | IP private device (192.168.x.x) |
| **Inside Global** | IP public yang dilihat internet |
| **Outside Local/Global** | IP tujuan di internet |

### Analogi NAT

```
Tanpa NAT:
  Kamu kirim surat dari alamat "Kamar 101, Kost Pak Budi"
  → Kantor pos: "Alamat ini gak ada di peta kota, gak bisa dikirim" ❌

Dengan NAT:
  Kamu kirim surat dari alamat "Kamar 101, Kost Pak Budi"
  → Pak Budi (router) ganti alamat pengirim jadi "Jl. Merdeka No. 5" (public)
  → Kantor pos: "Oh, alamat valid, saya kirim" ✅
  → Balasan datang ke "Jl. Merdeka No. 5"
  → Pak Budi cek catatan: "Oh ini untuk Kamar 101"
  → Forward ke kamu ✅
```

### Kenapa Private IP Gak Bisa ke Internet

```
IP Private (RFC 1918):
  10.0.0.0/8
  172.16.0.0/12
  192.168.0.0/16

ISP dan router internet dikonfigurasi untuk DROP paket dari IP private.
Alasan: IP private bisa dipakai siapa saja — tidak unik.
→ ISP gak tahu harus reply ke mana.
```

### 3 Tipe NAT

#### Static NAT (1:1)

```
192.168.10.114 (Server) ←→ 200.0.0.10 (Public)

Selalu sama, permanent. Dipakai untuk server yang harus
diakses dari luar (web server, mail server).
```

#### Dynamic NAT (Many:Many)

```
192.168.10.11 → 200.0.0.10 (ambil dari pool)
192.168.10.12 → 200.0.0.11 (ambil dari pool)
192.168.10.13 → 200.0.0.12 (ambil dari pool)
192.168.10.14 → ❌ pool habis, tunggu!

Pool: 200.0.0.10 - 200.0.0.12 (cuma 3 public IP)
Kalau pool habis, user baru gak bisa akses internet.
```

#### PAT / Overload (Many:1) ← Paling Umum

```
192.168.10.11:1025 → 200.0.0.1:1025
192.168.10.12:1026 → 200.0.0.1:1026
192.168.10.13:1027 → 200.0.0.1:1027
192.168.10.14:1028 → 200.0.0.1:1028
... ratusan device  → 200.0.0.1:xxxx

1 public IP, dibedakan pakai PORT NUMBER.
Ini yang dipakai di rumah kamu (WiFi router) dan 99% production.
```

### Perbandingan

| | Static NAT | Dynamic NAT | PAT (Overload) |
|---|---|---|---|
| Rasio | 1 private : 1 public | N private : N public | N private : 1 public |
| Public IP | Banyak (mahal) | Beberapa | **1 saja (murah)** |
| Use case | Server yang harus diakses dari luar | Jarang dipakai | **Default untuk semua** |
| Di production | ✅ Untuk server | ❌ Hampir gak dipakai | ✅ **Paling umum** |

---

## 🏢 Skenario

PT. Nusantara Digital mau konek ke **internet**. ISP kasih:

- **1 public IP:** 200.0.0.2/30 (untuk WAN link ke ISP)
- **1 public IP tambahan:** 200.0.0.10 (untuk Static NAT ke web server)

Requirement:
1. **Semua PC** bisa akses internet → PAT (overload) via 200.0.0.2
2. **Web Server (SRV-01)** bisa diakses dari internet → Static NAT ke 200.0.0.10

---

## 📋 IP Addressing Table

### Internal Network (dari Lab sebelumnya)

| VLAN | Divisi | Subnet | Gateway |
|------|--------|--------|---------|
| 10 | Engineering | 192.168.10.0/26 | 192.168.10.1 |
| 20 | Marketing | 192.168.10.64/27 | 192.168.10.65 |
| 30 | Finance | 192.168.10.96/28 | 192.168.10.97 |
| 40 | Management | 192.168.10.128/29 | 192.168.10.129 |
| 50 | Server Farm | 192.168.10.112/28 | 192.168.10.113 |

### WAN / Internet

| Device | Interface | IP Address | Subnet Mask | Keterangan |
|--------|-----------|------------|-------------|------------|
| R-CORE | Gi0/2 atau Se0/0/0 | 200.0.0.2 | 255.255.255.252 | WAN ke ISP |
| R-ISP | Gi0/0 atau Se0/0/0 | 200.0.0.1 | 255.255.255.252 | ISP side |
| R-ISP | Gi0/1 | 8.8.8.0/24 | 255.255.255.0 | Simulasi internet |
| Internet-Server | NIC | 8.8.8.8 | 255.255.255.0 | Simulasi server internet |

> 💡 **200.0.0.0/30** = WAN link ke ISP (point-to-point, cuma 2 IP). **8.8.8.0/24** = simulasi "internet" di Packet Tracer.

---

## 🖥️ Topologi

Tambah R-ISP dan Internet Server ke topologi Lab sebelumnya.

```
  [PC-Eng]  [PC-Mkt] [PC-Fin] [PC-Mgmt]  [SRV-01]
     │         │        │        │           │
  [SW-ENG] [SW-OFFICE]              [SW-SERVER]
     │         │                        │
     └────┬────┘────────────────────────┘
          │
      [R-CORE]
    Inside │ Gi0/0, Gi0/1, Gi0/2.50
           │
    Outside│ Se0/0/0 (200.0.0.2)
           │
      ═════╪═════ WAN Link (200.0.0.0/30)
           │
           │ Se0/0/0 (200.0.0.1)
      [R-ISP]
           │ Gi0/1 (8.8.8.1)
           │
    [Internet-Server]
       (8.8.8.8)
       Simulasi Internet
```

### Perangkat Baru

Pastikan sebelum memasangkan kabel WAN ke R-ISP dan menambahkan HWIC-2T untuk kabel serial, diwajibkan untuk save config router
```
copy running-config startup-config
atau
wr
```

```
! Verifikasi sudah tersave
show startup-config
Kalau output-nya sama dengan show running-config → aman.
```

| Device | Model | Hostname | Fungsi |
|--------|-------|----------|--------|
| Router | 2911 | R-ISP | Simulasi ISP router |
| Server | - | Internet-Server | Simulasi web server di internet (8.8.8.8) |

> 💡 **R-ISP dan Internet-Server cuma untuk simulasi.** Di dunia nyata, ISP punya ribuan router — kita cuma butuh 1 untuk simulasi bahwa "internet ada di sana".

---

## ⚙️ Konfigurasi

### Urutan Konfigurasi

```
Step 1: Tambah R-ISP dan Internet-Server ke topologi
Step 2: Konfigurasi R-ISP (simulasi ISP)
Step 3: Konfigurasi WAN interface di R-CORE
Step 4: Konfigurasi NAT di R-CORE
Step 5: Default route di R-CORE ke ISP
Step 6: Verifikasi & Testing
```

> ⚠️ **Sebelum mulai:** Pastikan internal network (VLAN, DHCP, inter-VLAN routing) masih jalan dari lab sebelumnya. Test ping antar VLAN dulu.

---

### Step 1 — R-ISP Configuration

```cisco
enable
configure terminal
hostname R-ISP

! ============================================
! Interface ke R-CORE (WAN link)
! ============================================
interface Serial0/0/0
 ip address 200.0.0.1 255.255.255.252
 clock rate 64000
 no shutdown
exit

! ============================================
! Interface ke Internet (simulasi)
! ============================================
interface GigabitEthernet0/1
 ip address 8.8.8.1 255.255.255.0
 no shutdown
exit

! ============================================
! Route balik ke network internal via R-CORE
! ISP harus tahu: "200.0.0.10 ada di R-CORE"
! ============================================
ip route 200.0.0.10 255.255.255.255 200.0.0.2

end
```

> 💡 **Kenapa R-ISP butuh route ke 200.0.0.10?** Karena Static NAT kita map SRV-01 ke 200.0.0.10. ISP harus tahu IP itu reachable via R-CORE. Di production, ISP yang config ini — kamu cuma request.

### Internet-Server

| Field | Value |
|-------|-------|
| IP Address | 8.8.8.8 |
| Subnet Mask | 255.255.255.0 |
| Default Gateway | 8.8.8.1 |

---

### Step 2 — R-CORE WAN Interface

Tambah WAN interface di R-CORE. Sebelumnya R-CORE cuma punya LAN interface.

> ⚠️ **Cek dulu:** R-CORE butuh Serial port. Kalau belum ada module HWIC-2T, pasang dulu (matikan router → drag module → nyalakan). Atau pakai GigabitEthernet kalau Serial sudah terpakai.

```cisco
enable
configure terminal

! ============================================
! WAN Interface ke ISP
! ============================================
interface Serial0/0/0
 ip address 200.0.0.2 255.255.255.252
 no shutdown
exit

! ============================================
! Default Route ke ISP
! ============================================
! "Semua traffic yang gak dikenal, kirim ke ISP"
ip route 0.0.0.0 0.0.0.0 200.0.0.1

end
```

### Penjelasan Default Route

```cisco
ip route 0.0.0.0 0.0.0.0 200.0.0.1
│        │                │
│        │                └── next-hop (IP ISP)
│        └── 0.0.0.0/0 = "semua destination yang gak ada di routing table"
└── static route

Artinya: traffic ke internet (8.8.8.8, google.com, dll) → kirim ke ISP
```

---

### Step 3 — NAT Configuration di R-CORE

#### Tentukan Inside dan Outside Interface

```cisco
configure terminal

! Semua sub-interface LAN = inside
interface GigabitEthernet0/0.10
 ip nat inside
exit

interface GigabitEthernet0/1.20
 ip nat inside
exit

interface GigabitEthernet0/1.30
 ip nat inside
exit

interface GigabitEthernet0/1.40
 ip nat inside
exit

interface GigabitEthernet0/2.50
 ip nat inside
exit

! WAN interface = outside
interface Serial0/0/0
 ip nat outside
exit
```

#### Static NAT — Server ke Public IP

```cisco
! SRV-01 (192.168.10.114) ←→ 200.0.0.10
ip nat inside source static 192.168.10.114 200.0.0.10
```

Penjelasan:

```cisco
ip nat inside source static 192.168.10.114 200.0.0.10
│                      │     │               │
│                      │     │               └── public IP (inside global)
│                      │     └── private IP SRV-01 (inside local)
│                      └── permanent 1:1 mapping
└── NAT command
```

#### PAT (Overload) — Semua PC ke 1 Public IP

```cisco
! ACL untuk definisi network internal yang boleh di-NAT
access-list 1 permit 192.168.10.0 0.0.0.255

! PAT: semua yang match ACL 1 → pakai IP WAN interface (200.0.0.2)
ip nat inside source list 1 interface Serial0/0/0 overload

end
```

Penjelasan:

```cisco
access-list 1 permit 192.168.10.0 0.0.0.255
│           │         │             │
│           │         │             └── wildcard: semua 192.168.10.x
│           │         └── network internal
│           └── ACL number 1 (standard)
└── definisi: "siapa yang boleh di-NAT"

ip nat inside source list 1 interface Serial0/0/0 overload
│                       │   │                      │
│                       │   │                      └── PAT (many:1)
│                       │   └── pakai IP interface Se0/0/0 (200.0.0.2)
│                       └── traffic yang match ACL 1
└── NAT command
```

> 💡 **Kenapa pakai interface, bukan IP?** Kalau ISP ganti IP kamu, NAT otomatis ikut berubah — gak perlu edit config.

---

### Full Config Summary R-CORE (NAT only)

```cisco
configure terminal

! Inside interfaces
interface GigabitEthernet0/0.10
 ip nat inside
interface GigabitEthernet0/1.20
 ip nat inside
interface GigabitEthernet0/1.30
 ip nat inside
interface GigabitEthernet0/1.40
 ip nat inside
interface GigabitEthernet0/2.50
 ip nat inside
exit

! Outside interface
interface Serial0/0/0
 ip nat outside
exit

! Static NAT — Server
ip nat inside source static 192.168.10.114 200.0.0.10

! PAT — Semua PC
access-list 1 permit 192.168.10.0 0.0.0.255
ip nat inside source list 1 interface Serial0/0/0 overload

! Default route ke ISP
ip route 0.0.0.0 0.0.0.0 200.0.0.1

end
```

---

## ✅ Verifikasi & Testing

### 1. Cek Interface NAT Inside/Outside

```cisco
show ip interface Serial0/0/0 | include NAT
show ip interface GigabitEthernet0/0.10 | include NAT
```
Jika ingin lebih detail
```
show running-config | section interface
```

Atau cek semua sekaligus:

```cisco
show running-config | include ip nat
```

Output:

```
 ip nat inside
 ip nat inside
 ip nat inside
 ip nat inside
 ip nat inside
 ip nat outside
ip nat inside source static 192.168.10.114 200.0.0.10
ip nat inside source list 1 interface Serial0/0/0 overload
```

### 2. Cek NAT Table

```cisco
show ip nat translations
```

Sebelum ada traffic → mungkin cuma Static NAT:

```
Pro  Inside global    Inside local      Outside local     Outside global
---  200.0.0.10       192.168.10.114    ---               ---
```

Setelah PC ping internet:

```
Pro  Inside global    Inside local      Outside local     Outside global
---  200.0.0.10       192.168.10.114    ---               ---
icmp 200.0.0.2:1025   192.168.10.11:1   8.8.8.8:1         8.8.8.8:1
icmp 200.0.0.2:1026   192.168.10.75:1   8.8.8.8:1         8.8.8.8:1
```

> 💡 **Ini inti dari NAT.** Lihat kolom Inside local (IP private) dan Inside global (IP public). Router "ingat" mapping ini supaya reply bisa dikirim balik ke device yang benar.

### 3. Cek NAT Statistics

```cisco
show ip nat statistics
```

Menampilkan jumlah translations, hits, dan misses.

### 4. Test PAT — PC Akses Internet

```
# Dari PC-Eng:
ping 8.8.8.8            ← simulasi ping ke internet

# Dari PC-Mkt:
ping 8.8.8.8

# Dari PC-Fin:
ping 8.8.8.8

# Dari PC-Mgmt:
ping 8.8.8.8
```

Semua harus **reply**. Cek NAT table setelah ping — akan muncul entry per PC.

### 5. Test Static NAT — Internet Akses Server

```
# Dari Internet-Server (8.8.8.8):
ping 200.0.0.10          ← ini public IP SRV-01
```

Harus **reply**. Artinya internet bisa reach SRV-01 via public IP.

### 6. Test Dari Internet — Pastikan Private IP Gak Reachable

```
# Dari Internet-Server:
ping 192.168.10.114      ← private IP SRV-01
```

Harus **timeout** ❌. Private IP gak boleh reachable dari internet — itulah tujuan NAT.

### Test Matrix

| Dari | Ke | Via | Seharusnya |
|------|----|-----|------------|
| PC-Eng | 8.8.8.8 | PAT (200.0.0.2) | ✅ Reply |
| PC-Mkt | 8.8.8.8 | PAT (200.0.0.2) | ✅ Reply |
| PC-Fin | 8.8.8.8 | PAT (200.0.0.2) | ✅ Reply |
| PC-Mgmt | 8.8.8.8 | PAT (200.0.0.2) | ✅ Reply |
| Internet-Server | 200.0.0.10 | Static NAT | ✅ Reply (reach SRV-01) |
| Internet-Server | 192.168.10.114 | - | ❌ Timeout (private IP) |

---

## 🔧 Troubleshooting Guide

### Master Flowchart

```
PC gak bisa ping internet (8.8.8.8)?
│
├── Step 1: Ping gateway internal
│   └── ping 192.168.10.1 (atau gateway VLAN masing-masing)
│       ├── ❌ Gagal → masalah internal (VLAN, trunk, DHCP)
│       └── ✅ Reply → lanjut
│
├── Step 2: Dari R-CORE, ping ISP
│   └── ping 200.0.0.1
│       ├── ❌ Gagal → WAN link bermasalah (kabel, IP, interface down)
│       └── ✅ Reply → lanjut
│
├── Step 3: Dari R-CORE, ping internet
│   └── ping 8.8.8.8
│       ├── ❌ Gagal → default route belum ada / salah
│       │   └── show ip route → cek ada "S* 0.0.0.0/0" ?
│       └── ✅ Reply → lanjut
│
├── Step 4: Dari PC, ping internet
│   └── ping 8.8.8.8
│       ├── ❌ Gagal → NAT bermasalah
│       │   ├── show ip nat translations → kosong? NAT gak jalan
│       │   ├── Cek inside/outside interface
│       │   ├── Cek ACL untuk NAT
│       │   └── Cek overload keyword
│       └── ✅ Reply → NAT jalan ✅
│
└── Step 5: Internet gak bisa akses server (200.0.0.10)?
    ├── Cek static NAT: show ip nat translations
    ├── Cek route di R-ISP ke 200.0.0.10
    └── Cek SRV-01 gateway benar (.113)
```

### NAT Translation Table Kosong

```
show ip nat translations → kosong setelah ping?
│
├── Cek inside/outside sudah di-set
│   └── show running-config | include ip nat
│       ├── "ip nat inside" ada di LAN interface?
│       ├── "ip nat outside" ada di WAN interface?
│       └── Kalau gak ada → interface belum di-tag
│
├── Cek ACL untuk PAT
│   └── show access-lists 1
│       └── Harus ada: permit 192.168.10.0 0.0.0.255
│           └── Kosong atau salah → PAT gak tahu siapa yang di-NAT
│
├── Cek NAT command
│   └── show running-config | include ip nat inside source
│       ├── Ada "overload"? Kalau tidak → Dynamic NAT, bukan PAT
│       └── ACL number cocok? (list 1 = access-list 1)
│
└── Cek interface outside benar
    └── "interface Serial0/0/0" di NAT command = interface WAN?
```

### Static NAT Gak Jalan

```
Internet gak bisa reach 200.0.0.10?
│
├── Cek R-ISP punya route ke 200.0.0.10
│   └── Di R-ISP: show ip route
│       └── Harus ada route ke 200.0.0.10 via 200.0.0.2
│
├── Cek static NAT entry
│   └── show ip nat translations
│       └── Harus ada: 200.0.0.10 ↔ 192.168.10.114
│
├── Cek SRV-01 gateway
│   └── Gateway harus 192.168.10.113 (bukan .1 atau kosong)
│
└── Cek inside/outside tag
    └── Interface server (inside) dan WAN (outside) sudah di-tag?
```

### Kesalahan Paling Umum

| Masalah | Penyebab | Solusi |
|---------|----------|--------|
| **PC gak bisa internet** | Lupa `ip nat inside` di sub-interface | Tambah `ip nat inside` di semua LAN sub-interface |
| **PC gak bisa internet** | Lupa `ip nat outside` di WAN | Tambah `ip nat outside` di WAN interface |
| **PC gak bisa internet** | Lupa `overload` di PAT command | Tambah keyword `overload` |
| **PC gak bisa internet** | ACL PAT salah / gak match | Cek wildcard mask di access-list |
| **PC gak bisa internet** | Lupa default route | `ip route 0.0.0.0 0.0.0.0 200.0.0.1` |
| **NAT table kosong** | Inside/outside belum di-tag | Tag setiap interface dengan `ip nat inside/outside` |
| **Internet gak bisa reach server** | R-ISP gak punya route ke public IP | Tambah static route di R-ISP |
| **Internet gak bisa reach server** | SRV-01 gateway salah | Fix gateway ke 192.168.10.113 |
| **Beberapa VLAN gak bisa internet** | Sub-interface VLAN belum `ip nat inside` | Cek semua sub-interface |
| **NAT table penuh** | Terlalu banyak connections | `clear ip nat translation *` |

### ⚠️ Urutan Inside/Outside Matters

```cisco
! SALAH — kebalik
interface GigabitEthernet0/0.10
 ip nat outside              ← harusnya inside!
interface Serial0/0/0
 ip nat inside               ← harusnya outside!

! BENAR
interface GigabitEthernet0/0.10
 ip nat inside               ← LAN = inside
interface Serial0/0/0
 ip nat outside              ← WAN = outside
```

Aturan simpel: **LAN = inside, WAN = outside.** Selalu.

### ⚠️ ACL untuk NAT vs ACL untuk Security

```
ACL 1 (standard) → untuk NAT: "siapa yang boleh di-NAT"
ACL 110+ (extended) → untuk security: "siapa boleh akses apa"

Ini TERPISAH dan gak saling ganggu.
Jangan campur — ACL 1 hanya untuk NAT.
```

### Cara Clear NAT Table

```cisco
! Hapus semua dynamic translations (debug/testing)
clear ip nat translation *

! Hapus specific entry
clear ip nat translation inside 192.168.10.11 outside 8.8.8.8
```

### Command Cheat Sheet

| Command | Fungsi |
|---------|--------|
| `show ip nat translations` | Lihat semua NAT mapping aktif |
| `show ip nat statistics` | Statistik NAT (hits, misses, active) |
| `show running-config \| include ip nat` | Cek config NAT |
| `show access-lists 1` | Cek ACL yang dipakai untuk PAT |
| `show ip route` | Cek default route ada |
| `clear ip nat translation *` | Reset NAT table |
| `debug ip nat` | Live debug NAT (hati-hati di production!) |

---

## 🏭 Production Best Practices

1. **PAT (overload) adalah default** — 99% production pakai PAT. Static NAT hanya untuk server yang harus diakses dari luar.

2. **Jangan expose server tanpa perlu** — Static NAT membuka server ke internet. Pastikan ada **firewall + ACL** di depannya.

3. **Monitor NAT table** — di production, NAT table bisa penuh kalau terlalu banyak connections. Monitor dan alert kalau mendekati limit.

4. **Log NAT translations** — untuk audit dan troubleshooting security incident, log siapa (private IP) akses ke mana (public destination) kapan.

5. **NAT + ACL = defense in depth:**

```
Traffic flow di production:
  PC → ACL (filter) → NAT (translate) → Firewall → Internet
  
  ACL: "Siapa boleh keluar?"
  NAT: "Ubah IP private ke public"
  Firewall: "Inspect content, block malware"
```

6. **Gunakan `interface` bukan IP di PAT command** — kalau ISP ganti IP, NAT otomatis ikut:

```cisco
! BAIK — auto-update kalau IP berubah
ip nat inside source list 1 interface Se0/0/0 overload

! KURANG BAIK — harus manual update kalau IP berubah
ip nat inside source list 1 pool MY-POOL overload
```

7. **Port forwarding untuk specific services** — di production, daripada Static NAT (expose semua port), lebih aman pakai port forwarding:

```cisco
! Cuma forward port 80 (HTTP) ke server
ip nat inside source static tcp 192.168.10.114 80 200.0.0.10 80
```

Ini lebih aman karena cuma port 80 yang terbuka, bukan semua port.

8. **NAT hairpinning** — kalau internal user akses server via public IP (200.0.0.10 dari dalam), butuh NAT hairpin config. Lebih baik internal user akses via **private IP langsung** dan gunakan internal DNS.

---

## 📝 Catatan untuk README di GitHub

- Screenshot topologi lengkap (termasuk R-ISP dan Internet-Server)
- <img width="1140" height="770" alt="image" src="https://github.com/user-attachments/assets/27db3351-e67a-4e05-a872-0a42c41ef096" />

- Screenshot `show ip nat translations` (setelah ping dari beberapa PC)
- 
- Screenshot ping dari PC ke 8.8.8.8 (internet)
- <img width="853" height="525" alt="image" src="https://github.com/user-attachments/assets/ca5f67e6-6bba-45f6-9de7-3d8fb8418a32" />

- Screenshot ping dari Internet-Server ke 200.0.0.10 (static NAT)
  <img width="589" height="317" alt="image" src="https://github.com/user-attachments/assets/216019a8-afc3-4a0e-930e-b77e661bf6a9" />

- Screenshot ping gagal dari Internet-Server ke 192.168.10.114 (private IP unreachable)
- File `.pkt`

---

## 🔗 Hubungan NAT dengan Lab Sebelumnya

```
Lab 004-006: VLAN + DHCP    → PC punya IP private
Lab 007-008: Routing         → Antar subnet bisa komunikasi
Lab 009:     ACL             → Filter siapa boleh akses apa
Lab 010:     NAT/PAT         → Private IP bisa ke internet ← SEKARANG
```

Tanpa NAT, semua lab sebelumnya hanya bisa komunikasi **internal**. Dengan NAT, network kamu sekarang terhubung ke "dunia luar".

---

## 🔄 Cara NAT Bekerja Step-by-Step

```
PC-Eng (192.168.10.11) ping 8.8.8.8:

1. PC kirim paket:
   Source: 192.168.10.11 → Dest: 8.8.8.8

2. Paket sampai di R-CORE (inside interface):
   Router cek: "Ada NAT? Ya, PAT overload"
   Ubah source: 192.168.10.11:1025 → 200.0.0.2:1025
   Simpan di NAT table

3. Paket keluar dari R-CORE (outside interface):
   Source: 200.0.0.2:1025 → Dest: 8.8.8.8

4. Sampai di Internet-Server (8.8.8.8):
   Reply: Source: 8.8.8.8 → Dest: 200.0.0.2:1025

5. Reply sampai di R-CORE (outside interface):
   Router cek NAT table: "200.0.0.2:1025 = 192.168.10.11:1025"
   Ubah dest: 200.0.0.2:1025 → 192.168.10.11:1025

6. Reply sampai di PC-Eng:
   Source: 8.8.8.8 → Dest: 192.168.10.11 ✅
```

---

## ⏭️ Lab Selanjutnya

→ **011_STP_EtherChannel** — Spanning Tree Protocol dan Link Aggregation: mencegah loop dan meningkatkan bandwidth di layer 2.
