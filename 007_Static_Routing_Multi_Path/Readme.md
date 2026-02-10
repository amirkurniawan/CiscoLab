# 007 — Static Routing Multi-Path & Failover

## 🎯 Tujuan Lab

Menghubungkan 2 kantor (Jakarta & Surabaya) menggunakan static route melalui 2 jalur berbeda — primary path dan backup path. Kalau jalur utama mati, traffic otomatis pindah ke jalur backup (failover).

---

## 📖 Konsep yang Dipelajari

| Konsep | Deskripsi |
|--------|-----------|
| **Static Route** | Route yang ditentukan manual oleh admin, bukan otomatis |
| **Administrative Distance (AD)** | Angka prioritas route — makin kecil makin diprioritaskan |
| **Floating Static Route** | Static route backup dengan AD lebih tinggi (aktif kalau primary mati) |
| **Default Route** | Route "catch-all" untuk semua tujuan yang tidak dikenal (`0.0.0.0/0`) |
| **Point-to-Point Link** | Koneksi langsung antar 2 router, biasanya /30 |
| **Failover** | Mekanisme auto-switch ke jalur backup saat jalur utama down |

### Static Route vs Dynamic Route

| | Static Route | Dynamic Route (OSPF, dll) |
|---|---|---|
| **Dikonfigurasi** | Manual oleh admin | Otomatis antar router |
| **Cocok untuk** | Network kecil, specific path | Network besar, complex topology |
| **Failover** | Manual (floating static) | Otomatis |
| **Resource** | Ringan | Butuh CPU/memory |
| **Di production** | Dipakai untuk specific route, default route | Dipakai untuk internal routing |

### Bagaimana Failover Bekerja

```
Normal: Primary path aktif (AD = 1)
  Jakarta ════ ISP-A ════ Surabaya     ← traffic lewat sini
  Jakarta ──── ISP-B ──── Surabaya     ← standby (AD = 10)

Primary down: Backup aktif
  Jakarta ═╳═╳ ISP-A ═╳═╳ Surabaya    ← mati
  Jakarta ════ ISP-B ════ Surabaya     ← traffic pindah ke sini
```

Router pilih route dengan **AD terkecil**. Floating static route sengaja di-set AD lebih besar — jadi cuma aktif kalau primary hilang dari routing table.

---

## 🏢 Skenario

PT. Nusantara Digital buka kantor cabang di Surabaya. Kedua kantor dihubungkan lewat 2 jalur WAN:

- **Jalur utama (Primary):** Lewat ISP-A — lebih cepat, lebih mahal
- **Jalur backup (Backup):** Lewat ISP-B — lebih lambat, lebih murah

Kalau ISP-A down, traffic harus **otomatis pindah** ke ISP-B tanpa perlu config manual.

---

## 📋 IP Addressing Table

### LAN Segments

| Lokasi | Network | Subnet Mask | Gateway | VLAN |
|--------|---------|-------------|---------|------|
| Jakarta LAN | 192.168.1.0/24 | 255.255.255.0 | 192.168.1.1 | - |
| Surabaya LAN | 192.168.2.0/24 | 255.255.255.0 | 192.168.2.1 | - |

### WAN Links (Point-to-Point /30)

| Link | Network | Router A IP | Router B IP |
|------|---------|-------------|-------------|
| Primary (ISP-A) | 10.0.0.0/30 | 10.0.0.1 (R-JKT) | 10.0.0.2 (R-SBY) |
| Backup (ISP-B) | 10.0.1.0/30 | 10.0.1.1 (R-JKT) | 10.0.1.2 (R-SBY) |

### Device IP Summary

| Device | Interface | IP Address | Subnet Mask | Fungsi |
|--------|-----------|------------|-------------|--------|
| R-JKT | Gi0/0 | 192.168.1.1 | 255.255.255.0 | Gateway Jakarta LAN |
| R-JKT | Se0/0/0 | 10.0.0.1 | 255.255.255.252 | Primary WAN ke R-SBY |
| R-JKT | Se0/0/1 | 10.0.1.1 | 255.255.255.252 | Backup WAN ke R-SBY |
| R-SBY | Gi0/0 | 192.168.2.1 | 255.255.255.0 | Gateway Surabaya LAN |
| R-SBY | Se0/0/0 | 10.0.0.2 | 255.255.255.252 | Primary WAN ke R-JKT |
| R-SBY | Se0/0/1 | 10.0.1.2 | 255.255.255.252 | Backup WAN ke R-JKT |
| PC-JKT | NIC | 192.168.1.10 | 255.255.255.0 | Client Jakarta |
| PC-SBY | NIC | 192.168.2.10 | 255.255.255.0 | Client Surabaya |

> 💡 **WAN link pakai Serial interface** di Packet Tracer karena mensimulasikan koneksi WAN jarak jauh. LAN pakai GigabitEthernet.

