# 006 — DHCP Dedicated Server & IP Helper-Address

## 🎯 Tujuan Lab

Memindahkan DHCP service dari router ke **dedicated server**, lalu menggunakan `ip helper-address` di router untuk relay DHCP request. Ini adalah cara DHCP yang sebenarnya dipakai di production — router hanya routing, server yang handle DHCP.

---

## 📖 Konsep yang Dipelajari

| Konsep | Deskripsi |
|--------|-----------|
| **Dedicated DHCP Server** | Server khusus yang menjalankan DHCP service |
| **ip helper-address** | Command di router untuk forward broadcast DHCP ke server |
| **DHCP Relay** | Proses router meneruskan DHCP request lintas VLAN/subnet |
| **Unicast vs Broadcast** | Helper-address mengubah broadcast jadi unicast ke server |

### Kenapa Pindah dari Router ke Server?

| | DHCP di Router (Lab 005) | DHCP di Server (Lab 006) |
|---|---|---|
| **Lease time** | Terbatas (Packet Tracer gak support) | Fully configurable |
| **Beban router** | Routing + DHCP = berat | Routing only = ringan |
| **Scalability** | Ratusan client | Ribuan client |
| **Kalau restart** | DHCP + routing mati | Routing tetap jalan |
| **Monitoring** | Minimal | GUI, logging detail |

### Bagaimana ip helper-address Bekerja

```
TANPA helper-address:
  PC (VLAN 20) ──broadcast──→ ❌ STUCK di VLAN 20
                                  Router gak forward broadcast
                                  Server (VLAN 50) gak terima

DENGAN helper-address:
  PC (VLAN 20) ──broadcast──→ Router (Gi0/1.20)
                                  │ helper-address 192.168.10.114
                                  │ convert broadcast → unicast
                                  ↓
                              Server (VLAN 50) ──→ kasih IP ✅
```

---

## 🏢 Skenario

Melanjutkan dari **Lab 005**. IT Manager minta DHCP dipindah ke dedicated server (SRV-01) karena:
- Router R-CORE sudah terlalu banyak beban
- Butuh monitoring dan kontrol lebih baik
- Best practice: 1 device 1 fungsi utama

**Yang berubah:**
- DHCP pool di router → **dihapus**
- DHCP service di SRV-01 → **diaktifkan**
- Router → tambah `ip helper-address` di setiap sub-interface

---

## 📋 DHCP Pool Planning (Sama dengan Lab 005)

| VLAN | Subnet | Pool Start | Pool End | Gateway | DNS |
|------|--------|-----------|----------|---------|-----|
| 10 - Engineering | 192.168.10.0/26 | .11 | .62 | 192.168.10.1 | 8.8.8.8 |
| 20 - Marketing | 192.168.10.64/27 | .75 | .94 | 192.168.10.65 | 8.8.8.8 |
| 30 - Finance | 192.168.10.96/28 | .101 | .110 | 192.168.10.97 | 8.8.8.8 |
| 40 - Management | 192.168.10.128/29 | .132 | .134 | 192.168.10.129 | 8.8.8.8 |

> SRV-01 tetap static IP: `192.168.10.114`

---

## 🖥️ Topologi

Sama dengan Lab 005 — tidak ada perubahan fisik.

```
                        [R-CORE]
                      DHCP Relay
                  Gi0/0  Gi0/1   Gi0/2
                    │      │       │
                trunk  trunk   trunk
                    │      │       │
               [SW-ENG] [SW-OFFICE] [SW-SERVER]
                  │       │  │  │       │
               PC-Eng  PC-Mkt │ PC-Mgmt SRV-01
              (DHCP)  (DHCP)  │ (DHCP)  (DHCP Server)
                       PC-Fin            Static IP
                      (DHCP)
```

### Alur DHCP Request

```
1. PC-Mkt broadcast DHCP Discover
          ↓
2. SW-OFFICE trunk ke R-CORE
          ↓
3. R-CORE (Gi0/1.20) terima broadcast
   ip helper-address 192.168.10.114
   → convert ke unicast, kirim ke SRV-01
          ↓
4. SRV-01 terima request, cek pool Marketing
   → kirim DHCP Offer (IP .75) balik via R-CORE
          ↓
5. PC-Mkt dapat IP 192.168.10.75 ✅
```

---

## ⚙️ Konfigurasi

### Urutan Konfigurasi

```
Step 1: Hapus DHCP pool di router (dari Lab 005)
Step 2: Konfigurasi DHCP service di SRV-01
Step 3: Tambah ip helper-address di router sub-interface
Step 4: Set PC ke DHCP
Step 5: Verifikasi
```

---

### Step 1 — Hapus DHCP di Router

```cisco
enable
configure terminal

! Hapus semua pool
no ip dhcp pool VLAN10-ENGINEERING
no ip dhcp pool VLAN20-MARKETING
no ip dhcp pool VLAN30-FINANCE
no ip dhcp pool VLAN40-MANAGEMENT

! Hapus excluded address (server yang handle sekarang)
no ip dhcp excluded-address 192.168.10.1 192.168.10.10
no ip dhcp excluded-address 192.168.10.65 192.168.10.74
no ip dhcp excluded-address 192.168.10.97 192.168.10.100
no ip dhcp excluded-address 192.168.10.129 192.168.10.131

end
```

