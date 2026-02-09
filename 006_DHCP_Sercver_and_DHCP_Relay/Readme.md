# 005 — DHCP Server & DHCP Relay

## 🎯 Tujuan Lab

Mengkonfigurasi DHCP Server di router agar semua PC mendapat IP address **otomatis** tanpa harus assign manual satu-satu. Di production, DHCP adalah service wajib — tidak mungkin IT assign IP manual ke 100+ device.

---

## 📖 Konsep yang Dipelajari

| Konsep | Deskripsi |
|--------|-----------|
| **DHCP** | Dynamic Host Configuration Protocol — assign IP otomatis ke device |
| **DHCP Pool** | Kumpulan IP yang tersedia untuk diberikan ke client |
| **Excluded Address** | IP yang di-reserve dan tidak boleh diberikan DHCP ke client |
| **DHCP Relay (ip helper-address)** | Forward DHCP request dari VLAN lain ke DHCP server |
| **Lease Time** | Durasi "pinjam" IP — setelah habis, client harus renew |
| **DORA Process** | Discover → Offer → Request → Acknowledge (cara DHCP bekerja) |

### Bagaimana DHCP Bekerja (DORA)

```
PC (baru nyala)                          DHCP Server (Router)
     │                                        │
     │──── 1. DISCOVER (broadcast) ──────────→│  "Ada DHCP server gak?"
     │                                        │
     │←─── 2. OFFER ─────────────────────────│  "Nih IP 192.168.10.2"
     │                                        │
     │──── 3. REQUEST ───────────────────────→│  "Oke saya mau IP itu"
     │                                        │
     │←─── 4. ACKNOWLEDGE ───────────────────│  "Sip, IP itu milik kamu"
     │                                        │
```

### Kenapa Butuh DHCP Relay?

DHCP Discover itu **broadcast** — dan broadcast **tidak melewati router**. Jadi kalau DHCP server ada di router, tapi PC ada di VLAN lain, broadcast-nya gak sampai.

```
Tanpa Relay:
  PC-Mkt (VLAN 20) ──broadcast──→ ❌ STUCK di VLAN 20, gak sampai router

Dengan Relay (ip helper-address):
  PC-Mkt (VLAN 20) ──broadcast──→ Switch ──→ Router sub-interface
                                              │
                                              ↓ relay ke DHCP service
                                              ✅ IP diberikan
```

---

## 🏢 Skenario

Melanjutkan dari **Lab 004**. PT. Nusantara Digital sudah punya network dengan VLAN dan VLSM. Sekarang IT Manager minta semua PC dapat IP otomatis — tidak ada lagi assign IP manual.

**Requirement:**
- DHCP Server berjalan di **R-CORE** (router sebagai DHCP server)
- Setiap VLAN punya **DHCP pool** sendiri
- Gateway (`.1`) dan reserved IP (`.2 - .10`) **tidak boleh** diberikan ke client
- Server Farm (VLAN 50) tetap **static IP** — server tidak pakai DHCP

---

## 📋 DHCP Pool Planning

| VLAN | Subnet | Excluded (Reserved) | DHCP Pool Range | Gateway |
|------|--------|-------------------|-----------------|---------|
| 10 - Engineering | 192.168.10.0/26 | .1 - .10 | .11 - .62 | 192.168.10.1 |
| 20 - Marketing | 192.168.10.64/27 | .65 - .74 | .75 - .94 | 192.168.10.65 |
| 30 - Finance | 192.168.10.96/28 | .97 - .100 | .101 - .110 | 192.168.10.97 |
| 40 - Management | 192.168.10.128/29 | .129 - .131 | .132 - .134 | 192.168.10.129 |
| 50 - Server | 192.168.10.112/28 | — | **Tidak pakai DHCP** | 192.168.10.113 |

> 💡 **Kenapa Server gak pakai DHCP?** Karena server butuh IP yang **tetap/static**. Kalau IP server berubah, semua service yang pointing ke IP tersebut ikut mati.