> 💡 **WAN link pakai /30** — hanya butuh 2 IP (1 per ujung router), hemat IP. Ini standar production.

---

## 🖥️ Topologi

```
          Jakarta                                    Surabaya
                        Primary (ISP-A)
      [R-JKT] ════════Se0/0/0══════════Se0/0/0════ [R-SBY]
      │  Gi0/0        Se0/0/1══════════Se0/0/1        Gi0/0 │
      │  .1              Backup (ISP-B)                .1    │
      │                                                      │
   [SW-JKT]                                            [SW-SBY]
      │                                                      │
   [PC-JKT]                                            [PC-SBY]
   .10                                                 .10
   192.168.1.0/24                              192.168.2.0/24
```

### Perangkat

| Device | Model | Hostname | Module Tambahan |
|--------|-------|----------|-----------------|
| Router 1 | 2911 | R-JKT | HWIC-2T (untuk Serial port) |
| Router 2 | 2911 | R-SBY | HWIC-2T (untuk Serial port) |
| Switch 1 | 2960 | SW-JKT | - |
| Switch 2 | 2960 | SW-SBY | - |
| PC 1 | - | PC-JKT | - |
| PC 2 | - | PC-SBY | - |

> ⚠️ **Router 2911 default tidak punya Serial port.** Kamu harus tambahkan module **HWIC-2T** di tab Physical → matikan router dulu → drag HWIC-2T ke slot yang kosong → nyalakan lagi.

---

## ⚙️ Konfigurasi

### Urutan Konfigurasi

```
Step 1: Tambah module HWIC-2T di kedua router
Step 2: Hostname semua device
Step 3: Konfigurasi interface LAN (Gi0/0) di kedua router
Step 4: Konfigurasi interface WAN (Serial) di kedua router
Step 5: Konfigurasi static route + floating static route
Step 6: Assign IP ke PC
Step 7: Verifikasi & Testing failover
```

---

### Step 1 — Tambah Module HWIC-2T

Untuk kedua router (R-JKT dan R-SBY):
1. Klik router → tab **Physical**
2. Klik tombol **power** untuk matikan router
3. Di sidebar kiri, cari **HWIC-2T**
4. Drag ke slot kosong di router
5. Klik **power** untuk nyalakan lagi

Sekarang router punya `Se0/0/0` dan `Se0/0/1`.

---

### R-JKT

```cisco
enable
configure terminal
hostname R-JKT

! ============================================
! LAN Interface
! ============================================
interface GigabitEthernet0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
exit

! ============================================
! WAN Primary (ke R-SBY via ISP-A)
! ============================================
interface Serial0/0/0
 ip address 10.0.0.1 255.255.255.252
 clock rate 64000
 no shutdown
exit

! ============================================
! WAN Backup (ke R-SBY via ISP-B)
! ============================================
interface Serial0/0/1
 ip address 10.0.1.1 255.255.255.252
 clock rate 64000
 no shutdown
exit

! ============================================
! Static Route ke Surabaya LAN (192.168.2.0/24)
! ============================================

! Primary route via ISP-A (AD default = 1)
ip route 192.168.2.0 255.255.255.0 10.0.0.2

! Floating static route via ISP-B (AD = 10, backup)
ip route 192.168.2.0 255.255.255.0 10.0.1.2 10

end
```

### R-SBY

```cisco
enable
configure terminal
hostname R-SBY

! ============================================
! LAN Interface
! ============================================
interface GigabitEthernet0/0
 ip address 192.168.2.1 255.255.255.0
 no shutdown
exit

! ============================================
! WAN Primary (ke R-JKT via ISP-A)
! ============================================
interface Serial0/0/0
 ip address 10.0.0.2 255.255.255.252
 no shutdown
exit

! ============================================
! WAN Backup (ke R-JKT via ISP-B)
! ============================================
interface Serial0/0/1
 ip address 10.0.1.2 255.255.255.252
 no shutdown
exit

! ============================================
! Static Route ke Jakarta LAN (192.168.1.0/24)
! ============================================

! Primary route via ISP-A (AD default = 1)
ip route 192.168.1.0 255.255.255.0 10.0.0.1

! Floating static route via ISP-B (AD = 10, backup)
ip route 192.168.1.0 255.255.255.0 10.0.1.1 10

end
```

### Penjelasan Static Route

```cisco
ip route 192.168.2.0 255.255.255.0 10.0.0.2
│        │                          │
│        │                          └── next-hop (IP router lawan)
│        └── destination network
└── command
```

```cisco
ip route 192.168.2.0 255.255.255.0 10.0.1.2 10
│        │                          │        │
│        │                          │        └── AD = 10 (backup)
│        │                          └── next-hop via ISP-B
│        └── destination network
└── command
```