Verifikasi sudah bersih:

```cisco
show running-config | include dhcp
```

Harusnya kosong (tidak ada output DHCP pool).

---

### Step 2 — Konfigurasi DHCP di SRV-01

Klik **SRV-01** → tab **Services** → klik **DHCP** di sidebar kiri.

**Turn On** DHCP service, lalu buat pool untuk setiap VLAN:

#### Pool 1 — Engineering (VLAN 10)

| Field | Value |
|-------|-------|
| Pool Name | ENGINEERING |
| Default Gateway | 192.168.10.1 |
| DNS Server | 8.8.8.8 |
| Start IP Address | 192.168.10.11 |
| Subnet Mask | 255.255.255.192 |
| Max Users | 52 |

Klik **Add** setelah mengisi.

#### Pool 2 — Marketing (VLAN 20)

| Field | Value |
|-------|-------|
| Pool Name | MARKETING |
| Default Gateway | 192.168.10.65 |
| DNS Server | 8.8.8.8 |
| Start IP Address | 192.168.10.75 |
| Subnet Mask | 255.255.255.224 |
| Max Users | 20 |

Klik **Add**.

#### Pool 3 — Finance (VLAN 30)

| Field | Value |
|-------|-------|
| Pool Name | FINANCE |
| Default Gateway | 192.168.10.97 |
| DNS Server | 8.8.8.8 |
| Start IP Address | 192.168.10.101 |
| Subnet Mask | 255.255.255.240 |
| Max Users | 10 |

Klik **Add**.

#### Pool 4 — Management (VLAN 40)

| Field | Value |
|-------|-------|
| Pool Name | MANAGEMENT |
| Default Gateway | 192.168.10.129 |
| DNS Server | 8.8.8.8 |
| Start IP Address | 192.168.10.132 |
| Subnet Mask | 255.255.255.248 |
| Max Users | 3 |

Klik **Add**.

> ⚠️ **Jangan lupa klik Add untuk setiap pool!** Kalau langsung pindah tanpa Add, pool tidak tersimpan.

> 💡 **Pool default "serverPool" bisa dihapus** — itu template bawaan, tidak dipakai.

---

### Step 3 — IP Helper-Address di Router

```cisco
enable
configure terminal

! VLAN 10 — Engineering
interface GigabitEthernet0/0.10
 ip helper-address 192.168.10.114
exit

! VLAN 20 — Marketing
interface GigabitEthernet0/1.20
 ip helper-address 192.168.10.114
exit

! VLAN 30 — Finance
interface GigabitEthernet0/1.30
 ip helper-address 192.168.10.114
exit

! VLAN 40 — Management
interface GigabitEthernet0/1.40
 ip helper-address 192.168.10.114
exit

end
```

> 💡 **VLAN 50 (Server) tidak butuh helper** — SRV-01 sendiri ada di VLAN 50 dan pakai static IP.

Verifikasi:

```cisco
show running-config | include helper
```

Output:

```
 ip helper-address 192.168.10.114
 ip helper-address 192.168.10.114
 ip helper-address 192.168.10.114
 ip helper-address 192.168.10.114
```

---

### Step 4 — Set PC ke DHCP

Klik setiap PC → Desktop → IP Configuration → pilih **DHCP**.

Kalau sebelumnya sudah DHCP (dari Lab 005), switch ke **Static** dulu, lalu balik ke **DHCP** untuk force renew dari server baru.

| Device | Mode | Seharusnya Dapat IP |
|--------|------|-------------------|
| PC-Eng | DHCP | 192.168.10.11 - .62 |
| PC-Mkt | DHCP | 192.168.10.75 - .94 |
| PC-Fin | DHCP | 192.168.10.101 - .110 |
| PC-Mgmt | DHCP | 192.168.10.132 - .134 |
| SRV-01 | **Static** | 192.168.10.114 |

---

## ✅ Verifikasi & Testing

### 1. Cek IP di PC

```
ipconfig
```

Pastikan:
- IP Address → dari range pool yang sesuai
- Subnet Mask → sesuai VLSM
- Default Gateway → sesuai tabel
- DNS Server → 8.8.8.8

### 2. Cek DHCP di Router (Sudah Bersih)

```cisco
show ip dhcp binding
```

Harusnya **kosong** — karena DHCP sudah bukan di router.

```cisco
show ip dhcp pool
```

Harusnya **kosong** juga.

### 3. Cek Helper-Address Aktif

```cisco
show running-config | include helper
```

Harus muncul 4 baris `ip helper-address 192.168.10.114`.

### 4. Ping Test

```
# Dari PC-Eng:
ping 192.168.10.1      ← gateway
ping [PC-Mkt IP]       ← Marketing
ping [PC-Fin IP]       ← Finance
ping 192.168.10.114    ← Server (DHCP source)
```

### 5. Test DHCP Dari Server Bekerja