### Excluded Address — Kenapa?

| Range | Dipakai Untuk |
|-------|---------------|
| `.1` | Gateway (router sub-interface) |
| `.2 - .10` | Network devices: access point, switch management, printer |

Kalau tidak di-exclude, DHCP bisa kasih `.1` ke PC random — dan gateway conflict, **seluruh VLAN mati**.

---

## 🖥️ Topologi

Sama dengan Lab 004 — tidak ada perubahan fisik. Hanya tambah konfigurasi DHCP di R-CORE.

```
                        [R-CORE]
                      DHCP Server
                  Gi0/0  Gi0/1   Gi0/2
                    │      │       │
                trunk  trunk   trunk
                    │      │       │
               [SW-ENG] [SW-OFFICE] [SW-SERVER]
                  │       │  │  │       │
               PC-Eng  PC-Mkt │ PC-Mgmt SRV-01
              (DHCP)  (DHCP)  │ (DHCP)  (STATIC)
                       PC-Fin
                      (DHCP)
```

---

## ⚙️ Konfigurasi

### Prasyarat

Lab 004 harus sudah selesai dan berjalan — semua VLAN, trunk, dan sub-interface sudah **up/up**.

### Urutan Konfigurasi

```
Step 1: Excluded address (reserve IP dulu, baru buat pool)
Step 2: Buat DHCP pool per VLAN
Step 3: Set PC ke mode DHCP
Step 4: Verifikasi
```

> ⚠️ **Excluded address HARUS di-set sebelum pool aktif.** Kalau pool aktif duluan tanpa exclusion, IP gateway bisa langsung diambil client.

---

### R-CORE — DHCP Configuration

```cisco
enable
configure terminal

! ============================================
! STEP 1: Excluded Address (Reserve IP)
! ============================================

! Engineering — reserve .1 sampai .10
ip dhcp excluded-address 192.168.10.1 192.168.10.10

! Marketing — reserve .65 sampai .74
ip dhcp excluded-address 192.168.10.65 192.168.10.74

! Finance — reserve .97 sampai .100
ip dhcp excluded-address 192.168.10.97 192.168.10.100

! Management — reserve .129 sampai .131
ip dhcp excluded-address 192.168.10.129 192.168.10.131

! ============================================
! STEP 2: DHCP Pool per VLAN
! ============================================

! Pool VLAN 10 — Engineering
ip dhcp pool VLAN10-ENGINEERING
 network 192.168.10.0 255.255.255.192
 default-router 192.168.10.1
 dns-server 8.8.8.8
 lease 7
exit

! Pool VLAN 20 — Marketing
ip dhcp pool VLAN20-MARKETING
 network 192.168.10.64 255.255.255.224
 default-router 192.168.10.65
 dns-server 8.8.8.8
 lease 7
exit

! Pool VLAN 30 — Finance
ip dhcp pool VLAN30-FINANCE
 network 192.168.10.96 255.255.255.240
 default-router 192.168.10.97
 dns-server 8.8.8.8
 lease 7
exit

! Pool VLAN 40 — Management
ip dhcp pool VLAN40-MANAGEMENT
 network 192.168.10.128 255.255.255.248
 default-router 192.168.10.129
 dns-server 8.8.8.8
 lease 1
exit

end
```

### Penjelasan Parameter

| Parameter | Fungsi | Contoh |
|-----------|--------|--------|
| `network` | Subnet yang di-serve DHCP | `network 192.168.10.0 255.255.255.192` |
| `default-router` | Gateway yang diberikan ke client | `default-router 192.168.10.1` |
| `dns-server` | DNS server yang diberikan ke client | `dns-server 8.8.8.8` |
| `lease` | Durasi pinjam IP (hari) | `lease 7` = 7 hari |

> 💡 **Kenapa Management lease 1 hari?** Karena Management cuma 5 host di subnet /29 (6 usable). Kalau lease lama dan device disconnect, IP-nya "terkunci" lama — subnet kecil bisa kehabisan IP.