**AD default static route = 1.** Dengan set AD = 10 di backup, route ini cuma masuk routing table kalau primary (AD 1) **hilang**.

### Clock Rate

```cisco
clock rate 64000
```

Di Packet Tracer, Serial link butuh clock rate di salah satu sisi (DCE side). Kalau error "clock rate can only be set on DCE interface", pindahkan command ini ke router satunya.

---

### Switch (Minimal Config)

```cisco
! SW-JKT
enable
configure terminal
hostname SW-JKT
end

! SW-SBY
enable
configure terminal
hostname SW-SBY
end
```

Switch di lab ini tidak butuh VLAN — cuma 1 LAN per lokasi.

---

### End Devices — IP Assignment

| Device | IP Address | Subnet Mask | Default Gateway |
|--------|------------|-------------|-----------------|
| PC-JKT | 192.168.1.10 | 255.255.255.0 | 192.168.1.1 |
| PC-SBY | 192.168.2.10 | 255.255.255.0 | 192.168.2.1 |

---

## ✅ Verifikasi & Testing

### 1. Cek Interface Status

```cisco
show ip interface brief
```

Semua interface harus **up/up**:

```
Interface          IP-Address      OK? Method Status    Protocol
GigabitEthernet0/0 192.168.1.1    YES manual up        up
Serial0/0/0        10.0.0.1       YES manual up        up
Serial0/0/1        10.0.1.1       YES manual up        up
```

### 2. Cek Routing Table

```cisco
show ip route
```

**Saat primary aktif** — hanya primary route yang muncul:

```
S    192.168.2.0/24 [1/0] via 10.0.0.2        ← primary (AD=1)
C    192.168.1.0/24 is directly connected, Gi0/0
C    10.0.0.0/30 is directly connected, Se0/0/0
C    10.0.1.0/30 is directly connected, Se0/0/1
```

> 💡 **`[1/0]` artinya AD=1, metric=0.** Floating static route (AD=10) **tidak muncul** karena primary masih aktif. Route backup "mengambang" di belakang — makanya disebut floating.

### 3. Ping Test (Primary Active)

```
# Dari PC-JKT:
ping 192.168.2.10    ← PC Surabaya (harus reply)

# Dari PC-SBY:
ping 192.168.1.10    ← PC Jakarta (harus reply)
```

### 4. Traceroute (Cek Jalur)

```
# Dari PC-JKT:
tracert 192.168.2.10
```

Output saat primary aktif:

```
1   192.168.1.1      ← R-JKT (gateway)
2   10.0.0.2         ← R-SBY via ISP-A (primary)
3   192.168.2.10     ← PC-SBY
```

### 5. Test Failover ⚡

**Ini bagian paling penting di lab ini!**

#### Step A — Matikan Primary Link

Di R-JKT:

```cisco
configure terminal
interface Serial0/0/0
 shutdown
end
```

#### Step B — Cek Routing Table

```cisco
show ip route
```

Sekarang floating static route **muncul**:

```
S    192.168.2.0/24 [10/0] via 10.0.1.2       ← backup (AD=10) AKTIF!
C    192.168.1.0/24 is directly connected, Gi0/0
C    10.0.1.0/30 is directly connected, Se0/0/1
```

`[10/0]` = AD 10, ini backup route.

#### Step C — Ping Lagi

```
# Dari PC-JKT:
ping 192.168.2.10    ← harus tetap reply (lewat backup)
```

#### Step D — Traceroute Lagi

```
tracert 192.168.2.10
```

Output saat backup aktif:

```
1   192.168.1.1      ← R-JKT
2   10.0.1.2         ← R-SBY via ISP-B (backup!) ← BEDA dari tadi
3   192.168.2.10     ← PC-SBY
```

#### Step E — Nyalakan Lagi Primary

```cisco
configure terminal
interface Serial0/0/0
 no shutdown
end
```

Cek `show ip route` — harus kembali ke primary (AD=1).

---

## 🔧 Troubleshooting Guide

### Ping Antar Kantor Gagal

```
PC-JKT gak bisa ping PC-SBY?
│
├── Cek ping ke gateway dulu
│   └── ping 192.168.1.1
│       ├── ❌ Gagal → masalah LAN (kabel, switch, IP PC)
│       └── ✅ Reply → lanjut
│
├── Cek ping antar router (WAN)
│   └── Dari R-JKT: ping 10.0.0.2
│       ├── ❌ Gagal → Serial link bermasalah
│       │   ├── show ip interface brief → cek up/up
│       │   ├── clock rate sudah di-set?
│       │   └── Kabel Serial sudah nyambung?
│       └── ✅ Reply → lanjut
│
├── Cek routing table
│   └── show ip route
│       ├── Route ke 192.168.2.0 tidak ada → static route belum di-config
│       └── Route ada tapi next-hop salah → cek IP next-hop
│
└── Cek routing di KEDUA router
    └── Route harus di-config di KEDUA sisi (R-JKT DAN R-SBY)
```