Cara paling gampang: tambah **1 PC baru** di salah satu switch, colok ke port VLAN yang benar, set ke DHCP. Kalau dapat IP → DHCP server berjalan.

---

## 🔧 Troubleshooting Guide

### PC Gak Dapat IP (0.0.0.0)

```
PC dapat 0.0.0.0?
│
├── Cek DHCP service di SRV-01
│   ├── Services → DHCP → Service harus ON
│   └── Pool sudah di-Add? (bukan cuma diisi tapi lupa Add)
│
├── Cek pool di server
│   ├── Start IP, gateway, mask harus sesuai VLAN
│   └── Gateway harus cocok dengan sub-interface router
│
├── Cek ip helper-address di router
│   └── show running-config | include helper
│       └── Harus ada di setiap sub-interface
│
├── Cek koneksi ke server
│   └── Dari router: ping 192.168.10.114
│       ├── ❌ Gagal → masalah VLAN 50 / trunk SW-SERVER
│       └── ✅ Reply → lanjut cek di bawah
│
├── Cek trunk semua switch
│   └── show interfaces trunk
│
└── Cek DHCP pool router sudah dihapus
    └── show ip dhcp pool (harus kosong)
        └── Kalau masih ada → router dan server conflict
```

### Kesalahan Umum

| Masalah | Penyebab | Solusi |
|---------|----------|--------|
| Pool lupa di-Add di server | Klik Add setelah isi setiap pool | Buka Services → DHCP → isi ulang → Add |
| Gateway salah di pool server | Gateway pool ≠ sub-interface router | Sesuaikan gateway di pool server |
| Subnet mask salah di pool | Mask gak cocok dengan VLSM | Cek tabel VLSM |
| helper-address salah IP | Typo IP server | `show run \| include helper` lalu fix |
| DHCP pool masih ada di router | Belum dihapus dari Lab 005 | `no ip dhcp pool [name]` |
| Router gak bisa reach server | VLAN 50 / trunk bermasalah | Cek trunk SW-SERVER + sub-interface Gi0/2.50 |
| PC masih pakai IP lama | Cache dari Lab 005 | Switch ke Static lalu balik ke DHCP |

### Command Cheat Sheet

| Command | Dimana | Fungsi |
|---------|--------|--------|
| `show running-config \| include helper` | Router | Cek helper-address per interface |
| `show running-config \| section interface` | Router | Cek helper-address per interface |
| `show running-config \| section GigabitEthernet0/1` | Router | Cek helper-address per interface Detail |
| `show ip dhcp binding` | Router | Harus kosong (DHCP bukan di router) |
| `show ip dhcp pool` | Router | Harus kosong |
| `show interfaces trunk` | Switch | Cek trunk aktif |
| `ipconfig` | PC | Cek IP yang didapat |
| `ipconfig /release` lalu `ipconfig /renew` | PC | Force renew DHCP |

---

## 🏭 Production Best Practices

1. **DHCP selalu di dedicated server** — router fokus routing, server fokus DHCP. Separation of concerns.

2. **ip helper-address wajib di setiap VLAN interface** — kalau lupa 1, VLAN tersebut gak bisa dapat DHCP.

3. **DHCP server harus static IP** — kalau server sendiri pakai DHCP, siapa yang kasih IP ke server? Chicken-and-egg problem.

4. **Redundancy** — di production besar, pakai 2 DHCP server dengan pool yang di-split (misal server A handle .11-.36, server B handle .37-.62) agar kalau 1 mati, yang lain masih serve.

5. **Monitoring** — di production, monitor DHCP pool usage. Kalau pool hampir penuh, alarm harus berbunyi sebelum user complain "gak bisa konek wifi".

6. **DHCP Snooping** (lab 008 nanti) — prevent rogue DHCP server. Tanpa ini, siapa saja bisa colok DHCP server palsu dan hijack network.

7. **Logging** — di production, DHCP server harusnya log siapa (MAC) dapat IP berapa dan kapan. Ini penting untuk audit dan security incident response.

---

## 📝 Catatan untuk README di GitHub

- Screenshot DHCP service di SRV-01 (Services → DHCP)
- <img width="1466" height="1298" alt="image" src="https://github.com/user-attachments/assets/d75a6bdb-ed9e-454c-a326-716f61d25552" />

- Screenshot `show running-config | include helper` dari router
  <img width="561" height="140" alt="image" src="https://github.com/user-attachments/assets/21ca66c8-cd5f-4606-b583-e5c379e1d823" />

- File `.pkt` (Packet Tracer save file)

---

## 🔄 Perbandingan Lab 005 vs Lab 006

| | Lab 005 (DHCP di Router) | Lab 006 (DHCP di Server) |
|---|---|---|
| DHCP service | R-CORE | SRV-01 |
| Router config | Pool + excluded | ip helper-address |
| Scalability | Kecil | Besar |
| Production-ready | ❌ | ✅ |
| Complexity | Simpel | Sedikit lebih complex |

---

## ⏭️ Lab Selanjutnya

→ **007_Static_Routing_Multi_Path** — Menghubungkan 2 network berbeda menggunakan static route, termasuk failover path untuk redundancy.