### End Devices — Set ke DHCP

Klik PC → Desktop → IP Configuration → pilih **DHCP** (bukan Static):

| Device | Mode | Seharusnya Dapat IP |
|--------|------|-------------------|
| PC-Eng | DHCP | 192.168.10.11 - .62 |
| PC-Mkt | DHCP | 192.168.10.75 - .94 |
| PC-Fin | DHCP | 192.168.10.101 - .110 |
| PC-Mgmt | DHCP | 192.168.10.132 - .134 |
| SRV-01 | **Static** (tidak diubah) | 192.168.10.114 |

Setelah set DHCP, PC akan otomatis menampilkan IP address, subnet mask, default gateway, dan DNS server.

---

## ✅ Verifikasi & Testing

### 1. Cek DHCP Binding di Router

```cisco
show ip dhcp binding
```

Setiap PC yang request DHCP harus muncul di sini dengan IP, MAC address, dan lease expiration.

### 2. Cek DHCP Pool Statistics

```cisco
show ip dhcp pool
```

Menampilkan berapa IP yang sudah terpakai dan tersisa per pool.

### 3. Cek DHCP Conflict (Jika Ada)

```cisco
show ip dhcp conflict
```

Kalau kosong = bagus. Kalau ada entry, berarti ada IP yang conflict (2 device pakai IP yang sama).

### 4. Cek di PC

Dari PC yang pakai DHCP:

```
ipconfig
```

Pastikan:
- IP Address → dari range DHCP pool (bukan 0.0.0.0)
- Subnet Mask → sesuai VLSM
- Default Gateway → sesuai tabel
- DNS Server → 8.8.8.8

### 5. Ping Test

```
# Dari PC-Eng (DHCP), ping semua:
ping 192.168.10.1      ← gateway sendiri
ping [PC-Mkt IP]       ← PC Marketing (DHCP)
ping [PC-Fin IP]       ← PC Finance (DHCP)
ping 192.168.10.114    ← Server (Static)
```

### 6. Test DHCP Renew

Dari PC, release dan renew IP:

```
ipconfig /release
ipconfig /renew
```

PC harus dapat IP baru (atau sama) dari pool.

---

## 🔧 Troubleshooting Guide

### PC Gak Dapat IP dari DHCP (Masih 0.0.0.0)

```
PC dapat 0.0.0.0 atau APIPA (169.254.x.x)?
│
├── Cek DHCP service aktif di router
│   └── show ip dhcp pool
│       └── Kalau kosong → pool belum dibuat
│
├── Cek network di pool sesuai subnet
│   └── network harus match dengan sub-interface IP/mask
│       Contoh: sub-interface 192.168.10.1/26
│               pool network 192.168.10.0 255.255.255.192 ✅
│               pool network 192.168.10.0 255.255.255.0   ❌ (salah mask)
│
├── Cek trunk aktif (show interfaces trunk di switch)
│   └── Broadcast DHCP gak bisa lewat kalau trunk mati
│
├── Cek VLAN assignment (show vlan brief di switch)
│   └── PC harus di port yang assign ke VLAN yang benar
│
└── Cek excluded address tidak meng-exclude semua IP
    └── show running-config | include excluded
        └── Pastikan masih ada sisa IP di pool
```

### PC Dapat IP Tapi Gak Bisa Ping Antar VLAN

| Cek | Command | Solusi |
|-----|---------|--------|
| Default gateway benar? | `ipconfig` di PC | Harus sesuai sub-interface router |
| Sub-interface up? | `show ip interface brief` | Harus up/up |
| Routing table? | `show ip route` | Semua subnet harus Connected |

### DHCP Conflict

| Masalah | Penyebab | Solusi |
|---------|----------|--------|
| 2 device dapet IP sama | Static IP bertabrakan dengan DHCP pool | Tambahkan ke excluded-address |
| Pool kosong | Semua IP sudah di-lease | Kurangi lease time atau perbesar subnet |
| Server dapat IP DHCP | Server set ke DHCP | Set server kembali ke Static |