### Failover Tidak Bekerja

| Masalah | Penyebab | Solusi |
|---------|----------|--------|
| Backup route tidak muncul setelah primary down | Floating static belum di-config | Tambah route dengan AD > 1 |
| Kedua route muncul bersamaan | AD sama (keduanya = 1) | Set backup AD lebih tinggi: `ip route ... 10` |
| Primary kembali tapi traffic masih lewat backup | Primary interface belum `no shutdown` | Nyalakan lagi primary interface |
| Traceroute menunjukkan jalur yang salah | Route di sisi lawan belum di-config | Config static route di **kedua** router |

### Kesalahan Umum

| Masalah | Penyebab | Solusi |
|---------|----------|--------|
| Serial interface down/down | Module HWIC-2T belum dipasang | Matikan router → pasang module → nyalakan |
| clock rate error | Command diketik di DTE side | Pindahkan clock rate ke router satunya |
| Route cuma 1 arah | Lupa config di router lawan | Static route harus di **KEDUA** router |
| Next-hop unreachable | IP next-hop salah / typo | Cek IP lawan dengan `show ip interface brief` |
| Serial link up tapi gak bisa ping | Subnet mask /30 salah | Harus 255.255.255.252 di kedua sisi |

### Command Cheat Sheet

| Command | Fungsi |
|---------|--------|
| `show ip route` | Lihat routing table (cek primary/backup aktif) |
| `show ip route static` | Lihat hanya static route |
| `show ip interface brief` | Status semua interface |
| `show interfaces Serial0/0/0` | Detail Serial interface (termasuk clock rate) |
| `tracert [ip]` | Lacak jalur dari PC (traceroute) |
| `traceroute [ip]` | Lacak jalur dari router |
| `shutdown` / `no shutdown` | Matikan / nyalakan interface (untuk test failover) |

---

## 🏭 Production Best Practices

1. **Selalu punya backup path** — single point of failure = network mati total saat WAN down. Dual WAN adalah minimum di production.

2. **AD backup harus lebih tinggi dari primary** — kalau sama, traffic di-load balance (bisa jadi masalah kalau bandwidth beda jauh).

3. **Test failover secara berkala** — jangan tunggu sampai beneran down baru tahu backup gak jalan. Schedule failover test bulanan.

4. **Monitor kedua link** — di production, pakai SNMP/monitoring untuk alert kalau salah satu link down. Jangan sampai backup sudah mati duluan tanpa ada yang tahu.

5. **Dokumentasikan jalur** — catat ISP mana primary, mana backup, contact number ISP, dan SLA masing-masing.

6. **Static route cocok untuk:**
   - Default route ke ISP
   - Specific route ke partner/vendor network
   - Backup route (floating static)

7. **Static route TIDAK cocok untuk:**
   - Internal routing di network besar (pakai OSPF)
   - Network yang topologinya sering berubah

8. **Point-to-Point link selalu /30 atau /31** — jangan pakai /24 untuk link antar 2 router, boros 252 IP.

---

## 📝 Catatan untuk README di GitHub

- Screenshot topologi di Packet Tracer
- <img width="982" height="1034" alt="image" src="https://github.com/user-attachments/assets/42847fdd-5958-4da1-9d84-557dcb6b3fc1" />

- Screenshot `show ip route` saat primary aktif
  <img width="776" height="231" alt="image" src="https://github.com/user-attachments/assets/cba71a30-6355-4abc-a041-78af4cd08ad8" />

- Screenshot `show ip route` saat primary down (failover)
  <img width="834" height="408" alt="image" src="https://github.com/user-attachments/assets/1fcd2a13-0ac0-43c5-bb0f-c9d643101824" />

- Screenshot traceroute via primary path
  <img width="579" height="241" alt="image" src="https://github.com/user-attachments/assets/38e5b9eb-7bd6-42c8-aeb6-408e7c796fd5" />
  <img width="633" height="170" alt="image" src="https://github.com/user-attachments/assets/9d669aa6-82a1-4482-a3d6-08c6b35df395" />


- Screenshot traceroute via backup path
  <img width="566" height="129" alt="image" src="https://github.com/user-attachments/assets/8a8b9aaa-11b7-4e95-978b-a1ae511a19cf" />
  <img width="708" height="181" alt="image" src="https://github.com/user-attachments/assets/ebc56acb-bb08-4d46-8fd9-e6f3f6a23fea" />


- File `.pkt` (Packet Tracer save file)

---

## ⏭️ Lab Selanjutnya

→ **008_OSPF_Single_Area** — Dynamic routing OSPF area 0, router otomatis belajar route tanpa perlu config manual satu-satu.