### Kesalahan Umum

| Masalah | Penyebab | Solusi |
|---------|----------|--------|
| Gateway tidak di-exclude | `ip dhcp excluded-address` lupa | Tambahkan excluded address |
| Pool network salah mask | Mask pool ≠ mask sub-interface | Samakan mask pool dengan sub-interface |
| DNS tidak diberikan | Lupa `dns-server` di pool | Tambahkan `dns-server 8.8.8.8` |
| Pool name typo | Nama pool salah | `no ip dhcp pool [nama-salah]` lalu buat ulang |
| Excluded range terlalu besar | Semua IP di-exclude | Cek `show run \| include excluded` |

### Command Cheat Sheet

| Command | Dimana | Fungsi |
|---------|--------|--------|
| `show ip dhcp binding` | Router | Lihat semua IP yang sudah di-lease |
| `show ip dhcp pool` | Router | Statistik per pool (total, used, available) |
| `show ip dhcp conflict` | Router | Cek IP yang conflict |
| `show running-config \| section dhcp` | Router | Lihat semua config DHCP |
| `clear ip dhcp binding *` | Router | Hapus semua lease (force renew) |
| `clear ip dhcp conflict *` | Router | Hapus record conflict |
| `no ip dhcp pool [name]` | Router (config) | Hapus DHCP pool |
| `ipconfig` | PC | Cek IP yang didapat |
| `ipconfig /release` | PC | Lepas IP DHCP |
| `ipconfig /renew` | PC | Minta IP baru dari DHCP |

---

## 🏭 Production Best Practices

1. **Selalu exclude gateway dan reserved range** — IP `.1` dan range untuk network devices harus di-exclude sebelum pool aktif. Satu conflict di gateway = seluruh VLAN mati.

2. **Lease time sesuaikan dengan kebutuhan:**
   - Office PC (jarang pindah): 7 hari
   - Meeting room / guest: 4-8 jam
   - Subnet kecil: 1 hari (biar IP cepat recycle)

3. **Server selalu static** — jangan pernah pakai DHCP untuk server, database, atau infrastructure device.

4. **DNS server:**
   - Production: pakai internal DNS (contoh: `192.168.10.114` kalau ada DNS server sendiri)
   - Lab/simple: `8.8.8.8` (Google) atau `1.1.1.1` (Cloudflare)

5. **Monitor DHCP pool** — di production, set alert kalau pool usage di atas 80%. Kalau sampai 100%, device baru gak bisa konek.

6. **DHCP Snooping** (akan dipelajari di lab 008) — mencegah rogue DHCP server di network. Tanpa ini, siapapun bisa colok DHCP server palsu dan redirect traffic.

7. **Dokumentasikan excluded range** — catat kenapa IP tertentu di-exclude dan device apa yang pakai. Tanpa ini, troubleshoot conflict jadi susah.

---

## 📝 Catatan untuk README di GitHub

- Screenshot `show ip dhcp binding` dari router
  <img width="1262" height="1113" alt="image" src="https://github.com/user-attachments/assets/34bff7f9-7db2-4bd3-b8ab-d951debb42ad" />

- Screenshot PC yang sudah dapat IP via DHCP (ipconfig)
  <img width="806" height="256" alt="image" src="https://github.com/user-attachments/assets/cc432e14-46ce-4aab-9ea8-8dd2825ab662" />

- Screenshot ping test antar VLAN
<img width="778" height="630" alt="image" src="https://github.com/user-attachments/assets/90367c7d-e922-4bbb-829a-842256d06931" />


- File `.pkt` (Packet Tracer save file)

---

## ⏭️ Lab Selanjutnya

→ **006_Static_Routing_Multi_Path** — Menghubungkan 2 network berbeda menggunakan static route, termasuk failover path untuk redundancy.
